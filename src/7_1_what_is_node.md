# 什么是 Node

我们必须先简要说明 Node 是什么。

Node 是一个 JS 运行时，它让 JS 在你的桌面（或服务器）上运行。
JS 最初被设计为浏览器的脚本语言，这意味着它依赖于浏览器来解释它并为其提供运行时。

这也意味着桌面上的 JS 需要被解释（或编译）并提供运行时才能做任何有意义的事情。
在桌面上，[V8 JS 引擎] 编译 JS，而 [Node] 提供运行时。

从语言设计的角度来看，JS 有一个优势：一切都被设计为异步处理。
正如你现在所知，如果我们想充分利用我们的硬件，这非常重要，
尤其是当你需要处理大量 I/O 操作时。

其中一个场景是 Web 服务器。
无论是从文件系统读取数据还是通过网卡进行通信，Web 服务器都会处理大量 I/O 任务。

[V8 JS 引擎]: https://en.wikipedia.org/wiki/V8_JS_engine
[Node]: https://en.wikipedia.org/wiki/Node.js

## 为什么是 Node 

- 在为浏览器进行 Web 开发时，JS 是不可避免的。
  在服务器上使用 JS 让程序员使用相同的语言进行前端和后端开发。 
- 后端和前端之间存在代码重用 (reuse) 的可能。 
- Node 的设计让它成为非常高性能的 web 服务器 。
- 当你只处理 JS 时，使用 JSon 和 web APIs 非常容易。

## 有用的事实

首先揭开一些神奇的事，好让使我们在开始编码时更容易理解。

### JS 事件循环

JS 是一种脚本语言，它自己不能做很多事情。它没有事件循环。
现在在 Web 浏览器中，浏览器提供了一个运行时，其中包括一个事件循环。
而在服务器上，Node 提供了这个功能。

你可能会说，如果没有某种事件循环，JS 作为一种语言将难以运行
（由于它基于回调的模型），但这不是重点。

### Node 是多线程的

与我多次看到的说法相反，Node 使用线程池，因此它是多线程的。
但是，“运行”你的代码的 Node 部分确实在单个线程上运行。
当我们说“不要阻塞事件循环”时，我们指的是这个线程，因为它会阻止 Node 在其他任务上取得进展。

我们会确切地看到为什么阻塞这个线程是一个问题，以及如何处理它。

### V8 JS 引擎

这才是我们需要关注的地方。 V8 引擎是一个 JS JIT 编译器。
这意味着当你编写 `for` 循环时，V8 引擎会将其转换为在 CPU 上运行的指令。
存在很多 JS 引擎，但 Node 最初是在 V8 引擎之上实现的。

V8 引擎本身对我们来说不是很有用；它只是解释 JS。
它不能做 I/O 方面的事情，也不能准备运行时或类似的东西。
仅使用 V8 编写 JS 将是一种非常有限的体验。

> 由于我们编写了 Rust（即使我们让它看起来有点像 JS），本书不会涵盖解释 JS 的部分。
> 我们现在的主要关注点是：Node 如何工作以及它如何处理并发。 

### Node 事件循环

Node 内部将其实际的工作分为两类：

#### 受 I/O 限制的任务

`libuv` 实现的跨平台 epoll/kqueue/IOCP 事件队列 处理等待外部事件发生的任务，
在我们的例子中是 `minimio`。

#### 受 CPU 限制的任务

CPU 密集型为主的任务由线程池处理。此线程池的默认大小为 4 个线程，但可以由 Node 运行时配置。

跨平台事件队列无法处理的 I/O 任务也在这里被处理，我们在示例中使用的文件读取就是这种情况。

Node 的大多数 C++ 扩展都使用这个线程池来执行它们的工作，
这也是它们用于计算密集型任务的众多原因之一。

## 更多信息

如果你确实想了解有 Node 事件循环的更多信息，你可以参考以下这个简短的 `libv` 文档页面，
而且提供给你两个我认为在这个主题上很棒（且正确）的演讲：

1. [Libuv 设计一览](http://docs.libuv.org/en/v1.x/design.html#design-overview)

2. 由 [@piscisaureus](https://github.com/piscisaureus) 主讲的演说，是一个 15 分钟的精彩概述。
   我特别推荐这个，因为它简短扼要：
   [Morning Keynote- Everything You Need to Know About Node.js Event Loop - Bert Belder, IBM]
3. 第二个稍长（30 分钟），由 [bryan hughes](https://github.com/nebrius) 主讲，也很棒：
   [The Node.js Event Loop: Not So Single Threaded]

[Morning Keynote- Everything You Need to Know About Node.js Event Loop - Bert Belder, IBM]:
https://www.youtube.com/watch?v=PNa9OMajw9w&t=309s

<!-- <iframe width="560" height="315" src="https://www.youtube.com/embed/PNa9OMajw9w" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe> -->

[The Node.js Event Loop: Not So Single Threaded]:
https://www.youtube.com/watch?v=zphcsoSJMvM

<!-- <iframe width="560" height="315" src="https://www.youtube.com/embed/zphcsoSJMvM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe> -->

现在，放松，喝杯茶，坐下来，我们一起完成所有事情。
