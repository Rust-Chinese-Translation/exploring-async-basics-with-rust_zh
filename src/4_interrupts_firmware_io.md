# 中断 | 固件 | I/O

我们就要结束本书通用 CS 的话题了。

这一章会把知识联系起来，看看整个计算机是如何作为一个系统来处理 I/O 和并发的。

## 总览

以下是从网卡读取数据的流程图：

<a href="./images/AsyncBasicsSimplified.png" target="_blank">
![Simplified Overview](./images/AsyncBasicsSimplified.png)
</a>
<p style="font-style: italic; text-align: center;">点击图片，将在新窗口打开</p>

> **声明：**
> 这原本是十分复杂的操作，但这里把事情简化了。我们跳过几步，将重点放在大多数人感兴趣的地方上。

## 1. 解读我们编写的代码

通过向 OS 发出一个系统调用来注册 (register) 一个 socket 。
在 macOS/Linux 上会得到 `file descriptor` ，而在 Windows 上得到 `socket` 。

然后是我们感兴趣的这个 socket 上的 `read` 事件。

## 2. 向操作系统注册事件

有三种不同的处理方式：

<ol type="A">
<li>

告诉 OS 我们对 `Read` 事件感兴趣，但我们希望等待线程的控制权让给 OS 来让 `Read` 事件发生。
然后 OS 通过存储寄存器状态挂起线程，再切换到其他线程。

**从我们的角度看，在有数据可读之前，这会阻塞线程。**
</li>
<li>

告诉 OS 我们对 `Read` 事件感兴趣，但我们只希望获取任务的句柄，通过轮询 `poll` 来检查事件是否准备好。

**OS 不会挂起线程，所以这不会阻塞代码。**
</li>
<li>

告诉 OS 我们可能对许多事件感兴趣，但我们希望订阅一个事件队列。
当轮询这个队列时，在一个或更多事件发生之前，会发生阻塞。

**当我们等待时间发生时，这会阻塞线程。**
</li>
</ol>

> 我的下一本书将是关于替代 C 的，因为它是一个非常有趣的处理 I/O 事件的模型，
> 这对于理解为什么 Rust 的并发抽象是这样建模的很重要。因此，这里不作详细介绍。

## 3. 网卡

> 我们会跳过一些对理解这个话题不太重要的步骤。

在网卡上有一个运行专用固件的微型控制器。我们可以想象这个微型控制器是在一个忙轮询检查是否有数据传入。

> 网卡处理其内部的确切方式很可能与此不同。
> 重点是，网卡上有一个非常简单但专门用于检查是否有传入事件的 CPU 。

一旦固件接收传入的数据，它就会发出硬件中断 (Hardware Interrupt) 。

## 4. 硬件中断
> 这是一个简化版的解释。如果你想深入了解，建议你阅读 Robert Mustacchi 的文章
> [Turtles on the wire: understanding how the OS uses the modern NIC](https://www.joyent.com/blog/virtualizing-nics) 。

现代 CPU 有一组 `Interrupt Request Lines` （中断请求线） 来处理发生自外部设备的事件。
CPU 有一组固定的中断线。

硬件中断 (hardware interrupt) 是一种随时可以发生的电信号。
CPU 立刻**中断**其正常工作流程，通过存储其寄存器的状态并查找中断处理程序来处理中断。
中断处理程序 (interrupt handler) 在中断描述符表 (Interrupt Descriptor Table) 中定义。

## 5. 中断处理程序

中断描述符表 [Interrupt Descriptor Table (IDT)] 是 OS 或驱动程序为可能发生的中断而注册处理程序的表。
每个入口 (entry) 都指向一个特定中断的处理函数。网卡的处理函数通常由网卡的驱动程序 (driver) 注册和处理。

[Interrupt Descriptor Table (IDT)]: https://en.wikipedia.org/wiki/Interrupt_descriptor_table

> IDT 在图中看起来像存储在 CPU 上，而实际并不是。
> IDT 位于主内存 (main memory) 的固定位置上。
> CPU 只在它的某个寄存器上保存指向这个表的指针。

## 6. 写入数据

这一步很不一样，因为取决于 CPU 和网卡上的固件。
如果网卡和 CPU 支持[直接内存访问][Direct Memory Access]（这在所有现代系统上应该都有），
那么网卡会直接把数据写到一组 OS 已经在主内存设置的缓冲区。
在这样的系统上，网卡上的固件可能在数据写入内存时发出中断。
`DMA` 十分高效，因为在数据已经在内存时，才会通知 CPU 。
在旧系统上，CPU 需要投入资源来处理网卡的数据传输。

[Direct Memory Access]: https://en.wikipedia.org/wiki/Direct_memory_access

DMAC （Direct Memory Access Controller 直接存储器存取控制器） 会控制内存访问。
它不是 CPU 的一部分。我们已经够深入了，现在这些对我们来说不太重要，让我们继续往下。

## 7. 驱动程序

驱动程序 (driver) 通常处理 OS 与网卡之间的通信。
缓冲区已满时，网卡发出中断 (interrupt) 。然后 CPU 跳到中断的处理程序 (handler) 。
这种中断类型的中断处理程序是由驱动程序注册的，因此实际上是驱动程序处理这个事件，
然后通知内核数据已经准备好被读取。

## 8. 读取数据

根据我们是选择 A、B 还是 C 选项，OS 将：

<ol type="A">
<li>

唤起线程
</li>

<li>

在下一次轮询 `poll` 时返回 `Ready`
</li>

<li>

唤起线程，并返回一个 `Read` 事件给我们注册的处理程序
</li>
</ol>

## 中断

有两种中断方式：
1. 硬件中断 (Hardware Interrupts)
2. 软件终端 (Software Interrupts)

它们具有不同的性质：

### 硬件中断

硬件中断由 [IRQ (Interrupt Request Line)] 发送电信号创建。这些硬件线路直接向CPU发送信号。

[IRQ (Interrupt Request Line)]: https://en.wikipedia.org/wiki/Interrupt_request_(PC_architecture)#x86_IRQs

### 软件中断

从软件中断，而不是从硬件中断。
在硬件中断的情况下，CPU跳转到中断描述符表并运行指定中断的处理程序。

### 固件

固件 (firmware) 并没有得到我们大多数人的关注；然而，他们是我们所生活的世界的重要组成部分。
它们运行在各种各样的硬件上，并且有各种奇怪和奇特的方法使我们编写程序的计算机工作。

当我想到固件的时候，我会想到《星球大战》里的场景，他们走进酒吧，里面有各种奇怪而晦涩的生物。
我想象固件的世界是这样的，很少有人知道它们在一个特定的系统上做什么或者如何工作。

现在，固件需要微型控制器或类似的东西才能工作。甚至 CPU 也要有固件才能工作。
这意味着在我们的系统中有比我们编程所针对的内核还有更多的小 CPUs 。

为什么这很重要？好吧，你还记得并发是关于效率的吧？
因为我们的系统中已经有很多 CPU 在为我们工作，所以我们的一个关注点是在编写代码时不要重复或复制这些工作。

如果网卡的固件不断地检查新数据是否到达，那么如果我们再重复地让 CPU 不断地检查新数据是否到达，
就将是相当浪费的。如果我们偶尔检查一下，或者在数据到达时及时通知，那就更好了。

