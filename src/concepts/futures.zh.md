# Futures

关于Rust的值得注意的一点是[*fearless concurrency*](https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html)。这就是应该授权您在不放弃安全性的情况下执行并发任务的想法。同样，Rust是一种低级语言，它是关于无所畏惧的并发*没有选择具体的实施策略*。这意味着我们*必须*对策略进行抽象，以允许选择*后来*，如果我们想以任何方式在不同策略的用户之间共享代码。

期货摘要*计算*。它们描述了“什么”，与“哪里”和“何时”无关。为此，他们的目标是将代码分解为可组合的小动作，然后由我们系统的一部分执行。让我们浏览一下计算事物以找到可以抽象的地方的含义。

## Send and Sync

幸运的是，并发Rust已经有两个众所周知的有效概念，它们抽象了程序并发部分之间的共享：`Send`和`Sync`。值得注意的是，`Send`和`Sync`特质抽象*策略*并发工作，组成整齐，并且不规定实现。

快速摘要：

-   `Send`摘要结束*传递数据*在一个计算中进行另一个并发计算（我们称其为接收方），而在发送方失去对它的访问。在许多编程语言中，通常都采用这种策略，但是缺少语言方面的支持，因此希望您自己执行“丢失访问”行为。这是漏洞的常规来源：发件人保留发送内容的句柄，甚至在发送后也可以使用它们。Rust通过使这种行为已知来减轻此问题。类型可以是`Send`（通过实施适当的标记特征），是否允许将它们发送出去，所有权和借用规则会阻止后续访问。

-   `Sync`关于*共享数据*在程序的两个并发部分之间。这是另一种常见的模式：由于写入存储位置或在另一方正在写入时进行读取本质上是不安全的，因此需要通过同步来缓和这种访问。[^1]双方有很多共同的方式来达成共识，即不要同时使用内存中的同一部分，例如互斥锁和自旋锁。同样，Rust为您提供了（安全！）不关心的选项。Rust使您能够表达某些东西*需要*同步，但不具体说明*怎么样*。

注意我们如何避免出现类似*“线”*，而是选择了“计算”。的全部力量`Send`和`Sync`是他们减轻了您的学习负担*什么*分享。在实现时，您只需要知道哪种共享方法适用于当前类型。这使推理保持局部性，不受该类型用户以后使用的任何实现的影响。

`Send`和`Sync`可以以有趣的方式进行合成，但这超出了本文的范围。您可以在[Rust Book][rust-book-sync]。

[rust-book-sync]: https://doc.rust-lang.org/stable/book/ch16-04-extensible-concurrency-sync-and-send.html

总结：Rust使我们能够安全地抽象并发程序的重要属性及其数据共享。它以非常轻巧的方式进行。语言本身只知道两个标记`Send`和`Sync`并尽可能地通过派生它们本身来帮助我们。剩下的就是图书馆的问题。

## An easy view of computation

