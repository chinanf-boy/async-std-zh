## Clean Shutdown

当前实现的问题之一是它无法处理正常关机。如果由于某种原因我们中断了accept循环，则所有正在进行的任务都将落在地板上。更正确的关闭顺序为：

1.  停止接受新客户
2.  传递所有未决消息
3.  退出程序

基于通道的体系结构中的干净关闭很容易，尽管它一开始可能看起来很神奇。在Rust中，一旦所有发送者都被丢弃，通道的接收者一侧就会关闭。也就是说，一旦生产者退出并丢弃他们的发送者，系统的其余部分就会自然关闭。在`async_std`这转化为两个规则：

1.  确保通道形成一个非循环图。
2.  注意以正确的顺序等待，直到系统的中间层处理挂起的消息。

在`a-chat`，我们已经有一个单向消息流：`reader -> broker -> writer`。但是，我们从不等待经纪人和作家，这可能会导致某些消息丢失。让我们添加等待到服务器：

```rust,edition2018
# extern crate async_std;
# extern crate futures;
# use async_std::{
#     io::{self, BufReader},
#     net::{TcpListener, TcpStream, ToSocketAddrs},
#     prelude::*,
#     task,
# };
# use futures::channel::mpsc;
# use futures::SinkExt;
# use std::{
#     collections::hash_map::{HashMap, Entry},
#     sync::Arc,
# };
#
# type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
# type Sender<T> = mpsc::UnboundedSender<T>;
# type Receiver<T> = mpsc::UnboundedReceiver<T>;
#
# fn spawn_and_log_error<F>(fut: F) -> task::JoinHandle<()>
# where
#     F: Future<Output = Result<()>> + Send + 'static,
# {
#     task::spawn(async move {
#         if let Err(e) = fut.await {
#             eprintln!("{}", e)
#         }
#     })
# }
#
#
# async fn connection_loop(mut broker: Sender<Event>, stream: TcpStream) -> Result<()> {
#     let stream = Arc::new(stream); // 2
#     let reader = BufReader::new(&*stream);
#     let mut lines = reader.lines();
#
#     let name = match lines.next().await {
#         None => Err("peer disconnected immediately")?,
#         Some(line) => line?,
#     };
#     broker.send(Event::NewPeer { name: name.clone(), stream: Arc::clone(&stream) }).await // 3
#         .unwrap();
#
#     while let Some(line) = lines.next().await {
#         let line = line?;
#         let (dest, msg) = match line.find(':') {
#             None => continue,
#             Some(idx) => (&line[..idx], line[idx + 1 ..].trim()),
#         };
#         let dest: Vec<String> = dest.split(',').map(|name| name.trim().to_string()).collect();
#         let msg: String = msg.trim().to_string();
#
#         broker.send(Event::Message { // 4
#             from: name.clone(),
#             to: dest,
#             msg,
#         }).await.unwrap();
#     }
#     Ok(())
# }
#
# async fn connection_writer_loop(
#     mut messages: Receiver<String>,
#     stream: Arc<TcpStream>,
# ) -> Result<()> {
#     let mut stream = &*stream;
#     while let Some(msg) = messages.next().await {
#         stream.write_all(msg.as_bytes()).await?;
#     }
#     Ok(())
# }
#
# #[derive(Debug)]
# enum Event {
#     NewPeer {
#         name: String,
#         stream: Arc<TcpStream>,
#     },
#     Message {
#         from: String,
#         to: Vec<String>,
#         msg: String,
#     },
# }
#
# async fn broker_loop(mut events: Receiver<Event>) -> Result<()> {
#     let mut peers: HashMap<String, Sender<String>> = HashMap::new();
#
#     while let Some(event) = events.next().await {
#         match event {
#             Event::Message { from, to, msg } => {
#                 for addr in to {
#                     if let Some(peer) = peers.get_mut(&addr) {
#                         let msg = format!("from {}: {}\n", from, msg);
#                         peer.send(msg).await?
#                     }
#                 }
#             }
#             Event::NewPeer { name, stream} => {
#                 match peers.entry(name) {
#                     Entry::Occupied(..) => (),
#                     Entry::Vacant(entry) => {
#                         let (client_sender, client_receiver) = mpsc::unbounded();
#                         entry.insert(client_sender); // 4
#                         spawn_and_log_error(connection_writer_loop(client_receiver, stream)); // 5
#                     }
#                 }
#             }
#         }
#     }
#     Ok(())
# }
#
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
    drop(broker_sender); // 1
    broker_handle.await?; // 5
    Ok(())
}
```

并向经纪人：

```rust,edition2018
# extern crate async_std;
# extern crate futures;
# use async_std::{
#     io::{self, BufReader},
#     net::{TcpListener, TcpStream, ToSocketAddrs},
#     prelude::*,
#     task,
# };
# use futures::channel::mpsc;
# use futures::SinkExt;
# use std::{
#     collections::hash_map::{HashMap, Entry},
#     sync::Arc,
# };
#
# type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
# type Sender<T> = mpsc::UnboundedSender<T>;
# type Receiver<T> = mpsc::UnboundedReceiver<T>;
#
# async fn connection_writer_loop(
#     mut messages: Receiver<String>,
#     stream: Arc<TcpStream>,
# ) -> Result<()> {
#     let mut stream = &*stream;
#     while let Some(msg) = messages.next().await {
#         stream.write_all(msg.as_bytes()).await?;
#     }
#     Ok(())
# }
#
# fn spawn_and_log_error<F>(fut: F) -> task::JoinHandle<()>
# where
#     F: Future<Output = Result<()>> + Send + 'static,
# {
#     task::spawn(async move {
#         if let Err(e) = fut.await {
#             eprintln!("{}", e)
#         }
#     })
# }
#
# #[derive(Debug)]
# enum Event {
#     NewPeer {
#         name: String,
#         stream: Arc<TcpStream>,
#     },
#     Message {
#         from: String,
#         to: Vec<String>,
#         msg: String,
#     },
# }
#
async fn broker_loop(mut events: Receiver<Event>) -> Result<()> {
    let mut writers = Vec::new();
    let mut peers: HashMap<String, Sender<String>> = HashMap::new();
    while let Some(event) = events.next().await { // 2
        match event {
            Event::Message { from, to, msg } => {
                for addr in to {
                    if let Some(peer) = peers.get_mut(&addr) {
                        let msg = format!("from {}: {}\n", from, msg);
                        peer.send(msg).await?
                    }
                }
            }
            Event::NewPeer { name, stream} => {
                match peers.entry(name) {
                    Entry::Occupied(..) => (),
                    Entry::Vacant(entry) => {
                        let (client_sender, client_receiver) = mpsc::unbounded();
                        entry.insert(client_sender);
                        let handle = spawn_and_log_error(connection_writer_loop(client_receiver, stream));
                        writers.push(handle); // 4
                    }
                }
            }
        }
    }
    drop(peers); // 3
    for writer in writers { // 4
        writer.await;
    }
    Ok(())
}
```

注意一旦退出accept循环，所有通道都会发生什么情况：

1.  首先，我们删除主要经纪人的发件人。这样，当阅读器完成时，经纪人通道的发送者就没有了，香奈儿关闭了。
2.  接下来，经纪人退出`while let Some(event) = events.next().await`环。
3.  至关重要的是，在此阶段，我们放弃`peers`地图。这会降低作者的发件人。
4.  现在我们可以加入所有作家。
5.  最后，我们加入经纪人，这也保证所有写操作都已终止。
