
本章会深入谈到：
* 系统调用 (system calls，或 syscalls) 是什么
* 抽象层次 (abstraction levels)
* 编写底层跨平台代码的难点

# 系统调用入门

与 OS 通信是通过系统调用实现的。
系统调用是 OS 提供的公有 API ，作为 OS 使用者编写的程序可以用系统调用来与 OS 通信。

大多数时候，我们作为编程语言的程序员或某运行时的使用者，这些系统调用已经被抽象出来了。
像 Rust 这样的语言，进行一次系统调用是轻而易举的，下面会谈到。

现在，系统调用是与内核通信的特有例子。
UNIX 类的内核有很多相似之处，UNIX 系统通过 **`libc`** 暴露出来。

Windows 则与基于 UNIX 的系统完全不同，它使用自己的 API，常被称作 WinAPI 。

大多数情况下，有一种方法可以实现同样的目标。
在功能方面，你可能没有注意到很大的差异，但正如我们将在下面看到的，
特别是当我们深入研究 [epoll、kqueue 和 IOCP] 的工作方式时，
它们在实现此功能的方式上可能会有很大的差异。

[epoll、kqueue 和 IOCP]: ./6_epoll_kqueue_iocp.md

# 系统调用的例子

为了更熟悉系统调用，我们会在三大系统上 （BSD(macos)、Linux 和 Windows） 实现一个很基础的系统调用。
也会看看如何以三种抽象层次实现它。

我们等会实现的系统调用是在输出 `stdout` 时用到的，因为这是一个常见的操作，见识这如何工作会很有趣。

## 低层次抽象

为了达到目的，我们需要写一些 [内联汇编][inline assembly] 。
开始写传给 CPU 的指令吧。

> 如果你想知道更详细的内联汇编知识，可以参考我的前一本书 [green threads] 的相关章节。

现在，基于这个层次的抽象，我们要为三种不同平台写不同的代码。

在 Linux 和 macOS 上，我们希望这个系统调用叫做 `write` 。
当你开始一个线程时，基于文件描述符 (file descriptors) 和 stdout 概念的系统操作就已经存在了。

1. Linux 平台的代码会这样写：

（你可以点击右上角的 `▶` 按钮来运行这段代码）

```rust
#![feature(llvm_asm)]
fn main() {
    let message = String::from("Hello world from interrupt!\n");
    syscall(message);
}

#[cfg(target_os = "linux")]
#[inline(never)]
fn syscall(message: String) {
    let msg_ptr = message.as_ptr();
    let len = message.len();

    unsafe {
        llvm_asm!("
        mov     $$1, %rax   # system call 1 is write on Linux
        mov     $$1, %rdi   # file handle 1 is stdout
        mov     $0, %rsi    # address of string to output
        mov     $1, %rdx    # number of bytes
        syscall             # call kernel, syscall interrupt
    "
        :
        : "r"(msg_ptr), "r"(len)
        : "rax", "rdi", "rsi", "rdx"
        )
    }
}
```

在 Linux 上 `write` syscall [^`write` syscall] 开始的代码是 `1` ，
上述代码中把 `$$1` 表示字面值 1 写到了 `rax` 寄存器 (register) 。

> 在内联汇编中 `$$` 使用了 AT&T 语法，表示你一个字面值 (literal value) 。
> 单个 `$` 表示引用一个参数的值，所以 `$0` 表示第一个参数的值，即 `msg_ptr` 的值。
> 我们也需要清空 (clobber) 写入的寄存器，所以得告诉编辑器那正在被修改，别使用寄存器里的任何值。

[^`write` syscall]: 译者注：为了文字简洁，`write` syscall 、`write` 不被翻译，
它们表示这个例子里功能为 `write` 的系统调用。

巧合的是，把 `1` 这个值放入 `rdi` 寄存器意味着我们希望写入 `stdout` 这个文件描述符。
这和 `write` 也有代码 `1` 无关。

接下来，我们分别给 `rsi`、`rdx` 寄存器传入字符串缓冲区 (string buffer) 的地址和长度， 
然后调用 `syscall` 指令。

