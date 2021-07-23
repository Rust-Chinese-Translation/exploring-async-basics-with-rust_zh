# 介绍主要例子

现在我们终于来到了本书编写一些代码的部分。

Node 事件循环 (event loop) 是一个经过多年开发的复杂软件。本书将不得不简化很多事情。

本书将尝试实现对我们更好地理解 Node 的重要部分。
最重要的是将它用作示例，我们可以在这当中使用从前几章中获得的知识来做出实际工作的东西。

我们的主要目标是探索异步概念。以 Node 为例主要是为了好玩。

**我们想写这样的东西：**

```rust, ignored
/// 把这个函数想象成你编写的 Javascript 程序
fn Javascript() {
    print("First call to read test.txt");
    Fs::read("test.txt", |result| {
        let text = result.into_string().unwrap();
        let len = text.len();
        print(format!("First count: {} characters.", len));

        print(r#"I want to create a "magic" number based on the text."#);
        Crypto::encrypt(text.len(), |result| {
            let n = result.into_int().unwrap();
            print(format!(r#""Encrypted" number is: {}"#, n));
        })
    });

    print("Registering immediate timeout 1");
    set_timeout(0, |_res| {
        print("Immediate1 timed out");
    });
    print("Registering immediate timeout 2");
    set_timeout(0, |_res| {
        print("Immediate2 timed out");
    });

    // 再次读取文件并显示文本
    print("Second call to read test.txt");
    Fs::read("test.txt", |result| {
        let text = result.into_string().unwrap();
        let len = text.len();
        print(format!("Second count: {} characters.", len));

        // 再一次读文件，但不是并行的。
        print("Third call to read test.txt");
        Fs::read("test.txt", |result| {
            let text = result.into_string().unwrap();
            print_content(&text, "file read");
        });
    });

    print("Registering a 3000 and a 500 ms timeout");
    set_timeout(3000, |_res| {
        print("3000ms timer timed out");
        set_timeout(500, |_res| {
            print("500ms timer(nested) timed out");
        });
    });

    print("Registering a 1000 ms timeout");
    set_timeout(1000, |_res| {
        print("SETTIMEOUT");
    });

    // `http_get_slow` 让我们定义一个想要模拟的延迟
    print("Registering http get request to google.com");
    Http::http_get_slow("http//www.google.com", 2000, |result| {
        let result = result.into_string().unwrap();
        print_content(result.trim(), "web call");
    });
}

fn main() {
    let rt = Runtime::new();
    rt.run(Javascript);
}
```

> 我们将在代码中的重要位置打印语句，以便我们可以了解实际代码发生的时间和地点。

马上你就会发现我们在示例中编写的 Rust 代码有些奇怪。

当我们想要像 Javascript 一样执行异步操作时，代码会使用回调，
并且我们可以调用“神奇的”模块，例如 `Fs` 或 `Crypto`，就像从 Node 导入模块一样。

我们这里的代码主要是调用用于注册事件和存储回调的函数，以便在事件准备好时运行。

**一个例子是 `set timeout` 函数：**

```rust, ignored
set_timeout(0, |_res| {
    print("Immediate1 timed out");
});
```

在这里实际上注册了感兴趣的 `timeout` 事件，当该事件发生时，
我们希望运行回调函数 `|_res| { print("Immediate1 timed out");  }`。

`_res` 是一个传递给回调的参数。在 Javascript 中，它会被省略，
但由于我们使用了类型化语言，因此我们创建了一个名为 `Js` 的类型。

`Js` 是一个代表 Javascript 类型的枚举体。
在 `set_timeout` 的情况下，它是 `Js::undefined` 。
在 `Fs::read` 的情况下，它是 `Js::string` 等。

运行这段代码后，我们会得到如下所示的内容：

<video autoplay loop>
<source src="./images/example_run.mp4" type="video/mp4">
Can't display video.
</video>

输出结果：

```
Thread: main     First call to read test.txt
Thread: main     Registering immediate timeout 1
Thread: main     Registered timer event id: 2
Thread: main     Registering immediate timeout 2
Thread: main     Registered timer event id: 3
Thread: main     Second call to read test.txt
Thread: main     Registering a 3000 and a 500 ms timeout
Thread: main     Registered timer event id: 5
Thread: main     Registering a 1000 ms timeout
Thread: main     Registered timer event id: 6
Thread: main     Registering http get request to google.com
Thread: pool3    received a task of type: File read
Thread: pool2    received a task of type: File read
Thread: main     Event with id: 7 registered.
Thread: main     ===== TICK 1 =====
Thread: main     Immediate1 timed out
Thread: main     Immediate2 timed out
Thread: pool3    finished running a task of type: File read.
Thread: pool2    finished running a task of type: File read.
Thread: main     First count: 39 characters.
Thread: main     I want to create a "magic" number based on the text.
Thread: pool3    received a task of type: Encrypt
Thread: main     ===== TICK 2 =====
Thread: main     SETTIMEOUT
Thread: main     Second count: 39 characters.
Thread: main     Third call to read test.txt
Thread: main     ===== TICK 3 =====
Thread: pool2    received a task of type: File read
Thread: pool3    finished running a task of type: Encrypt.
Thread: main     "Encrypted" number is: 63245986
Thread: main     ===== TICK 4 =====
Thread: pool2    finished running a task of type: File read.

===== THREAD main START CONTENT - FILE READ =====
Hello world! This is text to encrypt!
... [Note: Abbreviated for display] ...
===== END CONTENT =====

Thread: main     ===== TICK 5 =====
Thread: epoll    epoll event 7 is ready

===== THREAD main START CONTENT - WEB CALL =====
HTTP/1.1 302 Found
Server: Cowboy
Location: http://http/www.google.com
... [Note: Abbreviated for display] ...
===== END CONTENT =====

Thread: main     ===== TICK 6 =====
Thread: epoll    epoll event timeout is ready
Thread: main     ===== TICK 7 =====
Thread: main     3000ms timer timed out
Thread: main     Registered timer event id: 10
Thread: epoll    epoll event timeout is ready
Thread: main     ===== TICK 8 =====
Thread: main     500ms timer(nested) timed out
Thread: pool0    received a task of type: Close
Thread: pool1    received a task of type: Close
Thread: pool2    received a task of type: Close
Thread: pool3    received a task of type: Close
Thread: epoll    received event of type: Close
Thread: main     FINISHED
```

别担心，本书会解释一切，但我只是想从解释我们结束的地方开始。
