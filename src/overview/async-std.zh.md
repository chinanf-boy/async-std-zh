# Welcome to `async-std`

`async-std`，连同其[在支持的库][organization]，使您在异步编程中的生活更轻松。它为下游库和应用程序提供了基本的实现。名称反映了该库的方法：它尽可能遵照 Rust 主要标准库的模样，用 async 对等替换所有组件。

`async-std`提供了所有重要原语的接口：文件系统操作，网络操作和并发性基础（如计时器）。它还暴露了`task`，类似于在 Rust 标准库中的`thread`模块。但是它不仅包含 I/O 原语，而且还包含`async/await`兼容版本的原语，如`Mutex`。

> 译：primitives/原语/元语 - 原始的，早期的，固有的

[organization]: https://github.com/async-rs
