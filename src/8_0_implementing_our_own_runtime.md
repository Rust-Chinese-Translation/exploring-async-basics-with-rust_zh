# 实现自己的运行时

让我们开始写一些代码；我们有很多事情要做。

我将按照我认为最容易解析和理解的方式来处理开展所有工作。
这也意味着有时我必须引入一些稍后将解释的代码。
如果您一开始不了解某些内容，请不要担心。我会解释一切。

我们需要做的第一件事是创建一个 Rust 项目来运行我们的代码：

```shell
cargo new async-basics
cd async-basics
```

我们需要使用 `minimio` 库

在 `Cargo.toml` 中添加：

```toml
[dependencies]
minimio = {git = "https://github.com/cfsamson/examples-minimio", branch = "master"}
```

然后 clone 包含我们将要编写的所有代码的 repo ：

```
git clone https://github.com/cfsamson/examples-node-eventloop
```

接下来我们需要一个 `Runtime` 来保存我们的 `Runtime` 需要的所有状态。

首先导航到 `main.rs`（位于 `src/main.rs` ）。

> 这次我们将把所有内容都写到一个文件中，其顺序与我们在本书中的顺序大致相同。

# 定义的结构体
## `Runtime`

我在代码中添加了注释，以便更容易记住和理解。

```rust, ignored
pub struct Runtime {
    /// 线程池的可用线程
    available_threads: Vec<usize>,
    /// 计划运行的回调
    callbacks_to_run: Vec<(usize, Js)>,
    /// 所有注册的 (registered) 回调
    callback_queue: HashMap<usize, Box<dyn FnOnce(Js)>>,
    /// 待处理的 epoll 事件的数量，仅用于在此示例中打印
    epoll_pending_events: usize,
    /// 事件注册器，它向操作系统注册对感兴趣的事件
    epoll_registrator: minimio::Registrator,
    // epoll 线程的句柄
    epoll_thread: thread::JoinHandle<()>,
    /// None = 无限延时, Some(n) = 延时 n ms , Some(0) = 立即执行
    epoll_timeout: Arc<Mutex<Option<i32>>>,
    /// 线程池和 epoll 线程使用的通道 (channel) ，用来将事件发送到主循环
    event_reciever: Receiver<PollEvent>,
    /// 给回调创建一个唯一的标识
    identity_token: usize,
    /// 待处理事件的数量。当它为零时，我们就完成了所有任务。
    pending_events: usize,
    /// 线程池中的线程句柄
    thread_pool: Vec<NodeThread>,
    /// 保存所有的计时器，和一旦计时器过期就运行回调的 Id
    timers: BTreeMap<Instant, usize>,
    /// 临时保存待移除的定时器的结构体。我们让运行时拥有所有权，以便可以再次使用相同的内存
    timers_to_remove: Vec<Instant>,
}
```

现在，我在这里添加了一些注释来解释它们的用途，在接下来的章节中，本书将说明上面每一处地方。

我们将继续定义上面使用的一些类型。

## `Task`

```rust, ignored
struct Task {
    task: Box<dyn Fn() -> Js + Send + 'static>,
    callback_id: usize,
    kind: ThreadPoolTaskKind,
}

impl Task {
    fn close() -> Self {
        Task {
            task: Box::new(|| Js::Undefined),
            callback_id: 0,
            kind: ThreadPoolTaskKind::Close,
        }
    }
}
```

我们需要一个任务对象，它代表我们想要在线程池中完成的任务。
我将在稍后的 [章节](./8_9_infrastructure.md) 中详细介绍这个对象的类型，所以如果你现在觉得难以理解，不要太担心。
一切都会得到解释。

我们还创建了 `Close` 方法，它用来任务完成之后清理自己并关闭线程池。

`|| Js::Undefined` 可能看起来很奇怪，但它只是一个返回 `Js::Undefined` 的函数，
写成这样是因为我们不会仅针对这种情况而将 `task` 设为 `Optiona` 类型。

