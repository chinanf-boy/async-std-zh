## Handling Disconnections

目前，我们只有*加*地图的新同伴。这显然是错误的：如果一个对等方关闭了与聊天的连接，我们不应尝试向其发送更多消息。

处理断开连接的一个细微之处是，我们可以在阅读者的任务中或在编写者的任务中检测到断开连接。这里最明显的解决方案是仅将对等方从`peers`在两种情况下都映射，但这是错误的。如果*都*读取和写入失败，我们将删除对等体两次，但在两种故障之间可能会重新连接对等体！为了解决这个问题，我们仅在写操作完成后才删除对等体。如果读取端完成，我们将通知写入端也应停止。也就是说，我们需要添加一种功能来发出信号以关闭编写器任务。

解决这个问题的一种方法是`shutdown: Receiver<()>`渠道。但是，有一个更小的解决方案，它可以巧妙地使用RAII。关闭通道是一个同步事件，因此我们不需要发送关闭消息，只需丢弃发送者即可。这样，即使我们提前通过`?`或恐慌。

首先，让我们向`connection_loop`：

```rust,edition2018
# extern crate async_std;
# extern crate futures;
# use async_std::net::TcpStream;
# use futures::channel::mpsc;
# use futures::SinkExt;
# use std::sync::Arc;
#
# type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
# type Sender<T> = mpsc::UnboundedSender<T>;
# type Receiver<T> = mpsc::UnboundedReceiver<T>;
#
#[derive(Debug)]
enum Void {} // 1

#[derive(Debug)]
enum Event {
    NewPeer {
        name: String,
        stream: Arc<TcpStream>,
        shutdown: Receiver<Void>, // 2
    },
    Message {
        from: String,
        to: Vec<String>,
        msg: String,
    },
}

async fn connection_loop(mut broker: Sender<Event>, stream: Arc<TcpStream>) -> Result<()> {
    // ...
#   let name: String = unimplemented!();
    let (_shutdown_sender, shutdown_receiver) = mpsc::unbounded::<Void>(); // 3
    broker.send(Event::NewPeer {
        name: name.clone(),
        stream: Arc::clone(&stream),
        shutdown: shutdown_receiver,
    }).await.unwrap();
    // ...
#   unimplemented!()
}
```

1.  为了强制没有消息通过关闭通道发送，我们使用了无人居住的类型。
2.  我们将关闭通道传递给writer任务
3.  在读者中，我们创建了一个`_shutdown_sender`唯一的目的就是放弃。

在里面`connection_writer_loop`，我们现在需要在关机和消息通道之间进行选择。我们使用`select`为此使用宏：

```rust,edition2018
# extern crate async_std;
# extern crate futures;
# use async_std::{net::TcpStream, prelude::*};
use futures::channel::mpsc;
use futures::{select, FutureExt};
# use std::sync::Arc;

# type Receiver<T> = mpsc::UnboundedReceiver<T>;
# type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
# type Sender<T> = mpsc::UnboundedSender<T>;

# #[derive(Debug)]
# enum Void {} // 1

async fn connection_writer_loop(
    messages: &mut Receiver<String>,
    stream: Arc<TcpStream>,
    shutdown: Receiver<Void>, // 1
) -> Result<()> {
    let mut stream = &*stream;
    let mut messages = messages.fuse();
    let mut shutdown = shutdown.fuse();
    loop { // 2
        select! {
            msg = messages.next().fuse() => match msg {
                Some(msg) => stream.write_all(msg.as_bytes()).await?,
                None => break,
            },
            void = shutdown.next().fuse() => match void {
                Some(void) => match void {}, // 3
                None => break,
            }
        }
    }
    Ok(())
}
```

1.  我们添加关闭通道作为参数。
2.  因为`select`，我们不能使用`while let`循环，所以我们将其进一步解糖`loop`。
3.  在关机情况下，我们使用`match void {}`作为静态检查`unreachable!()`。

另一个问题是，在我们检测到`connection_writer_loop`以及我们实际上从`peers`映射，新消息可能会被推送到对等方的频道中。为了不完全丢失这些消息，我们将消息通道返回给代理。这也使我们能够建立一个有用的不变式，即消息通道在消息通道中严格超过对等端。`peers`地图，并使经纪人本身无法失败。

## Final Code

最终代码如下所示：

