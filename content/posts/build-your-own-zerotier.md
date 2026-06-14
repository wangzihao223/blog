+++
date = '2026-06-14T00:00:00+08:00'
draft = false
title = '用 Gleam 实现一个 ZeroTier 风格的二层 overlay 网络'
tags = ["Gleam", "Erlang", "OTP", "网络", "overlay"]
+++

## 这个项目做什么

ZeroTier 的核心思路是：把多台机器的虚拟网卡连到同一个虚拟交换机上，让它们像接在同一个局域网里一样通信。

这个项目用 Gleam / Erlang OTP 实现了一个最小版本：通过 UDP 隧道把多台机器的 TAP 设备连到一个虚拟二层交换机，支持 IPv4、IPv6、ARP 等所有以太网上层协议。

它不是完整的 ZeroTier，但核心思路是一样的：

```text
客户端 A                      服务端                      客户端 B
┌─────────┐                ┌─────────┐                ┌─────────┐
│  TAP    │                │ vswitch │                │  TAP    │
│ device  │                │(二层交换)│                │ device  │
└────┬────┘                └────┬────┘                └────┬────┘
     │                          │                          │
   vport                  udp_server                     vport
     │                    │         │                      │
  udp_client ════════════UDP       UDP════════════════ udp_client
```

## 分层结构

代码分成五层，每层职责单一：

```text
protocol    以太网帧头解析、UDP 隧道协议编解码
vswitch     二层交换机，MAC 学习 + 转发
device      本地设备封装（TAP 或 mock）
transport   客户端 transport + 服务端 UDP 接入层
vport       在 device 和 transport 之间泵送帧
```

vswitch 只看 MAC 地址，不管帧里装的是什么。device 只管读写，不管帧去哪里。vport 只管泵送，不碰底层资源。每一层都可以独立测试。

## 交换机怎么工作

vswitch 实现了最小版二层交换机逻辑：MAC 学习 + 单播/泛洪决策。

每收到一帧，先学习源 MAC 来自哪个客户端：

```gleam
pub fn handle_frame(
  table: Table,
  src_mac: Mac,
  dst_mac: Mac,
  input_port: PortId,
) -> #(Table, ForwardAction) {
  let updated_table = learn(table, src_mac, input_port)

  case lookup(updated_table, dst_mac) {
    Ok(output_port) -> #(updated_table, Unicast(output_port))
    Error(Nil) -> #(updated_table, Flood)
  }
}
```

如果目的 MAC 已知就单播，不知道就泛洪给所有客户端。这和真实二层交换机的行为一致。

## UDP 协议

客户端和服务端之间用一个简单的自定义 UDP 协议通信，第一个字节是消息类型：

```text
0x01  Hello(client_id)    客户端握手，声明自己的 ID
0x02  Welcome             服务端确认
0x03  ClientFrame(frame)  客户端上行以太网帧
0x04  ServerFrame(frame)  服务端下行以太网帧
```

客户端连接时先发 Hello，收到 Welcome 后进入正常收发。之后所有帧都包裹在 ClientFrame/ServerFrame 里传输。

## 资源归属问题

在 Erlang/OTP 里，有些资源天然绑定到创建它的进程：port、socket、以及 Gleam 的 `Subject`。如果把它们传给另一个进程去使用，会出各种问题。

这个项目遵守一条原则：**谁创建，谁持有，谁操作**。

`Subject` 的问题尤其典型。Gleam 的 `Subject` 是一个 `{pid, ref}` 对，`process.receive` 会检查 owner。如果在进程 A 里调用 `new_subject()`，然后把这个 subject 传给进程 B 去 `receive`，就会 panic：

```
Cannot receive with a subject owned by another process
```

这在这个项目里踩了好几次，每次都需要把 `new_subject()` 移到真正做 `receive` 的那个进程里。

## 服务端内部结构

服务端拆成三个并行进程：

```text
sessions actor      管理 client_id <-> endpoint 映射，缓存下行帧
upstream 进程       专门 udp.receive，上行帧送入 vswitch
downstream 进程     轮询 sessions actor，把下行帧发回客户端
```

最初是一个进程串行处理上下行，上行忙的时候下行要等，下行积压多的时候上行要等。拆开之后两个方向完全独立。

下行帧的路径：

```text
vswitch
  → forwarder 进程（每客户端一个，持有 receiver）
  → GotFrame 消息 → sessions actor（缓存）
  → PollAll → downstream 进程
  → udp.send_to
```

forwarder 进程存在的原因也是 Subject owner 问题。下行 receiver 必须由 forwarder 进程自己创建，它才有权 receive：

```gleam
let _ = spawn_unlinked(fn() {
  let receiver = server.new_receiver()   // 在 forwarder 进程里创建
  server.connect(switch, client_id, receiver)
  forwarder_loop(endpoint, receiver, sessions_self)
})
```

如果在 sessions actor 的消息处理函数里调用 `new_receiver()`，receiver 的 owner 就是 sessions actor，forwarder 进程拿来 receive 就 panic。

## OTP actor 里不能做裸 receive

另一个陷阱：在 OTP actor 的消息处理回调里，不能用 `process.receive` 去接收另一个 Subject 的消息。

OTP actor 框架控制了进程的邮箱。进来的消息如果不是 actor 声明的消息类型，框架会直接丢掉并打警告：

```
Actor discarding unexpected message: ...
```

最初的设计是让 sessions actor 持有 receiver，用 `PollAll` 在 handler 里调 `process.receive(receiver, 0)` 轮询。结果就是 vswitch 发来的帧都被框架丢掉，`receive` 永远拿不到东西。

解决方法：让 forwarder 进程持有 receiver 并做 receive，把帧作为 `GotFrame` 消息发给 sessions actor。actor 只处理自己认识的消息类型，不做裸 receive。

## 两种运行模式

**mock 模式**：用 Python helper 模拟帧 I/O，不需要真实网卡，适合开发和测试。

```sh
gleam run -m server
gleam run -m client -- --id client-1
```

**TAP 模式**：创建真实 TAP 设备，分配 IP，可以直接 ping：

```sh
# 机器 A
sudo ip tuntap add dev tap0 mode tap
sudo ip addr add 10.0.0.1/24 dev tap0
sudo ip link set tap0 up
sudo gleam run -m client -- --server <服务端IP> --id client-1 --tap tap0

# 机器 B
sudo ip tuntap add dev tap0 mode tap
sudo ip addr add 10.0.0.2/24 dev tap0
sudo ip link set tap0 up
sudo gleam run -m client -- --server <服务端IP> --id client-2 --tap tap0

# 验证
ping 10.0.0.2
```

## 支持的协议

因为工作在二层，vswitch 只看 MAC，不解析上层内容，所以以太网能跑的协议都支持：IPv4、IPv6、ARP、NDP，乃至自定义 EtherType。唯一约束是帧大小不能超过 UDP 单包限制，标准 1500 字节 MTU 完全没问题。

## 还缺什么

目前实现了核心链路，但离生产可用还差一些东西：

- supervision tree：actor 崩了没有自动重启
- 重连和心跳：客户端掉线后没有检测机制
- 旧 forwarder 进程泄漏：客户端重连会产生一个永远 timeout 的 forwarder
- 跨机器端到端测试脚本
- TAP + 双机 ping 的完整演示

代码在 [build_your_own_zerotier](https://github.com/wangzihao/build_your_own_zerotier)。
