## Writing an Accept Loop

让我们实现服务器的支架：一个将TCP套接字绑定到地址并开始接受连接的循环。

首先，让我们添加所需的导入样板：

```rust,edition2018
# extern crate async_std;
use async_std::{
    prelude::*, // 1
    task, // 2
    net::{TcpListener, ToSocketAddrs}, // 3
};

type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>; // 4
```

1.  `prelude`重新导出与期货和数据流一起使用所需的某些特征。
2.  的`task`模块大致对应于`std::thread`模块，但任务轻得多。一个线程可以运行许多任务。
3.  对于插座类型，我们使用`TcpListener`从`async_std`，就像`std::net::TcpListener`，但不阻塞并且使用`async`API。
4.  在此示例中，我们将跳过实现全面的错误处理。为了传播错误，我们将使用装箱的错误特征对象。你知道吗`From<&'_ str> for Box<dyn Error>`stdlib中的实现，它允许您将字符串与`?`操作员？

现在我们可以编写服务器的accept循环：

```rust,edition2018
# extern crate async_std;
# use async_std::{
#     net::{TcpListener, ToSocketAddrs},
#     prelude::*,
# };
#
# type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
#
async fn accept_loop(addr: impl ToSocketAddrs) -> Result<()> { // 1

    let listener = TcpListener::bind(addr).await?; // 2
    let mut incoming = listener.incoming();
    while let Some(stream) = incoming.next().await { // 3
        // TODO
    }
    Ok(())
}
```

1.  我们标记`accept_loop`作为`async`，这使我们可以使用`.await`内部语法。
2.  `TcpListener::bind`通话返回了一个未来，我们`.await`提取`Result`， 然后`?`得到一个`TcpListener`。注意如何`.await`和`?`一起很好地工作。就是这样`std::net::TcpListener`可以，但是`.await`添加。的镜像API`std`是一个明确的设计目标`async_std`。
3.  在这里，我们想迭代传入的套接字，就像在套接字中那样`std`：

```rust,edition2018,should_panic
let listener: std::net::TcpListener = unimplemented!();
for stream in listener.incoming() {
}
```

不幸的是，这不适用于`async`但是，因为不支持`async`语言中的for循环。因此，我们必须使用`while let Some(item) = iter.next().await`图案。

最后，让我们添加main：

```rust,edition2018
# extern crate async_std;
# use async_std::{
#     net::{TcpListener, ToSocketAddrs},
#     prelude::*,
#     task,
# };
#
# type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
#
# async fn accept_loop(addr: impl ToSocketAddrs) -> Result<()> { // 1
#     let listener = TcpListener::bind(addr).await?; // 2
#     let mut incoming = listener.incoming();
#     while let Some(stream) = incoming.next().await { // 3
#         // TODO
#     }
#     Ok(())
# }
#
// main
fn run() -> Result<()> {
    let fut = accept_loop("127.0.0.1:8080");
    task::block_on(fut)
}
```

意识到这一点的关键是在Rust中，与其他语言不同，调用异步函数可以**不**运行任何代码。异步功能仅构造期货，它们是惰性状态机。要开始通过异步功能逐步遍历未来的状态机，您应该使用`.await`。在非异步函数中，执行Future的一种方法是将其交给执行者。在这种情况下，我们使用`task::block_on`在当前线程上执行Future并阻塞直到完成。