> `syscall` 指令是一个相当新的指令。
> 早期 `x86` 架构的 32 位系统上，你发出一个软件中断指令 `int 0x80` 来进行一次系统调用。
> 软件中断指令在这里写的代码中是慢的，因此后面增加了名叫 `syscall` 的单独的指令。
> `syscall` 指令使用 [VDSO](http://articles.manugarg.com/systemcallinlinux2_6.html) ，
> 这是一个附属于每个进程内存的内存页 (memory page) ，
> 因此执行这个系统调用无需上下文切换 (context switch) 。

2. 在 macOS 上的代码是这样的：

（由于 Rust playground 运行在 Linux 上，所以这个例子不能直接运行）

```rust, ignored
#![feature(llvm_asm)]
fn main() {
    let message = String::from("Hello world from interrupt!\n");
    syscall(message);
}

#[cfg(target_os = "macos")]
fn syscall(message: String) {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    unsafe {
        llvm_asm!(
            "
        mov     $$0x2000004, %rax   # system call 0x2000004 is write on macos
        mov     $$1, %rdi           # file handle 1 is stdout
        mov     $0, %rsi            # address of string to output
        mov     $1, %rdx            # number of bytes
        syscall                     # call kernel, syscall interrupt
    "
        :
        : "r"(msg_ptr), "r"(len)
        : "rax", "rdi", "rsi", "rdx"
        )
    };
}
```

如你所见，除了 `0x2000004` 代码不再是 `1` ，这里的代码与 Linux 上的例子并没有太大区别。

3. Windows 平台上呢？

这是个好机会来解释一下，为什么上面写的这种代码是不好的做法。

如果你想让代码在未来很长一段时间都能工作，那么你不得不担心 OS 给你提供的保障 (guarantees) 。
据我所知，Linux 和 macOS 会提供保障，比如 macOS 上的 `$$0x2000004` 总是会代表 `write` ，
虽然我不确定这些保障会多稳定。Windows 在这样的底层内部代码方面不提供任何保障。

Windows 已经很多次改变过内部代码，也不提供官方文档。
我们能做的唯一一件事是反向工程表 (reverse engineered tables) ，像[这样]。
这意味着以前代表 `write` 的代码在更新 Windows 系统之后可能变成了 `delete` 。

[inline assembly]: https://doc.rust-lang.org/1.0.0/book/inline-assembly.html
[green threads]: https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/an-example-we-can-build-upon
[这样]: https://j00ru.vexillium.org/syscalls/nt/64/

## 进一步抽象

使用三大 OS 提供的 API 来进行抽象。

这个抽象帮助我们删除一些代码，因为在这个具体的例子中，幸运的是，Linux 和 macOS 上的系统调用是一样的，
所以如果不在 Windows 平台，只需使用 `#[cfg(not(target_os = "windows"))]` 
这个条件编译标志 (conditional compilation flag) ；如果在 Windows 平台，就相反。

### 使用 Linux 和 macOS 提供的 API

```rust
use std::io;

fn main() {
    let sys_message = String::from("Hello world from syscall!\n");
    syscall(sys_message).unwrap();
}

// and: http://man7.org/linux/man-pages/man2/write.2.html
#[cfg(not(target_os = "windows"))]
#[link(name = "c")]
extern "C" {
    fn write(fd: u32, buf: *const u8, count: usize) -> i32;
}

#[cfg(not(target_os = "windows"))]
fn syscall(message: String) -> io::Result<()> {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    let res = unsafe { write(1, msg_ptr, len) };

    if res == -1 {
        return Err(io::Error::last_os_error());
    }
    Ok(())
}

```

我来解释一下上面的代码。

```rust, ignored
#[link(name = "c")]
```

每个 Linux 都有某个版本的 `libc` ，这是与 OS 通信的 C 库。
拥有一个一致 API 的 `libc` 意味着不破坏任何人代码的情况下改变底层实现。
这个 flag 告诉编译器链接到编译目标系统的 C 库。

```rust, ignored
extern "C" {
    fn write(fd: u32, buf: *const u8, count: usize);
}
```

`extern "C"` 或 `extern` （如果未指定默认为 C 语言） 表示按照 C 的调用方式链接到 C 库中的具体函数。
接下来你会看到 Windows 情况下需要更改这里，因为 Windows 采用不同于 UNIX 类的调用方式。

链接到的函数必须有相同的名称，在这个例子中是 `write` 。
参数不必同名，但必须按照正确的顺序，按照链接库的名字给参数命名是好的做法。

`write` 函数接收一个 `file descriptor` ，在这个例子中是 `stdout` 的句柄 (handle) ，
还接收一个 `u8` 数组的指针，以及缓冲区的长度。

```rust, ignored
#[cfg(not(target_os = "windows"))]
fn syscall_libc(message: String) {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    unsafe { write(1, msg_ptr, len) };
}
```

这段代码先获得了字符串背后缓冲区的指针，这是 `*const u8` 类型的指针，对应于 `buf` 参数。
缓冲区的长度 `len` 对应于 `count` 参数。

你或许想问为何知道 `1` 是 `stdout` 的文件句柄，哪里可以找到这个值。

从 Rust 编写系统调用时，你会发现很多这样的情况。
通常常量 (constants) 被定义在 C 的头文件中，那里是无法链接到的，所以需要进行搜索。
`1` 在 UNIX 系统上总是代表 `stdout` 的文件描述符。

> [libc](https://github.com/rust-lang/libc) crate 所做的事情正好是提供封装的 `libc` 函数和这些常量，
> 等会儿会看到另一种写法，不使用这里写的类型代码。

调用 FFI 函数总是 unsafe 的，所以需要使用 `unsafe` 关键字。

### 使用 Windows 的 API

（你需要复制下面的代码到 Windows 系统的电脑上，然后再运行）

```rust, ignored
use std::io;

fn main() {
    let sys_message = String::from("Hello world from syscall!\n");
    syscall(sys_message).unwrap();
}

#[cfg(target_os = "windows")]
#[link(name = "kernel32")]
extern "stdcall" {
    /// https://docs.microsoft.com/en-us/windows/console/getstdhandle
    fn GetStdHandle(nStdHandle: i32) -> i32;
    /// https://docs.microsoft.com/en-us/windows/console/writeconsole
    fn WriteConsoleW(
        hConsoleOutput: i32,
        lpBuffer: *const u16,
        numberOfCharsToWrite: u32,
        lpNumberOfCharsWritten: *mut u32,
        lpReserved: *const std::ffi::c_void,
    ) -> i32;
}

#[cfg(target_os = "windows")]
fn syscall(message: String) -> io::Result<()> {

    // let's convert our utf-8 to a format windows understands
    let msg: Vec<u16> = message.encode_utf16().collect();
    let msg_ptr = msg.as_ptr();
    let len = msg.len() as u32;

    let mut output: u32 = 0;
        let handle = unsafe { GetStdHandle(-11) };
        if handle  == -1 {
            return Err(io::Error::last_os_error())
        }

        let res = unsafe {
            WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null())
            };
        if res  == 0 {
            return Err(io::Error::last_os_error());
        }

    assert_eq!(output as usize, len);
    Ok(())
}
```

上面的代码开始变得复杂起来了，花点时间一行行理解这里做了什么吧。

```rust, ignored
#[cfg(target_os = "windows")]
#[link(name = "kernel32")]
```

第一行只是告诉编辑器只在 Windows 目标平台上编译这段代码。

第二行链接器指令 (linker directive) ，告诉链接器链接到 `kernel32` 这个库。

```rust, ignored
extern "stdcall" {
    /// https://docs.microsoft.com/en-us/windows/console/getstdhandle
    fn GetStdHandle(nStdHandle: i32) -> i32;
    /// https://docs.microsoft.com/en-us/windows/console/writeconsole
    fn WriteConsoleW(
        hConsoleOutput: i32,
        lpBuffer: *const u16,
        numberOfCharsToWrite: u32,
        lpNumberOfCharsWritten: *mut u32,
        lpReserved: *const std::ffi::c_void,
    ) -> i32;
}
```

首先， `extern "stdcall"` 告诉编辑器不使用 `C` 调用方式，而使用 `stdcall` 的 Windows 调用方式。

然后是希望链接到的函数。在 Windows 上，为了让例子工作，这里需要两个函数：
`GetStdHandle` 和 `WriteConsoleW` 。

`GetStdHandle` 返回标准设备（比如 `stdout` ）的引用。

`WriteConsole` 有两种接收方式：`WriteConsoleW` 接收 Unicode 文本， 
`WriteConsoleA` 接收 ANSI 编码的文本。

如果你只写入英语文本，ANSI 编码是不错的选择，但是一旦你写其他语言 ——
你可能需要使用不以 `ANSI` 表示而以 `utf-8` 编码的特殊字符 ——
程序就会 break。

这就是为什么把 `utf-8` 编码的文本转换成 `utf-16` 编码的 Unicode 代码点 (codepoints) ，
`utf-16` 代码点可以表示这些字符，而且被 `WriteConsoleW` 函数使用。

```rust, ignored
#[cfg(target_os = "windows")]
fn syscall(message: String) -> io::Result<()> {

    // let's convert our utf-8 to a format windows understands
    let msg: Vec<u16> = message.encode_utf16().collect();
    let msg_ptr = msg.as_ptr();
    let len = msg.len() as u32;

    let mut output: u32 = 0;
        let handle = unsafe { GetStdHandle(-11) };
        if handle  == -1 {
            return Err(io::Error::last_os_error())
        }

        let res = unsafe {
            WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null())
            };

        if res  == 0 {
            return Err(io::Error::last_os_error());
        }

    assert_eq!(output, len);
    Ok(())
}
```

这里做的第一件事是把文本转换成 Windows 使用的 utf-16 编码。
幸好 Rust 有一个内置的函数进行这种转换。
`encode_utf16` 返回 `u16` 代码点的迭代器，从而可以变成 `Vec` 。

```rust, ignored
let msg: Vec<u16> = message.encode_utf16().collect();
let msg_ptr = msg.as_ptr();
let len = msg.len() as u32;
```

Next, we get the pointer to the underlying buffer of our `Vec` and get the
length.

```rust, ignored
let handle = unsafe { GetStdHandle(-11) };
   if handle  == -1 {
       return Err(io::Error::last_os_error())
   }
```

下一步是调用 `GetStdHandle` 。这里给它传入 值 `-11` 。
针对不同标准设备 (standard devices) 传入的值在 `GetStdHandle` 的文档里有描述：

| Handle | Value |
| ------ | ----- |
| Stdin  |   -10 |
| Stdout |   -11 |
| StdErr |   -12 |

找这里所调用函数的文档和信息并不常见，但是当我们的确需要找这些时，并不困难找到。

返回值的代码（比如 0、-1 的含义）在文档中也描述得很详细，
这里和 Linux/macOS 系统调用一样，对潜在的错误进行处理。

```rust, ignored
let res = unsafe {
    WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null())
    };

if res  == 0 {
    return Err(io::Error::last_os_error());
}
```

这一步调用了 `WriteConsoleW` 函数。没有太特别要说明的。

## 最高层次的抽象

这是一个简单的例子，大多数库会提供高级别的抽象。在 Rust 中，可以如此简单：

```rust
println!("Hello world from Stdlib");
```

## 最终跨平台系统调用代码

```rust
use std::io;

fn main() {
    let sys_message = String::from("Hello world from syscall!\n");
    syscall(sys_message).unwrap();
}

// and: http://man7.org/linux/man-pages/man2/write.2.html
#[cfg(not(target_os = "windows"))]
#[link(name = "c")]
extern "C" {
    fn write(fd: u32, buf: *const u8, count: usize) -> i32;
}

#[cfg(not(target_os = "windows"))]
fn syscall(message: String) -> io::Result<()> {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    let res = unsafe { write(1, msg_ptr, len) };

    if res == -1 {
        return Err(io::Error::last_os_error());
    }
    Ok(())
}

#[cfg(target_os = "windows")]
#[link(name = "kernel32")]
extern "stdcall" {
    /// https://docs.microsoft.com/en-us/windows/console/getstdhandle
    fn GetStdHandle(nStdHandle: i32) -> i32;
    /// https://docs.microsoft.com/en-us/windows/console/writeconsole
    fn WriteConsoleW(
        hConsoleOutput: i32,
        lpBuffer: *const u16,
        numberOfCharsToWrite: u32,
        lpNumberOfCharsWritten: *mut u32,
        lpReserved: *const std::ffi::c_void,
    ) -> i32;
}

#[cfg(target_os = "windows")]
fn syscall(message: String) -> io::Result<()> {

    // let's convert our utf-8 to a format windows understands
    let msg: Vec<u16> = message.encode_utf16().collect();
    let msg_ptr = msg.as_ptr();
    let len = msg.len() as u32;

    let mut output: u32 = 0;
        let handle = unsafe { GetStdHandle(-11) };
        if handle  == -1 {
            return Err(io::Error::last_os_error())
        }

        let res = unsafe {
            WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null())
            };

        if res  == 0 {
            return Err(io::Error::last_os_error());
        }

    assert_eq!(output, len);
    Ok(())
}
```
