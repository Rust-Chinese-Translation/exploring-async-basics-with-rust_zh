# Epoll, Kqueue 和 IOCP

有一些著名的库分别使用 Epoll、Kqueue 和 IOCP 为 Linux、Mac 和 Windows 实现了跨平台事件队列。

Node 的运行时有一部分基于 [libuv](https://github.com/libuv/libuv) ，这是一个跨平台的异步 I/O 库。
`libuv` 不仅在 Node 中使用，还构成了
[Julia](https://julialang.org/) 和 [Pyuv](https://github.com/saghul/pyuv) 
创建跨平台的基础事件队列；大多数语言都有绑定 `libuv` 。

在 Rust 中，我们有 [mio: Metal IO](https://github.com/tokio-rs/mio)。
`Mio` 为 [tokio](https://github.com/tokio-rs/tokio) 中使用的操作系统事件队列提供支持，
这是一个提供 I/O 、网络、调度等功能的运行时。
`Mio` 对于 `tokio` 正如 `libuv` 对于 `Node` 那样重要。

`tokio` 为许多 Web 框架提供支持，其中包括众所周知的、非常高效的
[Actix Web](https://github.com/actix/actix-web) 。

由于我们想了解一切是如何工作的，我决定写一个极其简化的事件队列版本。
我称它为 `minimio` ，因为它极大地受到了 `mio` 的启发。

> 我在 [Epoll, Kqueue and IOCP explained] 一书中详细描述了它们是如何工作的，而且在那本书中，
> 我还写了一个将在本书中用作跨平台的事件循环。如果你很好奇的话，可以访问
> [github repo](https://github.com/cfsamson/examples-minimio) 中的代码。

[Epoll, Kqueue and IOCP explained]: https://cfsamsonbooks.gitbook.io/epoll-kqueue-IOCP-explained

尽管如此，本书还是会简要介绍它们，以便你了解基础知识。

# 为什么使用 OS 支持的事件队列？

如果您还记得之前的章节，您就会知道我们需要与操作系统密切合作，以使 I/O 操作尽可能高效。
像 Linux、Macos 和 Windows 这样的操作系统提供了几种执行 I/O 的方式，包括阻塞和非阻塞。

所以阻塞操作对我们程序员来说是最不灵活的，因为我们将控制权交给操作系统，它挂起我们的线程。
最大的优点是一旦我们等待的事件准备好，我们的线程就会被唤醒。

非阻塞方法更灵活，但需要有一种方法来告诉我们任务是否准备就绪。
这通常是通过返回某种数据来完成的，这些数据表明它是“准备好” `Ready` 还是“未准备好” `NotReady` 。
一个缺点是我们需要定期检查此状态才能判断状态是否已更改。

通过 Epoll/Kqueue/IOCP 进行事件队列发挥了非阻塞方法的灵活性，且没有上述缺点。

> 这里不会介绍 `poll` 和 `select` 之类的方法，但如果你想了解一下这些方法以及它们与 `epoll` 的区别，
> 你可以看看这篇 [文章：epoll-vs-kqueue] 。

[文章：epoll-vs-kqueue]:http://web.archive.org/web/20190112082733/https://people.eecs.berkeley.edu/~sangjin/2012/12/21/epoll-vs-kqueue.html

# 基于就绪状态的事件队列

Epoll 和 Kqueue 被称为基于就绪状态 (Ready-based) 的事件队列，
这意味着它们会让您知道何时准备执行操作。
一个例子是准备好被读取的 socket 。

**当我们想使用 epoll/kqueue 从 socket 读取数据时，基本上会发生如下情况：**

1. 我们通过调用系统调用 `epoll_create` 或 `kqueue` 来创建一个事件队列。 
2. 我们向操作系统询问代表网络 socket 的文件描述符。 
3. 通过另一个系统调用，我们在这个 socket 上注册了对感兴趣的 `Read` 事件。
   重要的是，我们还通知操作系统，当事件在我们在 (1) 中创建的事件队列中准备就绪时，我们将收到通知。 
4. 接下来，我们调用 `epoll_wait` 或 `kevent` 来等待一个事件。这将阻塞（挂起）事件被调用的线程。 
5. 当事件准备好时，线程被解除阻塞（恢复），我们从“等待”调用中返回事件发生的数据。 
6. 在 (2) 中创建的 socket 上调用 `read`。

# 基于完成状态的事件队列

IOCP (Input/Output Completion Port) 代表 I/O 完成端口，
是一个基于完成状态 (completion-based) 的事件队列。
这种类型的队列会在事件完成时通知您。一个例子是数据被读入缓冲区。

以下是此类事件队列中发生的情况的基本细分：

1. 我们通过调用 `CreateIoCompletionPort` 这个系统调用来创建一个事件队列。 
2. 我们创建一个缓冲区，并要求操作系统给我们一个 socket 的句柄 (handle) 。 
3. 我们使用另一个系统调用在这个 socket 上注册感兴趣的 `read` 事件，
   但这次我们也传入了我们在 (2) 中创建的缓冲区，数据将被读取到该缓冲区。 
4. 接下来，我们调用 `GetQueuedCompletionStatusEx` ，它将阻塞线程直到事件完成。 
5. 我们的线程被解除阻塞，缓冲区现在存满了我们感兴趣的数据。

# Epoll

`epoll` 是 Linux 实现事件队列的方式。在功能方面，它与 `kqueue` 有很多共同点。
在 Linux 上使用 `epoll` 比使用 `select` 或 `poll` 等其他类似方法的优势在于
`epoll` 旨在非常有效地处理大量事件。

## Kqueue

`kqueue` 是 macOS 实现事件队列的方式，起源于 BSD ，存在于 FreeBSD、OpenBSD 等操作系统中。
从高层次的功能来看，它在概念上与 `epoll` 类似，但在实际使用上有所不同。

有些人认为它使用起来更复杂，更抽象和“通用”。

## IOCP

`IOCP` （Input/Output Completion Port，输入输出完成端口）是 Windows 处理此类事件队列的方式。

“完成端口”会在事件“完成”时通知您。现在这听起来可能是一个微小的区别，但事实并非如此。
当您想编写一个库时，这一点尤其明显，因为对两者进行抽象意味着您必须将
`IOCP` 基于就绪状态进行建模 (readiness-based) 或将 
`epoll/kqueue` 基于完成状态的建模 (completion-based) 。

将缓冲区借给操作系统也带来了一些挑战，因为在等待操作返回时，该缓冲区保持不变这一点非常重要。

> 我对此进行了研究，我的经验是：相比其他方式，
> 让“基于就绪状态”的模型表现得像“基于完成状态”的模型更容易。
> 这意味着您应该首先让 IOCP 工作，然后将 `epoll` 或 `kqueue` 加入该设计中。
