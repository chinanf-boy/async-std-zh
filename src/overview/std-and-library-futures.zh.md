# `std::future` and `futures-rs`

Rust 的`Future`，有两个不同的来源：

- 首先是`std::future::Future`，来自 Rust 的[标准库](https://doc.rust-lang.org/std/future/trait.Future.html)。
- 第二个是`futures::future::Future`，来自[futures-rs 箱子](https://docs.rs/futures/0.3/futures/prelude/trait.Future.html)。

而定义在[futures-rs](https://docs.rs/futures/0.3/futures/prelude/trait.Future.html)箱子中的 Future，是该类型的原始实现。（因为）要启用`async/await`语法，Future 的主要 trait 被转移到 Rust 的标准库中，成为了`std::future::Future`。从某种意义上说`std::future::Future`就可以看作是`futures::future::Future`。

了解`std::future::Future`和`futures::future::Future`两者之间区别，以及`async-std`榜上他俩的方法至关重要。在`std::future::Future`里面，其实并没有您作为一名用户，想要交互的内容 —— 除非通过在上面调用`.await`。`std::future::Future`的内部运作，对 implementing `Future`的人来说，大概率是最感兴趣的。不要混淆 —— 这点非常有用！以前定义在`Future`的大多数功能性代码，本身已经被移到一个称为[`FuturesExt`](https://docs.rs/futures/0.3/futures/future/trait.FutureExt.html) trait。根据这些信息，您或许可以推断`futures`库的角色是，作为 core Rust 异步特性的扩展。

传承`futures`,`async-std`也会再出口 core `std::future::Future`类型。您可以自由地操作这项扩展，具体是将`futures-preview`添加到`Cargo.toml`，以及导入`FuturesExt`。

## Interfaces and Stability

`async-std`旨在成为一个稳定可靠的库，处于 Rust 标准库的水平。这也意味着，我们接口不依赖`futures`库。然而，我们很感激许多用户已经开始喜欢这`futures-rs`带来的便捷。因此，`async-std`为它的类型实现所有`futures` trait。

幸运的是，上面的方法给了你完全的灵活性。如果你非常关心稳定性，你可以照原样使用`async-std`。如果你更喜欢`futures`库接口，赶上。两种用法都是一流的。

## `async_std::future`

我们认为，有一些支持的功能对 Futures 都很重要，不管是什么类型。这些可以在`async_std::future`模块，且有我们的稳定性保证。

## Streams and Read/Write/Seek/BufRead traits

由于 Rust 编译器的局限性，这些目前仅实现在`async_std`，但不能由用户自己实现。
