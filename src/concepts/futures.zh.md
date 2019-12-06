# Futures

关于 Rust 的值得注意的一点是[_无畏并发_](https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html)。这就是应该授权您在不放弃安全性的情况下，执行并发任务的思考。同样，Rust 作为一种低级语言，它的无畏并发中，_没有选择具体的实现策略_。这意味着，我们*必须*对策略进行抽象，以允许*后来*选择(给用户发挥的空间)，如果我们想以某种方式，在不同策略的用户之间共享代码，这又怎么搞。

Future 从*计算(computation)*抽象出来。描述了“什么(what)”，与“哪里(where)”和“何时(when)”无关。为此，他们的目标是将代码分解为可组合的小行动，然后由我们系统的一部分执行。让我们来趟旅程，计算下东西，找找可以抽象的地方，理解它的含义。

## Send and Sync

幸运的是，并发 Rust 已经有两个众所周知的有效概念，它们抽象了程序并发部分之间的共享：`Send`和`Sync`。值得注意的是，`Send`和`Sync` trait 抽象的*策略*，包括并发工作，组成整齐，并且不实际实现。

快速摘要：

- `Send`抽象了，在一个计算中，_将数据传递_，到另一个并发计算（让我们称其为接收方-receiver），而在发送方则失去对它的访问。在许多编程语言中，通常都采用这种策略，但是缺少语言方面的支持，因此希望您自己遵循“丢失访问”行为。这是漏洞的常规来源：发件人保留发送内容的控制权，甚至在发送后也去使用它们。Rust 通过将这种行为定为'已知'，来减轻此问题。类型可以为是/非`Send`（通过实现适当的标记 trait），能/不允许将它们发送出去，而所有权和借用规则会对后续访问进行控制(阻止与否)。

- `Sync`关于*共享数据*，处在程序的两个并发部分之间。这是另一种常见的模式：由于写入内存位置，或在另一方正在写入时，就进行读取，在本质上是不安全的，因此需要通过同步(sync)来缓和这种访问。[^1]这里有许多常见方式，针对双方共识，即不要同时使用内存中的同一部分，例如互斥锁和自旋锁（mutexes and spinlocks）。同样，Rust 为您提供了省心的（安全！）选项。Rust 使您能够表达某些*需要*同步的东西，但不用具体说明*怎么样*去做的。

注意我们避免出现类似 _"线程(thread)"_，而是选择了“计算(computation)”。`Send`和`Sync`的威力是，减轻了您对“*什么*是共享”的学习负担。在实现时，您只需要知道哪种共享方法，是适用于当前类型的。这保持了本地化的合理，不受该类型用户，以后会使用的任何实现的影响。

`Send`和`Sync`可以以有趣的方式进行合成，但这超出了本文的范围。您可以在[Rust Book][rust-book-sync]找到示例。

[rust-book-sync]: https://doc.rust-lang.org/stable/book/ch16-04-extensible-concurrency-sync-and-send.html

总结：Rust 使我们能够安全地抽象并发程序的重要属性，及其数据共享。它以非常轻巧的方式进行。语言本身只知道两个标记`Send`和`Sync`，并尽可能地通过派生它们本身来帮助我们。剩下的就是图书馆的问题。

## An easy view of computation

