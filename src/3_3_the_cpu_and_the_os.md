# CPU 与 OS 合作吗？

当我初次认为已经理解程序如何工作时，你若问我这个问题，我很可能回答，不，它们不会合作。
我们在 CPU 上运行程序，而且只要知道怎么做，就可以做任何想做的事情。
现在的话，首先我还没有想清楚，除非你知道从底层开始的工作机制，
否则不是那么容易确切地知道这个问题的答案。

让我开始意识到我回答错误的，是类似下面的代码：

```rust
#![feature(llvm_asm)]
fn main() {
    let t = 100;
    let t_ptr: *const usize = &t;
    let x = dereference(t_ptr);

    println!("{}", x);
}

fn dereference(ptr: *const usize) -> usize {
    let res: usize;
    unsafe {
        llvm_asm!("mov ($1), $0":"=r"(res): "r"(ptr))
        };
    res
}
```

> 这里使用汇编指令编写一个 dereference 函数。我们知道这里不可能牵涉到 OS 。

这段代码会如期输出 `100` 。但现在，创建一个地址为 `99999999999999` 的指针，
这不是一个有效的指针，单后看看传进同一个函数会发生什么。

```rust
#![feature(llvm_asm)]
fn main() {
    let t = 99999999999999 as *const usize;
    let x = dereference(t);

    println!("{}", x);
}
# fn dereference(ptr: *const usize) -> usize {
#     let res: usize;
#     unsafe {
#     llvm_asm!("mov ($1), $0":"=r"(res): "r"(ptr));
#     }
#
#     res
# }
```

现在会得到一个分段错误 (segmentation fault) 。这不奇怪，但为何 CPU 知道不允许解引用这个内存呢？

- CPU 询问了 OS 这个进程在每次解引用时都有权访问这个内存地址吗？ 
- 这个过程会不会很慢？
- CPU 怎么知道它上面有个 OS 在运行呢？
- 所有的 CPUs 都知道分段错误是什么吗？
- 为什么我们得到一个错误信息，而不是单纯的崩溃 (crash) ？

# 深入研究

事实证明，操作系统和CPU之间有大量的合作，但可能不是你天真地认为的那样。

许多现代 CPUs 提供一些 OS 使用的基础设施。这些基础设施提供安全和稳定。
实际上，大多数高级 CPUs 提供 OS （比如 Linux、 BSD、 Windows）实际使用的更多选项。

这两点我想重点强调：
1. CPU 如何防止不应该的内存访问
2. CPU 如何处理 I/O 那样的异步事件

本章会谈论一个问题，第二个问题放在下一章。

> 如果你想了解更多细节，我强烈建议你阅读 
> [Philipp Oppermann： Writing an OS in Rust 系列](https://os.phil-opp.com/) 。
> 它写得很好，会回答上述所有问题，以及更多问题。

# CPU 如何防止不应该的内存访问

正如上面提到的，现代 CPUs 已经定义了一些基础概念。比如：
- 虚拟内存 (virtual memory)
- 页表 (page table)
- 页面错误 (page fault)
- 异常 (exceptions)
- 优先级 ([privilege level])

[Privilege level]: https://en.wikipedia.org/wiki/Protection_ring

具体如歌工作取决于具体的 CPU ，所以这里把这些视为一般意义上的术语。

大多数现代 CPUs 有一个内存管理单元 (MMU, Memory Management Unit) 。
MMU 的任务是将程序中使用的虚拟地址转换为物理地址。

当 OS 开启一个进程（比如我们编写的程序），这会给进程建立一个页表，
而且会确保 CPU 上的某个特殊寄存器指向这个页表。

上面的代码中，我们解引用 `t_ptr` 时，某个时刻这个地址已经传入 MMU ，
MMU 在页表查找地址、把它转换成内存里的物理地址，而这个地址能获取到数据。

在第一个例子中，它指向栈上拥有 `100` 这个值的内存地址。

在第二个例子中，传入 `99999999999999` ，要求获取这个地址（解引用）的数据时，
在页表中查找转换后的地址，结果没能找到。然后 CPU 把这视为一个 “页面错误” 。

系统启动时，OS 给 CPU 提供一个中断描述符表 (Interrupt Descriptor Table) 。
这个表具有预定义的格式，OS 据此对 CPU 可能遇到的预定义的异常提供处理程序。

由于 OS 提供处理页面错误的函数指针，当试图解引用 `99999999999999` 地址时，
CPU 跳转到那个函数，从而把控制权交给 OS 。

然后 OS 很好地打印了信息，让我们知道遇到了一个叫 `segmentation fault` 的异常信息。
所以在代码运行的不同 OS 上，这个信息会不一样。

# 我们不能改变 CPU 里的页表吗？

这是优先级 (Privilege Level) 发挥作用的时候了。大多数现代 OS 有两个环形等级 (Ring Levels)：
内核空间 Ring 0 和用户空间 Ring 3 。

<!-- ![Privilege rings](./images/priv_rings.png) -->
 <img src="./images/priv_rings.png" width = "40%" height = "40%" alt="Privilege rings" align=center />

 大多数 CPUs 概念上有多于大多数 OS 使用的环形。
 这有历史原因的，也正是这个原因使用 `Ring 0` 和 `Ring 3` 而不是 1 和 2 。

现在，页表中的每个条目都有关于它的额外信息，
其中包括关于它所属的环形的信息。此信息是在 OS 启动时设置的。


在 `Ring 0` 中执行的代码几乎可以不受限制地访问外部设备和内存，
并且可以自由更改硬件级别所提供安全性的寄存器。

你在 `Ring 3` 中编写的代码通常对 I/O 和某些 CPU寄 存器（以及指令）的访问极为有限。
尝试 CPU 已经阻止从 `Ring 3` 级别发出指令或设置寄存器来更改页表。
然后，CPU 将此视为异常，并跳转到 OS 提供的异常处理程序。

这也是你除了通过系统调用与 OS 合作并处理 I/O 任务之外，别无选择的原因。
如果不是这样，系统就会很不安全。
