# Specification and Getting Started

## Specification

聊天使用基于 TCP 的简单文本协议。该协议包含 utf-8 消息，由`\n`分隔。

客户端连接到服务器，发送第一行登录(login)名。之后，客户端可以使用以下语法，将消息发送给其他客户端：

```text
login1, login2, ... loginN: message
```

然后，每个指定的客户端都会收到一个`from login: message`信息。

可能的会话，如下所示

```text
在 Alice's 电脑:        |   在 Bob's 电脑:

> alice                |   > bob
> bob: hello               < from alice: hello
                       |   > alice, bob: hi!
                           < from bob: hi!
< from bob: hi!        |
```

聊天服务器的主要挑战是跟踪大量并发连接。聊天客户端的主要挑战是，管理并发地传出消息，传入消息和用户键入。

## Getting Started

让我们创建一个新的 Cargo 项目：

```bash
$ cargo new a-chat
$ cd a-chat
```

将以下行添加到`Cargo.toml`：

```toml
[dependencies]
futures = "0.3.0"
async-std = "1"
```
