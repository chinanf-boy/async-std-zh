# Welcome to `async-std`

`async-std`，连同其[supporting libraries][organization]是一个使您在异步编程中的生活更轻松的库。它为下游库和应用程序提供了基本的实现。名称反映了该库的方法：它尽可能与Rust主要标准库建模，用异步对应物替换所有组件。

`async-std`提供了所有重要原语的接口：文件系统操作，网络操作和并发性基础（如计时器）。它还暴露了`task`在类似于`thread`在Rust标准库中找到的模块。但是它不仅包含I / O原语，而且还包含`async/await`原语的兼容版本`Mutex`。

[organization]: https://github.com/async-rs
