+++
title = "动手实现 SMTP"
date = "2026-06-17T16:45:00+08:00"
draft = false
tags = ["Elixir", "SMTP", "协议"]
+++
![SMTP 发送流程](/images/smtp_flow.png)
## 什么是 SMTP

SMTP，全称是 Simple Mail Transfer Protocol，中文通常叫简单邮件传输协议。它是互联网上发送邮件的核心协议。

平时我们说“发邮件”，背后通常会涉及两类协议：

- SMTP：负责发送邮件
- POP3 / IMAP：负责收取和同步邮件

也就是说，SMTP 主要解决的是“客户端或服务器如何把一封邮件交给邮件服务器，并继续投递到目标邮箱”的问题。

例如你从 QQ 邮箱发送一封邮件到 163 邮箱，大致过程是：

```text
邮件客户端 -> QQ SMTP 服务器 -> 163 邮件服务器 -> 收件人邮箱
```

我们的程序目前扮演的是第一个角色：一个 SMTP 客户端。

## SMTP 是文本协议

SMTP 是一种基于文本命令的协议。客户端和服务器之间通过 TCP 连接通信，双方发送的内容都是类似下面这样的文本行：

```text
EHLO localhost
MAIL FROM:<sender@qq.com>
RCPT TO:<receiver@example.com>
DATA
QUIT
```

服务器也会返回文本响应：

```text
220 smtp.qq.com ready
250 OK
354 End data with <CR><LF>.<CR><LF>
221 Bye
```

每一条响应前面通常都有一个三位数字响应码。客户端根据响应码判断当前步骤是否成功。

## 常见响应码

SMTP 响应码中，常见的有：

```text
220  服务已准备好，通常是连接成功后的欢迎信息
250  请求成功
334  AUTH LOGIN 认证过程中的继续输入提示
354  可以开始发送 DATA 邮件内容
235  认证成功
221  服务关闭连接
4xx  临时错误
5xx  永久错误或命令错误
```

例如服务器返回：

```text
235 Authentication successful
```

表示登录认证成功。

如果返回：

```text
502 Invalid parameters
```

通常表示客户端发送的命令或邮件内容格式不符合服务器要求。

## SMTP 的基本发送流程

一封普通邮件的 SMTP 发送流程大致如下：

```text
1. 建立 TCP 连接
2. 读取服务器 220 欢迎信息
3. 发送 EHLO
4. 如果服务器支持 STARTTLS，则升级为 TLS 加密连接
5. TLS 后再次发送 EHLO
6. 使用 AUTH 登录
7. 发送 MAIL FROM
8. 发送 RCPT TO
9. 发送 DATA
10. 发送邮件头和正文
11. 使用单独一行 . 结束邮件内容
12. 发送 QUIT
13. 关闭连接
```

下面分开说明每一步。

## 连接与欢迎信息

客户端首先连接 SMTP 服务器，例如：

```text
smtp.qq.com:587
```

连接成功后，服务器会先发送欢迎信息：

```text
220 newxmesmtplogicsvrsza53-0.qq.com XMail Esmtp QQ Mail Server.
```

这里的 `220` 表示服务已经准备好。

## EHLO

连接后，客户端发送：

```text
EHLO localhost
```

`EHLO` 的作用是向服务器打招呼，并让服务器返回它支持的能力列表。

服务器可能返回：

```text
250-smtp.qq.com
250-PIPELINING
250-SIZE 73400320
250-STARTTLS
250-AUTH LOGIN PLAIN XOAUTH XOAUTH2
250 SMTPUTF8
```

这里可以看到几个重要能力：

- `STARTTLS`：支持把当前明文 TCP 连接升级为 TLS 加密连接
- `AUTH LOGIN PLAIN`：支持登录认证
- `SIZE`：支持的邮件大小限制
- `SMTPUTF8`：支持 UTF-8 邮件地址或内容相关能力

## STARTTLS

在 587 端口上，SMTP 通常先建立明文 TCP 连接，然后通过 `STARTTLS` 升级为 TLS。

客户端发送：

```text
STARTTLS
```

服务器如果同意，会返回：

```text
220 Ready to start TLS
```

之后客户端不能继续直接用普通 TCP 文本发送命令，而是要把当前 socket 升级为 TLS socket。

在 Elixir/Erlang 中，这一步可以使用：

```elixir
:ssl.connect(socket, ssl_opts, timeout)
```

TLS 建立后，客户端通常要再次发送 `EHLO`。这是因为 TLS 前后服务器公布的能力可能不同，例如 TLS 之后才允许认证。

## AUTH LOGIN

`AUTH LOGIN` 是一种常见的 SMTP 登录方式。

流程是：

```text
AUTH LOGIN
334 ...
base64(username)
334 ...
base64(password)
235 Authentication successful
```

用户名和密码不是明文直接发送，而是 base64 后发送。

需要注意：base64 不是加密，只是一种编码方式。所以 `AUTH LOGIN` 通常必须配合 TLS 使用。

对于 QQ 邮箱，密码一般不是 QQ 登录密码，而是邮箱设置里生成的 SMTP 授权码。