虽然 computation 作为一个主题，可以写出一整本[book](https://computationbook.com/)出来，但一个非常简化的视图对我们来说，就足够了：一系列可组合的操作，这些操作可以基于决策进行分支，相继执行，并产生（yield）结果或产生（yield）错误

## Deferring computation

正如刚才提到的，`Send`和`Sync`是关于数据的。但是程序不仅涉及数据，还涉及对数据*进行计算*。这就是[`Futures`][futures]所要做的。在下一章中，我们将仔细研究它的工作方式。让我们用英语，看看 Future 允许我们表达什么。Future 的计划，是（将正常的计算流程）从：

- 做 X
- 如果 X 成功，则执行 Y

转向(这个流程)：

- 开始做 X
- 一旦 X 成功，就开始做 Y

```
- Do X
- If X succeeded, do Y

towards:

- Start doing X
- Once X succeeds, start doing Y
```

还记得前言中，有关“延迟计算”的话题吗？这就是所有的要点。不再*立即*告诉计算机执行什么，以及下一个决定，我们改成了，告诉它要开始做什么，以及如何对潜在事件做出反应 ... well 也就是 ... `Future`。

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

您可以随时调用它，因此可以完全控制何时调用它。但这也是问题所在：调用它的那一刻，控制权就转移到被调用的函数，直到它最终返回一个值。请注意，此返回值来自过去(得出值的那一刻)。过去有一个缺点：已经做出了所有决定。它有一个优点：结果是可见。我们可以解开程序过去的计算结果，然后决定如何处理它。

但是，我们想对*computation*进行抽象，并让其他人选择运行方式。从根本上说，这与一直查看先前的计算结果不兼容。所以，让我们找到一个类型来*描述*，一个不执行的 computation。让我们再次看一下函数：

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

说到时机，我们只能在调用函数*之前*或函数返回*之后*采取行动。而这是不可取的，因为它拿走了，*处于*运行中，帮我们做某事的能力。当与并行代码配合时，会拿走我们在首次运行时，启动并行任务的能力（因为我们放弃了控制权）。

这时，是祭出[threads](https://en.wikipedia.org/wiki/Thread_)的时候了。但是 threads 是一个非常特定的并发原语，且我们说过我们正在寻找一个抽象。

我们正在寻找的东西，要能在 future 中，代表正在进行的工作，和朝向一个 result。每当我们在 Rust 语言中，说“something”时，基本可表示为一种 trait。那就让我们从一个不完整的`Future` trait 定义开始吧：

```rust,edition2018
# use std::{pin::Pin, task::{Context, Poll}};
#
trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}
```

仔细观察，我们看到以下内容：

- 它是泛型的`Output`。
- 它提供了一个称为`poll`的函数，这使我们可以检查/推进当前 computation 的状态。
- （目前忽视`Pin`和`Context`，深入理解该 trait 之前，暂时不需要这些。）

每次调用`poll()`可能导致以下两种情况之一：

1.  computation 完成，`poll`将返回[`Poll::Ready`](https://doc.rust-lang.org/std/task/enum.Poll.html#variant.Ready)
2.  computation 尚未完成执行，它将返回[`Poll::Pending`](https://doc.rust-lang.org/std/task/enum.Poll.html#variant.Pending)

这使我们可以从外部检查，`Future`是否仍有未完成的工作，或者最终是完成了，可以给我们带来值。最简单（但效率不高）的方法是，在一个循环中，不断轮询(poll) Future。这有优化的可能性，好的 runtime 会帮你搞定。请注意，情况 1 发生后，再次`poll`可能会导致行为混乱。参考[futures-docs](https://doc.rust-lang.org/std/future/trait.Future.html)中的有关详细信息。

## Async

而`Future` trait 在 Rust 中已经存在了一段时间，构建和描述它们都很不方便。为此，Rust 现在具有特殊的语法：`async`。上面的示例，使用`async-std`，如下所示：

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

差别很小，对吧？我们要做的只是将函数标记上`async`，并插入 2 个特殊命令：`.await`。

这个`async`函数设置了一个延迟计算。调用此函数时，将产生一个`Future<Output = io::Result<String>>`，而不是立即返回`io::Result<String>`。（或者，更准确地说，为您生成一个类型，而这个类型实现了`Future<Output = io::Result<String>>`）

## What does `.await` do?

`.await`后缀，它的作用与其名字完全一样：在您使用它的那一刻，代码将一直等到所请求的操作（例如，打开文件或读取其中的所有数据）完成。`.await?`并不特殊，只是应用`?`操作符，整出了`.await`这个结果。那么，我们从初始代码示例中，又获得了什么？我们正在获取 Future，然后立即等待它们？

把`.await`行为称作一个 marker。在这里，代码会等待`Future`产生其价值。那 Future 又是如何驶向完成的呢？这你不用管了，marker 会允许组件（通常为“runtime”）负责*executing*这段代码，且当这个 computation 部分完成时，必须执行的所有其他操作。当后台执行操作的完成后，会回到这 marker 的代码位置。这就是为什么，这种编程风格也被称为*事件编程(evented programming)*。我们在等*事情的发生*（例如要打开的文件），然后做出反应（如开始读取）。

当同时执行两个或多个这些函数时，我们的运行时系统便可以通过处理目前正在进行的*所有其他事件*，来填充等待时间。

## Conclusion

来到 value（值） 的方面，我们寻找能够表达*一直工作，就能获得稍后可用的值*。从那里，我们会讨论了轮询(poll)的概念。

`Future`作为一种任意数据类型，且并不代表一个值，但具有*在将来的某个时刻产生值*的能力。根据用例，这种实现的方式千差万别，但接口简单。

接下来，我们将向您介绍`tasks`，实际使用*run* Future。

[^1]: Two parties reading while it is guaranteed that no one is writing is always safe.

[futures]: https://rust-lang.github.io/async-book/02_execution/02_future.html
