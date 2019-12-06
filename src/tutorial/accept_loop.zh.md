## Writing an Accept Loop

让我们实现服务器的支架：一个循环，它将 TCP socket 绑定到地址，并开始接受连接。

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

1.  `prelude`重新导出与 futures 和 streams 一起使用，所需的某些 trait。
2.  `task`模块大致对应`std::thread`模块，但 task 轻得多。一个线程(thread)可以运行许多 tasks。
3.  对于 socket 类型，我们使用`TcpListener`，来自`async_std`，它就像`std::net::TcpListener`，但非-阻塞，并且使用`async`API。
4.  在此示例中，我们会跳过实现全面的错误处理。为了传播错误，我们将使用装箱的（boxed） error trait 对象。你知道 stdlib 中，有个`From<&'_ str> for Box<dyn Error>`实现吗，它允许您将字符串`?`操作符？

现在我们可以编写服务器的 接收 循环：

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

1.  我们将`accept_loop`函数标记上了`async`，这使我们可以在内部使用`.await`语法。
2.  `TcpListener::bind` call 返回了一个 Future，此处，我们会`.await`(等待)，提取出`Result`，然后`?`得到一个`TcpListener`。
    注意`.await`和`?`的良好合作。这与`std::net::TcpListener`的工作如此相同，但多个`.await`而已。
    对`std` API 施行镜像魔法，是`async_std`的一个明确的设计目标。
3.  在这里，我们想迭代传入的 socket，就像在常规`std`中的那样：

```rust,edition2018,should_panic
let listener: std::net::TcpListener = unimplemented!();
for stream in listener.incoming() {
}
```

不幸的是，目前这还不适用于`async`，因为在语言中，还不支持`async` for 循环。因此，我们必须手动实现循环，通过使用`while let Some(item) = iter.next().await`模式。

最后，让我们添加 main：

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

意识到这一点的关键是，Rust 与其他语言不同，调用异步函数确实是**不**运行任何代码的。异步函数仅构造 Future，它们是惰性状态机。要在一个异步函数中，开始逐步遍历 Future 状态机，您应该使用`.await`。在非异步函数中，执行 Future 的一种方法是将其交给 executor。在这种情况下，我们会使用`task::block_on`，在当前线程上对 Future 执行并阻塞，直到它完成。
