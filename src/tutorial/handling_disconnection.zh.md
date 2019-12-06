## Handling Disconnections

目前，我们只对 map 有，*添加*新 peers 的操作。这显然是错误的：如果一个 peer 关闭了与 chat 的连接，我们就不应尝试向其发送更多消息。

处理断开连接的一个细微之处是，我们可以检测到，它是在 reader 的 task 中，还是在 writer 的 task 中。这里最显而易见的解决方案是：在两种情况下，试着仅将 peer 从`peers` map 中移除，但这是错误的。如果读取和写入（read and write）*两个*都失败，我们将会移除 peer 两次，但有种情况是，在两种错误之间， peer 重新连接上了！解决上面问题的思考是，我们仅在 write 端完结后，才移除 peer 。如果 read 端完结，我们将通知 write 端也应停止。也就是说，我们需要添加一项手段，发出信号，以关闭 writer task。

解决这个问题的一种方法是`shutdown: Receiver<()>` channel。但是，有一个更小的解决方案，它可以巧妙地使用 RAII。 shutdown channel 是一个同步事件，因此我们不需要发送一个 shutdown 消息，只需丢弃 sender 即可。这样，即使我们的`?`或 panics 提前返回，我们也能坐享其成地确定，发出 shutdown 仅一次。

首先，让我们向`connection_loop`，添加一个 shutdown channel：

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

1.  为了强制信息不通过 shutdown channel 发送，我们使用了仅作标记的类型。
2.  我们将 shutdown channel 传递给 writer task
3.  在 reader 中，我们创建了一个`_shutdown_sender`，其唯一的目的就是 get dropped。

在`connection_writer_loop`里面，我们现在需要在 shutdown 和 message channel 之间进行选择。为此，我们使用`select`宏：

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

1.  我们添加 shutdown channel 作为参数。
2.  因为`select`，我们不能使用`while let`循环，所以我们将其 desugar(解语法糖) 到 `loop`。
3.  在 shutdown 案例下，我们使用`match void {}`作为静态检查版的`unreachable!()`。

另一个问题是，新的信息可能会被推进 peer 的 channel，这问题发生的时机是，在我们在`connection_writer_loop`检测到 disconnection(断连)，和我们实际上从`peers` map 中移除这个 peer 之间。为了不完全失去这些消息，我们将 message channel 返回给 broker。这也使我们能够建立一个有用的不变式，message channel 在`peers` map 中，是严格要求'长命'过这个 peer 的，并使 broker 本身处于无法失败的状态。

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

1.  在 broker 中，我们创建一个 channel，来获取断开连接的 peer，及其未传递的消息。
2.  当输入事件 channel 耗尽时（即，所有 readers 退出时）， broker 的 main 循环退出。
3.  因为 broker 本身持有一个`disconnect_sender`，我们就知道在 main 循环中的 disconnections channel，无法完全排掉。
4.  我们将 peer 的 name 和待处理消息（pending messages），发送到 disconnections channel ，这里有两条路径，一条快乐，一条不是太快乐(in both the happy and the not-so-happy path)。
    同样，我们可以安全地 unwrap，因为 broker 的生命周期是超过 writers 的。
5.  我们 drop `peers` map ，以关闭 writers 的 message channel ，并确定关闭 writers。
    在当前设置中， broker 等待 reader 的关闭，其实并不是绝对必要的。但是，如果我们要添加一个 server-initiated shutdown（例如，kbd：[ctrl+c] 处理），这将是 broker 关闭 writers 的一种方式。
6.  最后，我们关闭，并排干 disconnections channel 。