虽然计算是写整体的主题[book](https://computationbook.com/)大约，一个非常简化的视图对我们来说足够了：一系列可组合的操作，这些操作可以基于决策进行分支，相继执行并产生结果或产生错误

## Deferring computation

正如刚才提到的，`Send`和`Sync`关于数据。但是程序不仅涉及数据，还涉及*计算*数据。那就是[`Futures`][futures]做。在下一章中，我们将仔细研究它的工作方式。让我们看看期货允许我们用英语表达什么。期货出自此计划：

-   做X
-   如果X成功，则执行Y

向：

-   开始做X
-   一旦X成功，就开始做Y

还记得前言中有关“延迟计算”的话题吗？仅此而已。而不是告诉计算机执行什么和决定什么*现在*，您可以告诉它要开始做什么以及如何对...中的潜在事件做出反应...`Future`。

[futures]: https://doc.rust-lang.org/std/future/trait.Future.html

## Orienting towards the beginning

让我们看一个简单的函数，特别是返回值：

```rust,edition2018
# use std::{fs::File, io, io::prelude::*};
#
fn read_file(path: &str) -> io::Result<String> {
    let mut file = File::open(path)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}
```

您可以随时调用它，因此可以完全控制何时调用它。但这就是问题所在：调用它的那一刻，您就将控制权转移到被调用的函数，直到它最终返回一个值。请注意，此返回值是关于过去的。过去有一个缺点：已经做出了所有决定。它有一个优点：结果可见。我们可以解开程序过去的计算结果，然后决定如何处理它。

但是我们想抽象*计算*并让其他人选择运行方式。从根本上说，这与一直查看先前的计算结果不兼容。所以，让我们找到一个类型*描述*不运行它的计算。让我们再次看一下函数：

```rust,edition2018
# use std::{fs::File, io, io::prelude::*};
#
fn read_file(path: &str) -> io::Result<String> {
    let mut file = File::open(path)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}
```

说到时间，我们只能采取行动*之前*调用函数或*后*返回的函数。这是不可取的，因为它需要我们做某事的能力*而*它运行。使用并行代码时，这将使我们能够在首次运行时启动并行任务（因为我们放弃了控制权）。

这是我们可以达到的时刻[threads](https://en.wikipedia.org/wiki/Thread_)。但是线程是一个非常特定的并发原语，我们说过我们正在寻找抽象。

我们正在寻找的东西代表着为实现结果而正在进行的工作。每当我们在Rust中说“某事”时，我们几乎总是意味着一种特质。让我们从一个不完整的定义开始`Future`特征：

```rust,edition2018
# use std::{pin::Pin, task::{Context, Poll}};
#
trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}
```

仔细观察，我们看到以下内容：

-   它是通用的`Output`。
-   它提供了一个称为`poll`，这使我们可以检查当前计算的状态。
-   （忽视`Pin`和`Context`目前，您不需要它们进行高级了解。）

每次致电`poll()`可能导致以下两种情况之一：

1.  计算完成，`poll`将返回[`Poll::Ready`](https://doc.rust-lang.org/std/task/enum.Poll.html#variant.Ready)
2.  计算尚未完成执行，它将返回[`Poll::Pending`](https://doc.rust-lang.org/std/task/enum.Poll.html#variant.Pending)

这使我们可以从外部检查是否`Future`仍然有未完成的工作，或者最终完成了，可以给我们带来价值。最简单（但效率不高）的方法是不断循环轮询期货。可能存在优化，这就是好的运行时为您所做的。请注意，`poll`情况1发生后再次可能会导致行为混乱。见[futures-docs](https://doc.rust-lang.org/std/future/trait.Future.html)有关详细信息。

## Async

而`Future`特质在Rust中已经存在了一段时间，构建和描述它们很不方便。为此，Rust现在具有特殊的语法：`async`。上面的示例，使用`async-std`，如下所示：

```rust,edition2018
# extern crate async_std;
# use async_std::{fs::File, io, io::prelude::*};
#
async fn read_file(path: &str) -> io::Result<String> {
    let mut file = File::open(path).await?;
    let mut contents = String::new();
    file.read_to_string(&mut contents).await?;
    Ok(contents)
}
```

差别很小，对吧？我们要做的只是标记功能`async`并插入2个特殊命令：`.await`。

这个`async`函数设置延迟计算。调用此函数时，将产生一个`Future<Output = io::Result<String>>`而不是立即返回`io::Result<String>`。（或者，更准确地说，为您生成一个实现`Future<Output = io::Result<String>>`）

## What does `.await` do?

的`.await`postfix完全符合其提示：在您使用它后，代码将一直等到所请求的操作（例如，打开文件或读取其中的所有数据）完成。的`.await?`并不特殊，只是应用`?`运算符的结果`.await`。那么，从初始代码示例中获得了什么？我们正在获取期货，然后立即等待它们？

的`.await`点充当标记。在这里，代码将等待`Future`产生其价值。未来将如何完成？不用管了标记允许组件（通常称为“运行时”）负责*执行中*这段代码负责计算完成时必须执行的所有其他操作。当您在后台执行操作时，将回到这一点。这就是为什么这种编程风格也被称为*事件编程*。我们在等*发生的事情*（例如要打开的文件），然后做出反应（通过开始读取）。

当同时执行两个或多个这些函数时，我们的运行时系统便可以通过处理来填充等待时间*所有其他事件*目前正在进行中。

## Conclusion

从价值出发，我们寻找表达*努力争取稍后可用的价值*。从那里，我们讨论了轮询的概念。

一种`Future`是任何不代表值的数据类型，但具有*在将来的某个时刻产生价值*。根据用例，这种实现的方式千差万别，但界面很简单。

接下来，我们将向您介绍`tasks`，我们将实际使用*跑*期货。

[^1]：保证没有人在写时，由两方阅读始终是安全的。

[futures]: https://rust-lang.github.io/async-book/02_execution/02_future.html
