# Tasks

既然我们知道未来是什么，我们就要经营它们！

在`async-std`，和[`tasks`][tasks]模块对此负责。最简单的方法是使用`block_on`功能：

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

这就要求运行时`async_std`执行读取文件的代码。不过，让我们一个接一个，从里到外。

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

这是一个`async` *块*. 异步块是调用`async`函数，并将指示编译器包含所有相关的指令。在Rust中，所有块都返回一个值，并且`async`块碰巧返回此类值`Future`.

但让我们来看看有趣的部分：

```rust,edition2018
# extern crate async_std;
# use async_std::task;
task::spawn(async { });
```

`spawn`拿一个`Future`开始在`Task`. 它返回一个`JoinHandle`. 生锈的未来有时被称为*寒冷的*未来。你需要一些东西来管理他们。为了运行一个未来，可能需要一些额外的簿记，例如它是运行的还是完成的，它放在内存中的位置以及当前的状态。这个记账部分被抽象成`Task`.

一个`Task`类似于`Thread`，但有一些小的区别：它将由程序而不是操作系统内核调度，如果遇到需要等待的点，程序本身将负责再次唤醒它。我们稍后再谈。一个`async_std`任务也可以有名称和ID，就像线程一样。

现在，只要你知道`spawn`一个任务，它将继续在后台运行。这个`JoinHandle`它本身就是一个未来，一旦`Task`已经接近尾声了。很像`threads`以及`join`函数，我们现在可以调用`block_on`把手放在*块*程序（或调用线程，具体来说）并等待它完成。

## Tasks in `async_std`

中的任务`async_std`是核心抽象之一。很像铁锈`thread`他们在原始概念的基础上提供了一些实用的功能。`Tasks`与运行时有关系，但它们本身是独立的。`async_std`任务具有许多理想的属性：

-   它们被分配到一个单独的分配中
-   所有任务都有*反馈语*，它允许它们通过`JoinHandle`
-   它们为调试提供了有用的元数据
-   它们支持任务本地存储

`async_std`s任务API为您处理备份运行时的设置和拆卸，而不依赖于显式启动的运行时。

## Blocking

`Task`假设s运行*同时*，可能是通过共享执行线程。这意味着阻塞*操作系统线程*，例如`std::thread::sleep`或是铁锈的io功能`std`图书馆将*停止执行共享此线程的所有任务*. 其他库（如数据库驱动程序）也有类似的行为。请注意*阻塞当前线程*它本身并不是不良行为，只是与`async-std`. 基本上，千万不要这样做：

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

如果要混合操作类型，请考虑将这些阻塞操作放在单独的`thread`.

## Errors and panics

任务通过正常模式报告错误：如果它们是错误的，那么`Output`应该是善良的`Result<T,E>`.

万一`panic`，行为的不同取决于是否有一个合理的部分处理`panic`. 如果没有，程序*中止*.

实际上，这意味着`block_on`将恐慌传播到阻塞组件：

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

在恐慌生成的任务时将中止：

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

一开始这可能看起来很奇怪，但另一个选择是默默地忽略衍生任务中的恐慌。当前的行为可以通过捕获派生任务中的恐慌并对自定义行为作出反应来改变。这为用户提供了恐慌处理策略的选择。

## Conclusion

`async_std`附带一个有用的`Task`与类似于`std::thread`. 它以结构化和明确的方式涵盖了错误和恐慌行为。

任务是独立的并发单元，有时它们需要通信。就在那里`Stream`我们进来。

[tasks]: https://docs.rs/async-std/latest/async_std/task/index.html
