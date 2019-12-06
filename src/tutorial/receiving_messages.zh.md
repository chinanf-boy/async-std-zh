## Receiving messages

让我们实现协议的接收部分。我们要：

1.  将传入的`TcpStream`用`\n`拆开，并将字节解码为 utf-8
2.  将第一行解释为登录名
3.  将其余的行解析为`login: message`

```rust,edition2018
# extern crate async_std;
# use async_std::{
#     io::BufReader,
#     net::{TcpListener, TcpStream, ToSocketAddrs},
#     prelude::*,
#     task,
# };
#
# type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
#
async fn accept_loop(addr: impl ToSocketAddrs) -> Result<()> {
    let listener = TcpListener::bind(addr).await?;
    let mut incoming = listener.incoming();
    while let Some(stream) = incoming.next().await {
        let stream = stream?;
        println!("Accepting from: {}", stream.peer_addr()?);
        let _handle = task::spawn(connection_loop(stream)); // 1
    }
    Ok(())
}

async fn connection_loop(stream: TcpStream) -> Result<()> {
    let reader = BufReader::new(&stream); // 2
    let mut lines = reader.lines();

    let name = match lines.next().await { // 3
        None => Err("peer disconnected immediately")?,
        Some(line) => line?,
    };
    println!("name = {}", name);

    while let Some(line) = lines.next().await { // 4
        let line = line?;
        let (dest, msg) = match line.find(':') { // 5
            None => continue,
            Some(idx) => (&line[..idx], line[idx + 1 ..].trim()),
        };
        let dest: Vec<String> = dest.split(',').map(|name| name.trim().to_string()).collect();
        let msg: String = msg.trim().to_string();
    }
    Ok(())
}
```

1.  我们用`task::spawn`函数 孵 一个独立的任务，让它与每个客户端一起工作。也就是说，在接受客户之后`accept_loop`能立即开始等待下一个。这是事件驱动的体系结构的核心优势：我们同时为许多客户端提供服务，而无需花费很多硬件线程。

2.  幸运的是，“将字节流分成几行”的功能已经实现。`.lines()`调用会传回`String`的数据流(stream)。

3.  我们得到第一行 -- login

4.  而且，我们再次手动实现了一个异步 for 循环。

5.  最后，我们将每一行解析为目标登录名和消息本身的列表。

## Managing Errors

上述解决方案中的一个严重问题是，尽管我们在`connection_loop`正确传播了 errors，之后我们却将错误扔到了地上！原因是，`task::spawn`是不会立即返回错误（它做不到，它需要先让 future 完成运行）。而我们可以通过，等待 task 能可以加入了，来“fix”它，如下所示：

```rust,edition2018
# #![feature(async_closure)]
# extern crate async_std;
# use async_std::{
#     io::BufReader,
#     net::{TcpListener, TcpStream, ToSocketAddrs},
#     prelude::*,
#     task,
# };
#
# type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
#
# async fn connection_loop(stream: TcpStream) -> Result<()> {
#     let reader = BufReader::new(&stream); // 2
#     let mut lines = reader.lines();
#
#     let name = match lines.next().await { // 3
#         None => Err("peer disconnected immediately")?,
#         Some(line) => line?,
#     };
#     println!("name = {}", name);
#
#     while let Some(line) = lines.next().await { // 4
#         let line = line?;
#         let (dest, msg) = match line.find(':') { // 5
#             None => continue,
#             Some(idx) => (&line[..idx], line[idx + 1 ..].trim()),
#         };
#         let dest: Vec<String> = dest.split(',').map(|name| name.trim().to_string()).collect();
#         let msg: String = msg.trim().to_string();
#     }
#     Ok(())
# }
#
# async move |stream| {
let handle = task::spawn(connection_loop(stream));
handle.await
# };
```

`.await`等到客户端完成，然后`?`传播结果。

但是，此解决方案有两个问题！_第一_，因为我们立即等待客户端，所以我们一次只能处理一个客户端，这完全违反了异步的目的！_第二_，如果客户端遇到 IO 错误，则整个服务器都会立即退出。也就是说，仅一个端点的不稳定互联网连接，就可以让整个聊天室瘫痪了！

在这种情况下，处理客户端错误的正确方法是记录它们，并继续为其他客户端提供服务。为此，我们使用一个辅助函数：

```rust,edition2018
# extern crate async_std;
# use async_std::{
#     io,
#     prelude::*,
#     task,
# };
fn spawn_and_log_error<F>(fut: F) -> task::JoinHandle<()>
where
    F: Future<Output = io::Result<()>> + Send + 'static,
{
    task::spawn(async move {
        if let Err(e) = fut.await {
            eprintln!("{}", e)
        }
    })
}
```
