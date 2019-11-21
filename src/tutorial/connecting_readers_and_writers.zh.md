## Connecting Readers and Writers

那么我们如何确保读入邮件`connection_loop`流入相关`connection_writer_loop`？我们应该以某种方式保持`peers: HashMap<String, Sender<String>>`允许客户查找目标频道的地图。但是，此映射会有些共享的可变状态，因此我们必须包装一个`RwLock`并回答棘手的问题，即如果客户在收到消息的同时加入该怎么办。

使状态的推理更简单的一个技巧来自参与者模型。我们可以创建一个专门的经纪人任务，该任务拥有`peers`通过渠道映射并与其他任务进行通信。通过隐藏`peers`在这样的“ actor”任务中，我们无需使用互斥对象，并且还明确了序列化点。事件“鲍勃将消息发送给爱丽丝”的顺序和“爱丽丝加入”的顺序由代理的事件队列中相应事件的顺序确定。

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

1.  代理应处理两种类型的事件：消息或新对等体的到达。
2.  经纪人的内部状态是`HashMap`。请注意，我们不需要`Mutex`在这里，可以自信地说，在代理循环的每次迭代中，当前的对等体是什么
3.  为了处理消息，我们通过渠道将其发送到每个目的地
4.  为了处理新的对等方，我们首先在对等方的地图中注册它...
5.  ...，然后产生一个专用任务，将消息实际写入套接字。
