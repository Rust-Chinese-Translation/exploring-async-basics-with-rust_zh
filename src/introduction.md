# 简介

## 前言

**别阻塞事件循环！别忙轮询！提高吞吐量！使用异步 I/O ！并发与并行不一样！**

你很可能在这之前听过或者看过这些言论，而且不止一次。
有时你认为已经懂了一切，却之后依然有些糊涂。
当你想在基础层面理解它们时，这种感觉尤为突出。

我也是！

因此，我花了几百个小时弄清楚，然后把研究的结果写在这本书里。
现在我邀请你与我一起，在书中揭开异步编程的奥秘。

本书的目标是带你探索并发编程 (concurrent programming) **为什么需要** 以及 **如何实现** 。

首先，你会建立起良好的基础知识；
然后运用这些知识构建一个受 Node 启发的运行时 (runtime) ，
从而来探究 Node.js 的工作原理。

> 本书是开源的：
> - 英文：
>   [渲染版](https://cfsamson.github.io/book-exploring-async-basics) |
>   [repo](https://github.com/cfsamson/book-exploring-async-basics)
> - 中文：
>   [渲染版](https://zjp-cn.github.io/exploring-async-basics-with-rust_zh) |
>   [repo](https://github.com/zjp-CN/exploring-async-basics-with-rust_zh)
> 
> 本书的示例代码放在了这个仓库：
> [examples-node-eventloop](https://github.com/cfsamson/examples-node-eventloop)
> 
> 书籍和代码遵循 MIT 许可。所以尽情使用吧。

然而，我得提醒你一句，我们需要从正式定义一个 “任务” (task) 出发，
到固件之类难懂的话题，这个过程如同经历从哲学的高峰一路直到深海那般艰难。
我认为那些深层次的话题包括底层 OS 系统调用和 Windows 的底层结构这样的奇怪事物，
我还得确认一下这些内容。

> 本书将涵盖三大操作系统 Linux、 macOS 和 Windows ，
> 也会涉及如何在 64 位系统上运行异步的细节问题。

## 本书的受众

起初，我想探索 Rust Futures 的基本原理和内部工作机制。
通过阅读 RFCs 、积极查找资料和参与讨论 ，我意识到了，
要真正理解为什么是 Futures 、 Futures 如何工作这两个问题，
我得很好地理解一般的异步代码是如何工作的，以及处理异步的不同策略。

**如果你符合下面的情况，那么本书适合你阅读：**

- 想深入理解 并发是什么、处理并发的策略 
- 对如何 在三大 OS 上进行系统调用、不同抽象层次实现系统调用 感到好奇
- 想更深入了解 OS 、 CPU 和硬件如何处理并发
- 想学习 Epoll、 Kqueue 和 IOCP 的基础知识
- 认为 运用所学知识编写一个小型 (toy) Node.js 运行时 这件事非常酷
- 想知道更多关于 Node 的事件循环 (event loop) 是什么、为啥网络上大多关于它的解说图非常令人误解
- 已经熟悉一些 Rust ，但想了解更多有关 Rust 的知识

你对上述某些问题的回答是不是 yes ？
是的话，就加入这趟探险，因为我们会对上面所有话题都有更好的理解。

> 我们只使用 Rust 的标准库。原因是我们真正想弄清异步是怎么工作的；
> 想弄清 Rust 的标准库如何在异步上提供抽象、取得平衡的同时，
> 它保持了精简，从而让我们容易窥探真正发生的事实。

## 关于本书代码

本书使用 `mdbook` ，虽然它有一个优点，可以直接在书中运行代码，
但是涉及 I/O 和跨平台系统调用时，Rust playground 并不合适。

我强烈建议在你的本机上创建一个项目，跟着本书内容复制粘贴代码，在你的电脑上运行它们。

你也可以从 [examples-node-eventloop](https://github.com/cfsamson/examples-node-eventloop)
这个 repo clone 或下载示例代码。

## 预备知识

你不必跟着本书来成为一个 Rust 程序员。（译者注：意思是这本书不是一本 Rust 的入门书）

本书会有很多章节来探索概念，里面的代码示例很小，易于理解，
但是越往后代码越多，你得事先学习基础的语法才能理解大多数代码。
关于基础语法，[Rust Book](https://doc.rust-lang.org/book) 是你入门的最佳途径。

我强烈建议阅读本书之前请阅读我写的另一本书： 
[用 200 行 Rust 代码解释绿色线程](https://app.gitbook.com/@cfsamson/s/green-threads-explained-in-200-lines-of-rust/)
。因为它涉及了一些 Rust 基础、栈、线程和内联汇编 (inline assembly) 等知识，
所以本书不再重复这些概念。当然，阅读它并非必要。

> 你可以在这找到 [安装 Rust] 的一切说明。

[安装 Rust]: https://www.rust-lang.org/tools/install

## 声明

1. 本书会实现一个**小型**的 Node.js 事件循环成品，这个成品不怎么好，但是使用了类似于事件循环的概念。
2. 本书**不会**把重点放在代码质量和安全性上，而是把重点放在理解代码背后的概念与想法上。
   本书不得不走捷径来让重点简明扼要。
3. 我会尽量指出这些捷径及捷径的危险之处，也会尽量指出哪些地方显然可以改进或者走更短的捷径。
 
> 虽然本书涉及一些复杂的话题，但是出于篇幅原因，不得不很大程度上将它们简化。
> 你或许能花费一些工夫成为本书所涉及领域的专家，
> 但目前请允许我没能把这些复杂的话题精确而详尽地阐述 —— 它们本应值得如斯。

## 贡献

感谢许多人对本书做出贡献。

没有什么能比 分享难懂的知识、让下一个好奇的人更容易理解知识 更让我更感兴趣的了！

欢迎参与贡献，即使只是提交拼写、格式或者标点错误的修正 PR 。
你可以到以下 repo 参与贡献。

1. 可到 [原作 repo](https://github.com/cfsamson/book-exploring-async-basics) 
提交反馈、内容更正方面的 PR 。
2. 到 [示例代码 repo](https://github.com/cfsamson/examples-node-eventloop) 
提交与改进代码相关的 PR 。
3. 到 [翻译 repo](https://github.com/zjp-CN/exploring-async-basics-with-rust_zh)
提交翻译相关的 PR 。

这些贡献会让下一位读者更好地理解这本书。

## 由来与姊妹篇

我为什么写下这本书，以及姊妹篇？

起初，我只是希望写一篇关于 Rust Futures 3.0 的文章，
而现在的结果是 3 本关于并发的书， 1 本单独关于 Rust Futures 的书。

这个过程让我意识到了为何会对一段小时候被威胁的记忆十分模糊：
我不停地对所有事情提 "why?" ，车停了，而我被迫下车。

总之，在阅读 RFCs 和讨论 Rust 异步的时候，我写下了以下一些书：

- [Green threads explained in 200 lines of Rust](https://app.gitbook.com/@cfsamson/s/green-threads-explained-in-200-lines-of-rust/)

- [The Node Experiment - Exploring Async Basics with Rust](https://cfsamson.github.io/book-exploring-async-basics/) （本书）

- [Exploring Epoll, Kqueue and IOCP with Rust](https://github.com/cfsamson/book-exploring-epoll-kqueue-iocp) （本书的姊妹篇）

- Exploring Rust's Futures[^tbc] （未完成）：从不同的视角看待 Rust Futures

[^tbc]: 译者注：这本书似乎指的是《Futures Explained in 200 Lines of Rust》。已有
[英文版 (by cfsamson)](https://cfsamson.github.io/books-futures-explained) |
[中文版 (by nkbai)](https://github.com/nkbai/200-Rust-Futures) 。
