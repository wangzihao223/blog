+++
date = '2026-06-14T00:00:00+08:00'
draft = false
title = 'Celery 是什么：从消息队列到 Worker Runtime'
tags = ["Celery", "消息队列", "分布式任务", "RabbitMQ"]
+++

## Celery 解决什么问题

在 Web 服务里，经常会遇到一些不适合直接在请求线程里执行的事情：

- 发送邮件
- 生成报表
- 转码图片或视频
- 同步第三方接口
- 定时任务
- 批量数据处理

这些任务有几个共同点：耗时、不稳定、可以异步执行、失败后可能需要重试。

Celery 就是为这类场景设计的分布式任务队列框架。它让业务进程只负责把任务发出去，真正执行任务的部分交给独立的 worker 进程。

一个最简单的模型是：

```text
Web/API 进程
    |
    | publish task
    v
Broker(RabbitMQ/Redis)
    |
    | consume task
    v
Worker 进程
```

所以 Celery 不是 RabbitMQ，也不是 Redis。Celery 是任务模型、消息协议、worker runtime 和一些周边能力的组合。RabbitMQ、Redis 只是 Celery 可以使用的 broker。

## Celery 是服务还是协议

严格说，Celery 本身更像一个任务框架，不只是一个协议，也不是单独一个服务。

它包含几层东西：

- 协议：任务消息应该长什么样。
- client/producer：怎么把任务编码并发布到 broker。
- worker：怎么从 broker 消费消息、解码、执行任务、确认消息。
- result backend：任务结果放在哪里，比如 Redis、数据库等。
- 调度和控制能力：定时任务、重试、撤销任务、监控等。

如果只看跨语言互通，最核心的是 Celery 的消息协议。只要另一个语言能生成 Celery worker 看得懂的消息，就可以把任务发给 Python Celery worker；反过来，只要另一个语言能消费并解码 Celery 消息，也可以实现自己的 worker runtime。

## Producer、Broker、Worker

Celery 的核心角色一般有三个。

Producer 负责创建任务消息。比如一个 HTTP 接口收到请求以后，不直接执行耗时逻辑，而是把任务参数编码成消息发到 broker。

Broker 负责暂存和分发消息。RabbitMQ、Redis 都可以做 broker。broker 不理解业务含义，它只负责把消息可靠地送到消费者。

Worker 负责消费消息并执行任务。worker 会维护一组任务名到函数的映射，收到消息以后根据 task name 找到对应 handler，然后执行。

流程可以画成这样：

```text
1. producer 创建任务
2. producer 把任务发布到 broker
3. broker 把任务投递给某个 worker
4. worker 解码消息
5. worker 根据 task name 找 handler
6. handler 执行业务逻辑
7. worker ack 或 reject 消息
8. worker 可选地写入 result backend
```

## RabbitMQ 里的几个概念

如果 Celery 使用 RabbitMQ 作为 broker，就会遇到 connection、channel、exchange、queue、routing key、consumer、delivery tag 这些概念。

Connection 是 TCP 连接。一个应用一般不会为每个任务都新建连接，而是复用连接。

Channel 是 connection 里的虚拟通道。RabbitMQ 允许一个 TCP 连接上开多个 channel。实际 publish、consume、ack 通常都发生在 channel 上。这样可以减少 TCP 连接数量，同时让多个消费者并行工作。

Exchange 负责接收发布的消息，并根据规则把消息路由到 queue。常见 exchange 类型有 direct、fanout、topic。

Queue 是真正存放消息的地方。worker 通常是从 queue 里消费消息。

Routing key 是发布消息时带上的路由键。direct exchange 通常会用 routing key 匹配绑定关系，把消息送到对应 queue。

Consumer 是订阅 queue 的消费者。一个 queue 可以有多个 consumer。默认语义是竞争消费：一条消息只会投递给其中一个 consumer，而不是广播给所有 consumer。

如果想广播，通常不是让一个 queue 广播，而是使用 fanout exchange，把同一条消息复制到多个 queue，每个 queue 再由自己的 consumer 消费。

## delivery_tag 和 ack

RabbitMQ 投递消息时，会给这次投递生成一个 delivery tag。worker 处理完消息以后，需要用这个 delivery tag 进行 ack 或 reject。

需要注意的是，delivery tag 是 channel 级别的。也就是说，在哪个 channel 收到的消息，就应该在哪个 channel 上 ack。不能随便拿另一个 channel 去确认。

ack 的意思是：这条消息已经处理完成，broker 可以删除它。

reject 或 nack 的意思是：这条消息没有成功处理。此时可以选择丢弃，也可以重新入队。

