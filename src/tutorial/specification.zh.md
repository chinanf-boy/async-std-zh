# Specification and Getting Started

## Specification

聊天使用基于TCP的简单文本协议。该协议包含utf-8消息，由分隔`\n`。

客户端连接到服务器，并作为第一行发送登录信息。之后，客户端可以使用以下语法将消息发送给其他客户端：

```text
login1, login2, ... loginN: message
```

然后，每个指定的客户端都会收到一个`from login: message`信息。

可能的会话如下所示

```text
On Alice's computer:   |   On Bob's computer:

> alice                |   > bob
> bob: hello               < from alice: hello
                       |   > alice, bob: hi!
                           < from bob: hi!
< from bob: hi!        |
```

聊天服务器的主要挑战是跟踪许多并发连接。聊天客户端的主要挑战是管理并发传出消息，传入消息和用户键入。

## Getting Started

让我们创建一个新的Cargo项目：

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
