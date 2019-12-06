# Tasks

既然我们知道 Future 是什么，就想要撸它！

在`async-std`，[`tasks`][tasks]模块负责这项行为。最简单的方法是使用`block_on`函数：

```rust,edition2018
# extern crate async_std;
use async_std::{fs::File, io, prelude::*, task};

async fn read_file(path: &str) -> io::Result<String> {
    let mut file = File::open(path).await?;
    let mut contents = String::new();
    file.read_to_string(&mut contents).await?;
    Ok(contents)
}

fn main() {
    let reader_task = task::spawn(async {
        let result = read_file("data.csv").await;
        match result {
            Ok(s) => println!("{}", s),
            Err(e) => println!("Error reading file: {:?}", e)
        }
    });
    println!("Started task!");
    task::block_on(reader_task);
    println!("Stopped task!");
}
```

这就要求 runtime 先放进`async_std`烤一烤，再去执行这份会读取文件的代码。不过停一停，让我们一个个来，从内到外。

```rust,edition2018
# extern crate async_std;
# use async_std::{fs::File, io, prelude::*, task};
#
# async fn read_file(path: &str) -> io::Result<String> {
#     let mut file = File::open(path).await?;
#     let mut contents = String::new();
#     file.read_to_string(&mut contents).await?;
#     Ok(contents)
# }
#
async {
    let result = read_file("data.csv").await;
    match result {
        Ok(s) => println!("{}", s),
        Err(e) => println!("Error reading file: {:?}", e)
    }
};
```

这是一个`async` _代码块(block)_。 async block 是调用`async`函数的必要条件，所以同样，还会指示编译器，去包括所有相关指令。在 Rust 中，所有 block 都返回一个值，并且`async` block 就是会返回`Future`值。

但让我们来看看有趣的部分：

```rust,edition2018
# extern crate async_std;
# use async_std::task;
task::spawn(async { });
```

`spawn`拿了一个`Future`，并在`Task`上开始 running。它返回一个`JoinHandle`。 这时的 Future 在 Rust 社区中，有时被称为*冻(cold)* Futures。你需要一些东西来管理/激活他们。为了运行一个 Future，可能需要一些额外的记录信息，例如，它是 running 还是完成的状态，和它放在内存中的位置以及当前的状态。这个记录部分被抽象成`Task`。

一个`Task`类似于`Thread`，但有一些小的区别：它将由程序，而不是操作系统内核调度，如果遇到需要等待(wait)的点，程序本身承担起再次唤醒它的负责 —— 这个我们稍后再谈。一个`async_std` task 也可以有一个 name 和一个 ID，就像线程一样。

现在，已给出足够的知识了：你知道了，一旦`spawn`(生)出一个 task，它将继续在后台运行。这个`JoinHandle`它本身就是一个 Future，一旦`Task`已经接近完成了，自然跟随着完成。这与`threads`以及`join`函数很像，我们现在可以通过在'控制杆'上调用`block_on`，*阻塞*程序（或调用线程，具体来说的话）并等待它完成。

## Tasks in `async_std`

`async_std`中的 Tasks 是核心抽象之一。很像 Rust 的`thread`，他们在原始概念的基础上，提供了一些实用的函数。`Tasks`与 runtime 之间存有关系，但它们本身是独立的。`async_std`任务具有许多理想的特性：

- 它们被分配到一个单独的分配(single allocation)中
- 所有 tasks 都有*后门通道*，它允许它们通过`JoinHandle` 传播 results 和 errors 到那 spawning task。
- 它们为调试提供了有用的元数据
- 它们支持 task 本地存储

`async_std`的 task API 为您处理，一个后台的 runtime 的设置和拆卸，而不用依赖于一个显式启动的 runtime。

## Blocking

假设，`Task`是*同时*运行的，(可能)具有共享执行的潜在线程。这意味着，能够阻塞*操作系统线程*的操作，例如`std::thread::sleep`或是 Rust `std` 库的 io 函数，会让*共享此线程的所有 tasks，停止执行*。其他库（如数据库驱动程序）也有类似的行为。请注意*阻塞当前线程*它本身并不是不良行为，只是与`async-std`的并发执行模型不能混用。本质上来说，千万不要这样做：

```rust,edition2018
# extern crate async_std;
# use async_std::task;
fn main() {
    task::block_on(async {
        // this is std::fs, which blocks
        std::fs::read_to_string("test_file");
    })
}
```

如果要混合不同操作，请考虑将这些阻塞操作，放在单独的`thread`上。

## Errors and panics

Tasks 通过正常模式报告错误：如果它们是错误的，那么它们的`Output`应该是某`Result<T,E>`类型。

万一`panic`，行为的不同之处取决于，是否有一个合理的部分处理`panic`。 如果没有，程序*中止*.

实际上，这意味着，`block_on`会将 panics 传播到阻塞组件：

```rust,edition2018,should_panic
# extern crate async_std;
# use async_std::task;
fn main() {
    task::block_on(async {
        panic!("test");
    });
}
```

```text
thread 'async-task-driver' panicked at 'test', examples/panic.rs:8:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace.
```

在 panicing 阶段，一个 spawned task 将中止：

```rust,edition2018,should_panic
# extern crate async_std;
# use async_std::task;
# use std::time::Duration;
task::spawn(async {
    panic!("test");
});

task::block_on(async {
    task::sleep(Duration::from_millis(10000)).await;
})
```

```text
thread 'async-task-driver' panicked at 'test', examples/panic.rs:8:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace.
Aborted (core dumped)
```

一开始这可能看起来很奇怪，但另一个选择是默默忽略 spawned task 中的 panics。当前的行为是可以改变的，具体通过捕获 spawned task 中的 panics，并用自定义行为对其作出反应。这为用户提供了，处理 panics 策略的选择。

## Conclusion

`async_std`附带一个有用的`Task`类型，与`std::thread` API 类似。 它以结构化和明确性的方式，涵盖了错误和 panics 行为。

Tasks 作为独立的并发单元，有时，它们之间是需要通信的。这个 moment，`Stream`出现了。

[tasks]: https://docs.rs/async-std/latest/async_std/task/index.html