```rust,edition2018
# extern crate async_std;
# extern crate futures;
use async_std::{
    io::BufReader,
    net::{TcpListener, TcpStream, ToSocketAddrs},
    prelude::*,
    task,
};
use futures::channel::mpsc;
use futures::{select, FutureExt, SinkExt};
use std::{
    collections::hash_map::{Entry, HashMap},
    future::Future,
    sync::Arc,
};

type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
type Sender<T> = mpsc::UnboundedSender<T>;
type Receiver<T> = mpsc::UnboundedReceiver<T>;

#[derive(Debug)]
enum Void {}

// main
fn run() -> Result<()> {
    task::block_on(accept_loop("127.0.0.1:8080"))
}

async fn accept_loop(addr: impl ToSocketAddrs) -> Result<()> {
    let listener = TcpListener::bind(addr).await?;
    let (broker_sender, broker_receiver) = mpsc::unbounded();
    let broker_handle = task::spawn(broker_loop(broker_receiver));
    let mut incoming = listener.incoming();
    while let Some(stream) = incoming.next().await {
        let stream = stream?;
        println!("Accepting from: {}", stream.peer_addr()?);
        spawn_and_log_error(connection_loop(broker_sender.clone(), stream));
    }
    drop(broker_sender);
    broker_handle.await;
    Ok(())
}

async fn connection_loop(mut broker: Sender<Event>, stream: TcpStream) -> Result<()> {
    let stream = Arc::new(stream);
    let reader = BufReader::new(&*stream);
    let mut lines = reader.lines();

    let name = match lines.next().await {
        None => Err("peer disconnected immediately")?,
        Some(line) => line?,
    };
    let (_shutdown_sender, shutdown_receiver) = mpsc::unbounded::<Void>();
    broker.send(Event::NewPeer {
        name: name.clone(),
        stream: Arc::clone(&stream),
        shutdown: shutdown_receiver,
    }).await.unwrap();

    while let Some(line) = lines.next().await {
        let line = line?;
        let (dest, msg) = match line.find(':') {
            None => continue,
            Some(idx) => (&line[..idx], line[idx + 1 ..].trim()),
        };
        let dest: Vec<String> = dest.split(',').map(|name| name.trim().to_string()).collect();
        let msg: String = msg.trim().to_string();

        broker.send(Event::Message {
            from: name.clone(),
            to: dest,
            msg,
        }).await.unwrap();
    }

    Ok(())
}

async fn connection_writer_loop(
    messages: &mut Receiver<String>,
    stream: Arc<TcpStream>,
    shutdown: Receiver<Void>,
) -> Result<()> {
    let mut stream = &*stream;
    let mut messages = messages.fuse();
    let mut shutdown = shutdown.fuse();
    loop {
        select! {
            msg = messages.next().fuse() => match msg {
                Some(msg) => stream.write_all(msg.as_bytes()).await?,
                None => break,
            },
            void = shutdown.next().fuse() => match void {
                Some(void) => match void {},
                None => break,
            }
        }
    }
    Ok(())
}

#[derive(Debug)]
enum Event {
    NewPeer {
        name: String,
        stream: Arc<TcpStream>,
        shutdown: Receiver<Void>,
    },
    Message {
        from: String,
        to: Vec<String>,
        msg: String,
    },
}

async fn broker_loop(events: Receiver<Event>) {
    let (disconnect_sender, mut disconnect_receiver) = // 1
        mpsc::unbounded::<(String, Receiver<String>)>();
    let mut peers: HashMap<String, Sender<String>> = HashMap::new();
    let mut events = events.fuse();
    loop {
        let event = select! {
            event = events.next().fuse() => match event {
                None => break, // 2
                Some(event) => event,
            },
            disconnect = disconnect_receiver.next().fuse() => {
                let (name, _pending_messages) = disconnect.unwrap(); // 3
                assert!(peers.remove(&name).is_some());
                continue;
            },
        };
        match event {
            Event::Message { from, to, msg } => {
                for addr in to {
                    if let Some(peer) = peers.get_mut(&addr) {
                        let msg = format!("from {}: {}\n", from, msg);
                        peer.send(msg).await
                            .unwrap() // 6
                    }
                }
            }
            Event::NewPeer { name, stream, shutdown } => {
                match peers.entry(name.clone()) {
                    Entry::Occupied(..) => (),
                    Entry::Vacant(entry) => {
                        let (client_sender, mut client_receiver) = mpsc::unbounded();
                        entry.insert(client_sender);
                        let mut disconnect_sender = disconnect_sender.clone();
                        spawn_and_log_error(async move {
                            let res = connection_writer_loop(&mut client_receiver, stream, shutdown).await;
                            disconnect_sender.send((name, client_receiver)).await // 4
                                .unwrap();
                            res
                        });
                    }
                }
            }
        }
    }
    drop(peers); // 5
    drop(disconnect_sender); // 6
    while let Some((_name, _pending_messages)) = disconnect_receiver.next().await {
    }
}

fn spawn_and_log_error<F>(fut: F) -> task::JoinHandle<()>
where
    F: Future<Output = Result<()>> + Send + 'static,
{
    task::spawn(async move {
        if let Err(e) = fut.await {
            eprintln!("{}", e)
        }
    })
}
```

1.  在代理中，我们创建一个渠道来获取断开连接的对等方及其未传递的消息。
2.  当输入事件通道耗尽时（即，所有阅读器退出时），代理的主循环退出。
3.  因为经纪人本身持有`disconnect_sender`，我们知道断开通道无法在主回路中完全排空。
4.  我们将对等方的姓名和待处理消息发送到断开和断开通道中，它们的路径都比较愉快。同样，我们可以安全地解开包装，因为经纪人的寿命超过了作家。
5.  我们掉落`peers`映射到关闭作者的消息通道，并确定关闭作者。在当前设置中，代理不一定要等待读者的关闭，在当前设置中并不是绝对必要的。但是，如果我们添加服务器启动的关机（例如，kbd：[ctrl+c]处理），这将是代理关闭编写器的一种方式。
6.  最后，我们关闭并排干断开通道。