这只是让我们不必在代码中一直对 `task` 进行 `match` 或 `map` ，解析已经绰绰有余了。

## `NodeThread`

`NodeThread` 代表线程池中的一个线程；它有一个 `JoinHandle`（调用 `thread::spawn` 时得到它）
和一个 channel 的 `Sender` 。此 channel 发送类型为 `Task` 的消息。

```rust, ignored
#[derive(Debug)]
struct NodeThread {
    pub(crate) handle: JoinHandle<()>,
    sender: Sender<Task>,
}
```

这里引入了两种新类型：`Js` 和 `ThreadPoolTaskKind` 。
首先，我们将介绍 `ThreadPoolTaskKind` (线程池任务种类) 。

在示例中，有三种事件：

1. `FileRead`：已读取文件
2. `Encrypt`：表示来自 `Crypto` 模块的操作
3. `Close`：让 `threadpool` 中的线程关闭循环，并让线程在退出进程之前结束。

# 定义的枚举体
## `ThreadPoolTaskKind`

正如你可能理解的，这个对象只在 `threadpool` 中使用：

```rust, ignored
pub enum ThreadPoolTaskKind {
    FileRead,
    Encrypt,
    Close,
}
```

## `Js`

接下来是`Js` 对象。它代表了不同的` Javascript` 类型，
它只是用来让我们的代码看起来更像 Javascript，也方便我们抽象闭包的返回类型。

我们还将在这个对象上实现两个方便的方法，使我们的 Javascripty 代码看起来更简洁一些。

我们已经根据我们的模块文档知道返回类型 —— 就像你在使用 Node 模块时从文档中知道它一样，
但我们实际上需要处理 Rust 中的类型。

```rust, ignored
#[derive(Debug)]
pub enum Js {
    Undefined,
    String(String),
    Int(usize),
}

impl Js {
    /// 这是一个方便的方法，因为我们知道类型
    fn into_string(self) -> Option<String> {
        match self {
            Js::String(s) => Some(s),
            _ => None,
        }
    }

    /// 这是一个方便的方法，因为我们知道类型
    fn into_int(self) -> Option<usize> {
        match self {
            Js::Int(n) => Some(n),
            _ => None,
        }
    }
}
```

## `PollEvent`

接下来我们定义 `PollEvent` (轮询事件) 。
虽然我们定义了 `ThreadPoolTaskKind` 来表示我们可以**发送**给 `eventpool` 的事件类型，
但我们现在要定义一些接收事件，它们来自于 epoll 的事件队列和 `threadpool` 。

```rust, ignored
/// 描述了 epoll-eventloop 处理的三个主要事件
enum PollEvent {
    /// 来自 `threadpool` 的事件，其元组包含 `thread_id` `callback_id`
    /// 和我们期望在回调中处理的数据
    Threadpool((usize, usize, Js)),
    /// 来自基于 epoll 的 eventloop 的事件，其中包含事件的 `event_id`
    Epoll(usize),
    Timeout,
}
```

# static `RUNTIME`

最后定义一个方便而必要的静态变量，让我们的 Javascript 代码看起来有点像 Javascript 。

它代表“运行时”；实际上是一个指向我们的 `Runtime` 的指针，我们一开始将它初始化为一个空指针。

我们需要使用 unsafe 来修改它。
我稍后会解释这是如何安全的，但我也想在这里提一下，
这可以使用 [lazy_static](https://github.com/rust-lang-nursery/lazy-static.rs) 来避免 unsafe 。
但是这不仅需要我们添加 `lazy_static` 作为依赖（这是好事，因为它没有我们想在本书中解释的“魔法”），
而且降低了代码可读性 —— 我们的代码足够复杂了。

```rust, ignored
static mut RUNTIME: *mut Runtime = std::ptr::null_mut();
```

现在让我们继续看看运行时的核心是什么：主循环 (main loop) 。

# 继续

我们已经介绍了运行时大部分所需的东西，已经走得很远了。
下一章将重点介绍实现我们需要的所有功能。
