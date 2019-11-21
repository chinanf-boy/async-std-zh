## Sending Messages

现在该实现另一半了-发送消息。实现发送的最明显方法是给每个`connection_loop`访问写入的一半`TcpStream`彼此的客户。这样，客户可以直接`.write_all`给收件人的消息。但是，这将是错误的：如果Alice发送`bob: foo`，然后查理发送`bob: bar`，鲍勃实际上可能会收到`fobaor`。通过套接字发送消息可能需要几个系统调用，因此两个并发`.write_all`可能会互相干扰！

根据经验，每个任务只应写入一个任务`TcpStream`。因此，让我们创建一个`connection_writer_loop`通过通道接收消息并将其写入套接字的任务。该任务将是消息序列化的重点。如果Alice和Charley同时向Bob发送了两条消息，则Bob会以到达消息的顺序来查看消息。

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

1.  我们将使用来自`futures`箱。
2.  为简单起见，我们将使用`unbounded`通道，并且在本教程中不会讨论背压。
3.  如`connection_loop`和`connection_writer_loop`分享相同`TcpStream`，我们需要将其放入`Arc`。注意，因为`client`只从流中读取`connection_writer_loop`只写信息流，我们在这里没有比赛。
