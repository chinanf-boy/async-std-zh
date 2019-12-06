# async-rs/async-std [![explain]][source] [![translate-svg]][translate-list]

<!-- [ ] [![size-img]][size] -->

[explain]: http://llever.com/explain.svg
[source]: https://github.com/chinanf-boy/Source-Explain
[translate-svg]: http://llever.com/translate.svg
[translate-list]: https://github.com/chinanf-boy/chinese-translate-list
[size-img]: https://packagephobia.now.sh/badge?p=Name
[size]: https://packagephobia.now.sh/result?p=Name

「 Async version of the Rust standard library 」

[中文](./readme.md) | [english](https://github.com/async-rs/async-std)

---

## 校对 ✅

<!-- doc-templite START generated -->
<!-- repo = 'async-rs/async-std' -->
<!-- commit = '65afd41a33c32059fdd42575f8d4199fd98333ee' -->
<!-- time = '2019-11-19' -->

| 翻译的原文 | 与日期        | 最新更新 | 更多                       |
| ---------- | ------------- | -------- | -------------------------- |
| [commit]   | ⏰ 2019-11-19 | ![last]  | [中文翻译][translate-list] |

[last]: https://img.shields.io/github/last-commit/async-rs/async-std.svg
[commit]: https://github.com/async-rs/async-std/tree/65afd41a33c32059fdd42575f8d4199fd98333ee

<!-- doc-templite END generated -->

- [x] [SUMMARY](src/SUMMARY.md)
- [x] [介绍](src/introduction.zh.md)
  - [x] [欢迎来到`async-std`！](src/overview/async-std.zh.md)
  - [x] [`std::future`和`futures-rs`](src/overview/std-and-library-futures.zh.md)
  - [x] [稳定性保证](src/overview/stability-guarantees.zh.md)
- [x] [使用 async-std 的异步概念](src/concepts.zh.md)
  - [x] [Futures](src/concepts/futures.zh.md)
  - [x] [Tasks](src/concepts/tasks.zh.md)
  - [ ] [TODO：异步读/写](src/concepts/async-read-write.zh.md)
  - [ ] [TODO：Streams 和 Channels](src/concepts/streams.zh.md)
- [x] [教程：实现聊天](src/tutorial/index.zh.md)
  - [x] [规格和入门](src/tutorial/specification.zh.md)
  - [x] [编写一个 Accept 循环](src/tutorial/accept_loop.zh.md)
  - [x] [接收讯息](src/tutorial/receiving_messages.zh.md)
  - [x] [发送讯息](src/tutorial/sending_messages.zh.md)
  - [x] [连接 Readers 和 Writers](src/tutorial/connecting_readers_and_writers.zh.md)
  - [x] [全部一起](src/tutorial/all_together.zh.md)
  - [x] [干净关闭](src/tutorial/clean_shutdown.zh.md)
  - [x] [处理断开连接](src/tutorial/handling_disconnection.zh.md)
  - [x] [实现一个 Client](src/tutorial/implementing_a_client.zh.md)
- [ ] [TODO：异步模式](src/patterns.zh.md)
  - [ ] [TODO：收集的小模式](src/patterns/small-patterns.zh.md)
- [x] [安全实践](src/security/index.zh.md)
  - [x] [安全披露和政策](src/security/policy.zh.md)
- [x] [术语](src/glossary.zh.md)

### 贡献

欢迎 👏 勘误/校对/更新贡献 😊 [具体贡献请看](https://github.com/chinanf-boy/chinese-translate-list#贡献)

## 生活

[If help, **buy** me coffee —— 营养跟不上了，给我来瓶营养快线吧! 💰](https://github.com/chinanf-boy/live-need-money)

---

<h1 align="center">async-std</h1>
<div align="center">
 <strong>
   Async version of the Rust standard library
 </strong>
</div>

<br />

<div align="center">
  <!-- Crates version -->
  <a href="https://crates.io/crates/async-std">
    <img src="https://img.shields.io/crates/v/async-std.svg?style=flat-square"
    alt="Crates.io version" />
  </a>
  <!-- Downloads -->
  <a href="https://crates.io/crates/async-std">
    <img src="https://img.shields.io/crates/d/async-std.svg?style=flat-square"
      alt="Download" />
  </a>
  <!-- docs.rs docs -->
  <a href="https://docs.rs/async-std">
    <img src="https://img.shields.io/badge/docs-latest-blue.svg?style=flat-square"
      alt="docs.rs docs" />
  </a>

  <a href="https://discord.gg/JvZeVNe">
    <img src="https://img.shields.io/discord/598880689856970762.svg?logo=discord&style=flat-square"
      alt="chat" />
  </a>
</div>

<div align="center">
  <h3>
    <a href="https://docs.rs/async-std">
      API Docs
    </a>
    <span> | </span>
    <a href="https://llever.com/async-std-zh">
      Book
    </a>
    <span> | </span>
    <a href="https://github.com/async-rs/async-std/releases">
      Releases
    </a>
    <span> | </span>
    <a href="https://async.rs/contribute">
      Contributing
    </a>
  </h3>
</div>

<br/>

此箱子提供异步版本的[`std`]。它提供了您惯用的所有接口，不同的是，它是异步版本，可使用 Rust 的`async`/`await`语法。

[`std`]: https://doc.rust-lang.org/std/index.html

## 特征

- **现代：** 从头到`std::future`和`async/await`开始构建，快速的编译时间。
- **快速：**我们强大的分配器和线程池设计，可提供超高吞吐量，并具有可预见的低延迟。
- **直观：**与 stdlib 的完全对等，意味着您只需要学习一次 API。
- **明确：** [详细文档][docs]和[无障碍指南][book]让，使用异步 Rust 从未如此简单。

[docs]: https://docs.rs/async-std
[book]: https://llever.com/async-std-zh

## 例子

```rust
use async_std::task;

fn main() {
    task::block_on(async {
        println!("Hello, world!");
    })
}
```

在我们的网站中，可以在[`examples`]目录找到更多示例，包括网络和文件访问。

[`examples`]: https://github.com/async-rs/async-std/tree/master/examples

## 哲学

我们认为 Async Rust 应该像 Sync Rust 一样容易上手。我们也相信最好的 API 是您已经知道的 API。最后，我们认为为标准库提供异步副本是最好的做法，而 stdlib 成为性能和生产力两点上，一个可靠的基石。

Async-std 是该愿景的体现。它结合了，单-分配任务的创建，自适应无锁执行器，线程池和网络驱动程序，且使用 Rust 熟悉的 stdlib API，而这样的一个平滑系统，是以如此低延迟的高速度处理工作。

## 安装

用[cargo add][cargo-add]安装运行：

```sh
$ cargo add async-std
```

我们还通过 async-std 提供了一组“不稳定”功能。见[功能文档][features documentation]中，了解如何启用它们的信息。

[cargo-add]: https://github.com/killercup/cargo-edit
[features documentation]: https://docs.rs/async-std/#features

## 执照

<sup>
Licensed under either of <a href="LICENSE-APACHE">Apache License, Version
2.0</a> or <a href="LICENSE-MIT">MIT license</a> at your option.
</sup>

<br/>

<sub>
Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in this crate by you, as defined in the Apache-2.0 license, shall
be dual licensed as above, without any additional terms or conditions.
</sub>
