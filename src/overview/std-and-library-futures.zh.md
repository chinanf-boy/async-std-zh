# `std::future` and `futures-rs`

Rust有两种类型，通常称为`Future`：

-   首先是`std::future::Future`来自Rust的[standard library](https://doc.rust-lang.org/std/future/trait.Future.html)。
-   第二个是`futures::future::Future`来自[futures-rs crate](https://docs.rs/futures/0.3/futures/prelude/trait.Future.html)。

在定义的未来[futures-rs](https://docs.rs/futures/0.3/futures/prelude/trait.Future.html)板条箱是该类型的原始实现。要启用`async/await`语法，未来的主要特征被转移到Rust的标准库中，并成为`std::future::Future`。从某种意义上说`std::future::Future`可以看作是`futures::future::Future`。

了解两者之间的区别至关重要`std::future::Future`和`futures::future::Future`，以及`async-std`带给他们。在自身，`std::future::Future`不是您要与用户互动的内容-除非通过调用`.await`在上面。内部运作`std::future::Future`实施人员最感兴趣的是`Future`。没错-这非常有用！以前定义的大多数功能`Future`它本身已经被移到一个称为[`FuturesExt`](https://docs.rs/futures/0.3/futures/future/trait.FutureExt.html). 根据这些信息，您可以推断`futures`库作为核心Rust异步特性的扩展。

与`futures`,`async-std`再出口核心`std::future::Future`键入。您可以主动选择`futures-preview`将其添加到`Cargo.toml`以及导入`FuturesExt`.

## Interfaces and Stability

 `async-std`旨在成为一个稳定可靠的图书馆，处于锈标准图书馆的水平。这也意味着我们不依赖`futures`我们接口的库。然而，我们很感激许多用户已经开始喜欢这种方便`futures-rs`带来。因此，`async-std`实现所有`futures`其类型的特征。

幸运的是，上面的方法给了你完全的灵活性。如果你非常关心稳定性，你可以使用`async-std`照原样。如果你喜欢`futures`库接口，将它们链接到中。两种用途都是一流的。

## `async_std::future`

我们认为，有一些支持功能对于处理任何类型的期货都很重要。这些可以在`async_std::future`模块和都包含在我们的稳定性保证中。

## Streams and Read/Write/Seek/BufRead traits

由于Rust编译器的局限性，目前在`async_std`，但不能由用户自己实现。
