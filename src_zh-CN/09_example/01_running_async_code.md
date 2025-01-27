# 运行异步代码

一个 HTTP 服务器 理应能够并发地服务多个客户端；也就是，它不应该等待前一个请求完成后才处理当前请求。官方书通过创建一个线程池，使得每个链接都由独立的线程处理来 [解决这个问题](https://doc.rust-lang.org/book/ch20-02-multithreaded.html#turning-our-single-threaded-server-into-a-multithreaded-server)。在这里，与其通过增加线程来提高吞吐量，我们使用异步代码来达到同样的效果。

让我们修改 `handle_connection` 来返回一个 future，只需要声明为 `async fn`:

```rust,ignore
{{#include ../../examples/09_02_async_tcp_server/src/main.rs:handle_connection_async}}
```

在函数声明里加上 `async` 会改变它的返回类型，从单元类型 `()` 变成一个实现了 `Future<Output=()>` 的类型。

如果我们尝试编译这个代码，编译器会警告我们它不会工作：

```console
$ cargo check
    Checking async-rust v0.1.0 (file:///projects/async-rust)
warning: unused implementer of `std::future::Future` that must be used
  --> src/main.rs:12:9
   |
12 |         handle_connection(stream);
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: `#[warn(unused_must_use)]` on by default
   = note: futures do nothing unless you `.await` or poll them
```

因为我们还没有 `await` 或者 `poll` `handle_connection` 返回的结果，它永远不会运行的。如果你这时运行了服务器，并且在浏览器中访问 `127.0.0.1:7878`，你会看到链接被拒绝了；我们的服务器没有在处理请求。

我们不能在同步代码中 `await` 或者 `poll` future 类型。我们需要一个异步运行时来处理调度及运行 future 类型至完成状态。请在 [选择一个运行时小节](../08_ecosystem/00_chapter.md) 中获取更多关于异步运行时，执行器和反应器的信息。

## 增加异步运行时

这里我们会使用 `async-std` 库的执行器。`async-std` 库里的 `#[async_std::main]` 属性允许我们编写异步的 main 函数。为了使用它，得先在 `Cargo.toml` 里启用 `async-std` 的 `attributes` 特性：

```toml
[dependencies.async-std]
version = "1.6"
features = ["attributes"]
```

第一步，我们要切换到异步的 main 函数，并且 `await` 异步版 `handle_connection` 函数返回的 future。然后，我们需要测试服务器是怎样响应的。这里是代码应有的样子：

```rust
{{#include ../../examples/09_02_async_tcp_server/src/main.rs:main_func}}
```

现在，我们来测试看看是否我们的服务器能够并发地处理连接。简单的把 `handle_connection` 改成异步并不意味着服务器可以同时处理多个链接，我们将看到为什么。

为了阐明这个原因，我们来模拟一个缓慢的请求。当客户端请求到 `127.0.0.1:7878/sleep` 时，我们的服务器会休眠 5 秒：

```rust,ignore
{{#include ../../examples/09_03_slow_request/src/main.rs:handle_connection}}
```

这非常类似官方书中 [模拟慢请求](https://doc.rust-lang.org/book/ch20-02-multithreaded.html#simulating-a-slow-request-in-the-current-server-implementation)一节，但是有个重要区别：我们使用的是非阻塞函数 `async_std::task::sleep` 而不是阻塞函数 `std::thread::sleep`。这很重要，要记住一块代码是在 `async fn` 中并且被 `await`，因为它可能会阻塞。为了测试我们的服务器能否并发处理连接，我们需要保证 `handle_connection` 是非阻塞的。

如果你运行这个服务器，你会看到，一个发送给 `127.0.0.1:7878/sleep` 的请求，会阻塞其他后续的请求 5秒！这是因为当我们在 `await` `handle_connection` 的结果时，没有其他的并发任务能有进展。在下一小节，我们会看到如何使用异步代码来并发处理连接。