## MAIL FROM

认证成功后，客户端发送：

```text
MAIL FROM:<sender@qq.com>
```

这一步声明“信封发件人”。它属于 SMTP 信封层，邮件服务器用它处理投递和退信。

它和邮件内容里的 `From:` 头不是完全同一层概念，但正常情况下应该保持一致。

## RCPT TO

然后发送：

```text
RCPT TO:<receiver@example.com>
```

这一步声明“信封收件人”。

如果要发给多个收件人，可以发送多次：

```text
RCPT TO:<a@example.com>
RCPT TO:<b@example.com>
RCPT TO:<c@example.com>
```

## DATA

信封信息确认后，客户端发送：

```text
DATA
```

服务器如果允许接收邮件内容，会返回：

```text
354 End data with <CR><LF>.<CR><LF>
```

然后客户端发送完整邮件内容，包括邮件头和正文：

```text
From: sender@qq.com
To: receiver@example.com
Subject: Hello
MIME-Version: 1.0
Content-Type: text/plain; charset=utf-8

Hello from Elixir.
```

最后必须用单独一行点号结束：

```text
.
```

协议字节上就是：

```text
\r\n.\r\n
```

这表示 DATA 内容发送完毕。

## 邮件头、正文与 MIME

SMTP 只负责传输邮件内容，但邮件内容本身需要符合邮件格式。

最简单的纯文本邮件类似：

```text
From: sender@qq.com
To: receiver@example.com
Subject: Test
MIME-Version: 1.0
Content-Type: text/plain; charset=utf-8
Content-Transfer-Encoding: 8bit

这是邮件正文。
```

如果要发送附件，就不能只发普通文本，而要使用 MIME 的 `multipart/mixed` 格式。

带附件的邮件大致如下：

```text
Content-Type: multipart/mixed; boundary="esender-123"

--esender-123
Content-Type: text/plain; charset=utf-8

正文内容

--esender-123
Content-Type: application/pdf; name="report.pdf"
Content-Disposition: attachment; filename="report.pdf"
Content-Transfer-Encoding: base64

JVBERi0xLjQK...

--esender-123--
```

其中：

- `boundary` 用来分隔正文和附件
- 附件内容通常要 base64 编码
- 最后的 `--boundary--` 表示整个 multipart 内容结束

## 动手实现SMTP

本项目是一个用 Elixir 编写的 SMTP 练习项目，核心模块是：

```text
EsenderElixir.SMTP.Client
```

目前已经实现了：

- 使用 `:gen_tcp` 连接 SMTP 服务器
- 读取服务器 `220` banner
- 解析单行和多行 SMTP 响应
- 发送 `EHLO`
- 发送 `STARTTLS`
- 使用 `:ssl.connect/3` 把 TCP socket 升级成 TLS socket
- TLS 后再次发送 `EHLO`
- 使用 `AUTH LOGIN` 登录
- 发送 `MAIL FROM`
- 发送 `RCPT TO`
- 发送 `DATA`
- 构造普通文本邮件
- 构造带附件的 MIME multipart 邮件
- 发送 `QUIT`
- 关闭 TCP 或 TLS socket

## 项目里的主流程

项目主流程在：

```text
lib/esender_elixir.ex
```

核心流程是：

```text
读取配置
构造邮件内容
连接 SMTP 服务器
EHLO
STARTTLS
TLS EHLO
AUTH LOGIN
MAIL FROM
RCPT TO
DATA
QUIT
关闭连接
```

配置文件可以写在：

```text
config/smtp.secret.exs
```

示例：

```elixir
[
  username: "your@qq.com",
  password: "smtp-auth-code",
  host: "smtp.qq.com",
  port: 587,
  domain: "localhost",
  to: "receiver@example.com",
  subject: "SMTP test",
  body: "Hello from Elixir.",
  attachments: [
    "F:/files/a.txt",
    "F:/files/report.pdf"
  ]
]
```

## Mix Task 命令入口

项目还提供了一个 Mix task：

```text
lib/mix/tasks/esender.ex
```

可以通过命令行执行：

```bash
mix esender
```

也可以用命令行参数覆盖配置文件：

```bash
mix esender --to someone@example.com --subject "hello"
```

命令行后面的普通参数会作为附件路径：

```bash
mix esender --to someone@example.com F:/files/a.txt F:/files/b.pdf
```

Mix task 内部会先启动应用：

```elixir
Mix.Task.run("app.start")
```

这样可以确保 `:ssl` 应用已经启动，避免 `:ssl_not_started`。

## 小结

SMTP 协议本身并不复杂，它是一个基于文本命令和响应码的协议, 适合作为小项目练练手。

真正容易出错的地方通常在这些细节：

- 587 端口需要 `STARTTLS`
- `AUTH LOGIN` 需要在 TLS 后进行
- 用户名和密码要 base64 编码
- QQ 邮箱需要 SMTP 授权码
- `DATA` 里必须发送完整邮件头和正文
- 邮件内容结束必须是单独一行 `.`
- 附件不是 SMTP 命令，而是 MIME 邮件内容格式
- 文本协议换行应使用 `\r\n`