所以比较安全的 worker 通常会使用手动 ack：

```text
收到消息
  |
  v
执行业务 handler
  |
  +-- 成功 -> ack
  |
  +-- 失败 -> reject/nack
```

如果使用自动 ack，broker 在投递消息后就认为消息完成了。worker 如果中途崩溃，这条消息可能就丢了。

## Celery 消息长什么样

Celery 的消息可以简单理解成三部分：

```text
AMQP properties
AMQP headers
payload/body
```

properties 是 AMQP 层的属性，比如：

- content_type：消息体的格式，常见是 application/json。
- content_encoding：编码，常见是 utf-8。
- correlation_id：通常和 task id 对应，用来关联请求和结果。
- message_id：消息 id。
- type：任务类型，通常是 task name。

headers 是 Celery 协议层的任务元数据，比如：

- id：任务 id。
- task：任务名。
- eta：预计执行时间，通常是 ISO 8601 字符串。
- expires：过期时间。
- retries：当前重试次数。
- timelimit：任务时间限制。
- root_id：任务链路里的根任务 id。
- parent_id：父任务 id。
- argsrepr：参数的展示字符串。
- kwargsrepr：关键字参数的展示字符串。

payload/body 是真正传给任务的参数。Celery protocol v2 常见 JSON 结构是：

```json
[
  [1, 2],
  {"debug": true},
  {
    "callbacks": null,
    "errbacks": null,
    "chain": null,
    "chord": null
  }
]
```

它对应的是：

```text
[args, kwargs, embed]
```

args 是位置参数，kwargs 是关键字参数，embed 是 Celery canvas 相关的元数据，比如 callback、chain、chord。

## Worker Pool 怎么并发

一个 worker runtime 可以做得很复杂，也可以先做一个简单版本。

最小可用的 worker pool 可以这样设计：

```text
Master
  |
  +-- RabbitMQ connection
  |
  +-- WorkerSlot 1
  |     +-- channel 1
  |     +-- subscription 1
  |
  +-- WorkerSlot 2
  |     +-- channel 2
  |     +-- subscription 2
  |
  +-- WorkerSlot 3
        +-- channel 3
        +-- subscription 3
```

Master 负责创建 RabbitMQ connection、启动多个 worker slot、处理启动失败和关闭流程。

每个 worker slot 有自己的 channel 和 subscription。多个 subscription 订阅同一个 queue 时，RabbitMQ 会把消息分发给其中一个消费者。这样不需要 master 自己调度每一条任务，broker 已经完成了基本分发。

任务执行逻辑可以放在 registry 里：

```text
task name -> handler
```

worker 收到消息以后，先 decode，再用 task name 查 registry，最后调用 handler。

## 自己实现一个 Celery 需要做什么

如果只做一个 MVP，范围可以收得很小：

1. 实现 Celery protocol encode/decode。
2. 实现 AMQP publish，把任务发到 RabbitMQ。
3. 实现 worker subscribe，从 RabbitMQ 订阅任务。
4. 实现 task registry，把任务名映射到 handler。
5. 实现手动 ack/reject。
6. 可选实现 result backend，把结果写到 Redis 或数据库。

这里最重要的是边界要清楚。

协议层只负责 Celery 消息怎么编码和解码，不应该关心 RabbitMQ 或 Redis。

broker 层只负责 publish、subscribe、ack、reject，不应该关心业务 handler。

runtime 层负责 worker pool、registry、任务执行和错误处理。

可以拆成这样：

```text
protocol
  encode/decode Celery message

broker
  publish/subscribe/ack/reject

worker runtime
  pool/registry/handler dispatch

result backend
  store task result
```

我最近写的 gcelery MVP 基本就是按这个思路拆的：先做纯 Gleam 的 Celery 协议编码/解码，再接 RabbitMQ 发布和订阅，最后补一个简单的 worker pool。这样可以先验证和 Python Celery 的兼容性，再逐步扩展 result backend、重试、定时任务等能力。

## 总结

Celery 的核心不是“某个队列”，而是一套异步任务系统。

它把任务执行拆成了几个独立部分：producer 只负责发任务，broker 负责暂存和分发，worker 负责执行，result backend 负责保存结果。

理解 Celery 时，最关键的是抓住两条线：

- 消息协议：一条 Celery task 在 AMQP 里到底长什么样。
- worker runtime：消息被消费后，如何找到 handler、执行任务、ack 或 reject。

只要这两条线清楚，就可以理解 Python Celery 的行为，也可以用其他 BEAM 语言实现一个能和 Celery 互通的最小 runtime。
