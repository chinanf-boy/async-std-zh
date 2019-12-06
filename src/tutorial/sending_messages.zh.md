## Sending Messages

现在该实现另一半了 —— 发送消息。实现发送的最明显方法是，给每个`connection_loop`访问，平分彼此客户端`TcpStream`的 write。这样，一个客户可以直接`.write_all`，给到收件人消息。但是，这会是错误的：如果 Alice 发送`bob: foo`，然后 Charley 发送`bob: bar`，Bob 实际上可能会收到`fobaor`。在 socket 上，发送消息可能需要几个 syscalls，因此两个并发`.write_all`可能会互相干扰！

根据经验，每个 task 应该只写入一个`TcpStream`。因此，让我们创建一个`connection_writer_loop` task,它能在一个通道(channel)上接收消息，并将其写入 socket。该 task 将是消息序列化的重点。如果 Alice 和 Charley 同时向 Bob 发送了两条消息，则 Bob 会以到达消息的顺序，来查看消息。

```rust,edition2018
# extern crate async_std;
# extern crate futures;
# use async_std::{
#     net::TcpStream,
#     prelude::*,
# };
use futures::channel::mpsc; // 1
use futures::sink::SinkExt;
use std::sync::Arc;

# type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
type Sender<T> = mpsc::UnboundedSender<T>; // 2
type Receiver<T> = mpsc::UnboundedReceiver<T>;

async fn connection_writer_loop(
    mut messages: Receiver<String>,
    stream: Arc<TcpStream>, // 3
) -> Result<()> {
    let mut stream = &*stream;
    while let Some(msg) = messages.next().await {
        stream.write_all(msg.as_bytes()).await?;
    }
    Ok(())
}
```

1.  我们将使用`futures`箱子的 channels。
2.  简单起见，我们将使用`unbounded` channels，并且在本教程中不会讨论 backpressure。
3.  如`connection_loop`一样，`connection_writer_loop`分享相同的`TcpStream`，我们需要将其放入`Arc`。注意，因为`client`只能读取 stream，和`connection_writer_loop`只写入 stream，我们在这里没有竞态问题。
