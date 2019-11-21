## Receiving messages

让我们实现协议的接收部分。我们要：

1.  拆分传入`TcpStream`上`\n`并将字节解码为utf-8
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

1.  我们用`task::spawn`函数产生一个独立的任务以与每个客户端一起工作。也就是说，在接受客户之后`accept_loop`立即开始等待下一个。这是事件驱动的体系结构的核心优势：我们同时为许多客户端提供服务，而无需花费很多硬件线程。

2.  幸运的是，“将字节流分成几行”功能已经实现。`.lines()`呼叫会传回`String`的。

3.  我们得到第一行-登录

4.  而且，我们再次实现了手动异步for循环。

5.  最后，我们将每一行解析为目标登录名和消息本身的列表。

## Managing Errors

上述解决方案中的一个严重问题是，尽管我们在`connection_loop`，然后我们将错误放到地板上！那是，`task::spawn`仅在加入后才立即返回错误（不能，它需要先运行future才能完成）。我们可以通过等待任务加入来“修复”它，如下所示：

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

的`.await`等到客户端完成，然后`?`传播结果。

但是，此解决方案有两个问题！*第一*，因为我们立即等待客户端，所以我们一次只能处理一个客户端，这完全违反了异步的目的！*第二*，如果客户端遇到IO错误，则整个服务器都会立即退出。也就是说，一个同龄人的不稳定的互联网连接使整个聊天室瘫痪了！

在这种情况下，处理客户端错误的正确方法是记录它们，并继续为其他客户端提供服务。因此，我们为此使用一个辅助函数：

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
