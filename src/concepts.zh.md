# Async concepts using async-std

[Rust Futures][futures]以困难著称。我们认为情况并非如此。在我们看来，它们是现今最简单的并发概念之一，并具有直观的解释。

但是，人们的'困难'总有充分的理由。Future 的三个基本概念似乎始终是造成混乱的根源：延迟计算，异步性和执行策略的独立性（deferred computation, asynchronicity and independence of execution strategy）。

这些概念并不难，但是很多人并不习惯。这种基本的困惑被细节的许多实现所放大。这些实现的大多数说明，甚至只针对高级用户，对于初学者可能很难。而我们，试图提供易于理解的原语，和概念的平易近人一面。

Future 是一个抽象代码运行方式的概念。若是这概念能出声，会说："我什么也不做"。在命令式语言中，这是一个怪异的概念，因为通常，程序中的一件事应——立即——发生在另一件事之后。

那么 Future 如何运行？看下去，你可以自己上手！

Future 什么都不做，不会*执行*代码，哪怕一丢丢。而这部分称为*executor（执行者）*。

> 译者：这里我觉得可以看作为，作者写了个笑话，Future 明明什么都不做，却给个执行者的称呼。当然，这是概念的一部分。

一个*执行者*决定，*什么时候*和*怎么样*执行您的 Future。而`async-std::task`模块为您提供了此类执行值的接口。

不过，让我们从一些动机开始。

[futures]: https://en.wikipedia.org/wiki/Futures_and_promises
