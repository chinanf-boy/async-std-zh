## Connecting Readers and Writers

那么我们如何确保，`connection_loop`读到的信息流入对应的`connection_writer_loop`？我们应该以某种方式保有一个`peers: HashMap<String, Sender<String>>` map，一个允许客户查找目标频道的'地图'。但是，此 map 会有些共享的可变状态，因此我们必须裹入一个`RwLock`，并回答个棘手的问题，即如果客户端在收到消息的同时，加入该怎么办。

使状态的推理更简单的一个技巧，来自参与者模型(actor model)。我们可以创建一个专用的代理人（dedicated broker）任务，该任务拥有`peers` map ，并通过 channels 与其他任务进行通信。通过将`peers`隐藏在，如一个 "actor" 任务内，我们无需使用 mutxes，并且还明确了序列化点。两件事件，“Bob 将消息发送给 Alice”和“Alice 加入”的顺序，由代理人的事件队列中，相应事件的顺序确定。

```rust,edition2018
# extern crate async_std;
# extern crate futures;
# use async_std::{
#     net::TcpStream,
#     prelude::*,
#     task,
# };
# use futures::channel::mpsc;
# use futures::sink::SinkExt;
# use std::sync::Arc;
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
use std::collections::hash_map::{Entry, HashMap};

#[derive(Debug)]
enum Event { // 1
    NewPeer {
        name: String,
        stream: Arc<TcpStream>,
    },
    Message {
        from: String,
        to: Vec<String>,
        msg: String,
    },
}

async fn broker_loop(mut events: Receiver<Event>) -> Result<()> {
    let mut peers: HashMap<String, Sender<String>> = HashMap::new(); // 2

    while let Some(event) = events.next().await {
        match event {
            Event::Message { from, to, msg } => {  // 3
                for addr in to {
                    if let Some(peer) = peers.get_mut(&addr) {
                        let msg = format!("from {}: {}\n", from, msg);
                        peer.send(msg).await?
                    }
                }
            }
            Event::NewPeer { name, stream } => {
                match peers.entry(name) {
                    Entry::Occupied(..) => (),
                    Entry::Vacant(entry) => {
                        let (client_sender, client_receiver) = mpsc::unbounded();
                        entry.insert(client_sender); // 4
                        spawn_and_log_error(connection_writer_loop(client_receiver, stream)); // 5
                    }
                }
            }
        }
    }
    Ok(())
}
```

1.  代理人（**Broker**）应处理两种类型的事件：一个消息，或一个新端点的到来。
2.  代理人的内部状态是`HashMap`。请留意，在这里我们不需要`Mutex`，且可以自信地讲出，在代理人循环的每次迭代中，当前的端点集合(set of peers)是什么
3.  为了处理消息，我们通过 channel，将其发送到每个目的地
4.  为了处理新的 peer，我们首先在 peer 的 map 中，注册它...
5.  ... 然后 spawn(孵) 一个专用任务（dedicated task），将消息实际写入 socket。
