---
unlisted: true
---

# Rust Async Traps

<!-- truncate -->

## Blocking scheduler thread

Async runtimes (like Tokio) has its own scheduler. It's similar to OS thread scheduling but different in many ways:

- It happens within Rust application. It uses Rust async to switch control flow. It doesn't use OS functionalties. It's in userspace.
- It's coorporative. If the scheduled Rust code don't coorporatively pause, it won't forcefully suspend it, and the scheduler will be always occupied.
- It's more lightweight than OS thread scheduling. Creating a future is faster and require less memory than creating a OS thread. Switching between different Rust futures is faster than OS context switch.

The normal sleep `std::thread::sleep` and normal locking `std::sync::Mutex` will **block thread using OS functionality**. Async runtime won't be notified when they block the current thread. It will occupy the async runtime's scheduling thread. This causes reduced concurrency and possible deadlocks.

In Tokio, use `tokio::sync::Mutex` for mutex and `tokio::time::sleep` and sleep. They will coorporatively pause and avoid that issue.

That issue is not limited to only locking and sleep. It also involves networking and all kinds of IOs.

Tokio supports [`spawn_blocking`](https://dtantsur.github.io/rust-openstack/tokio/task/fn.spawn_blocking.html) which make it run in new scheduler thread. The code that does non-async blocking should be ran in `spawn_blocking`.

### Deadlock caused by blocking scheduler thread

[How to deadlock Tokio application in Rust with just a single mutex](https://turso.tech/blog/how-to-deadlock-tokio-application-in-rust-with-just-a-single-mutex)

[Why do I get a deadlock when using Tokio with a std::sync::Mutex?](https://stackoverflow.com/questions/63712823/why-do-i-get-a-deadlock-when-using-tokio-with-a-stdsyncmutex)


## Cancellation safety

Rust's future is very different to Java `CompletableFuture` and JS `Promise` and C# `Task`. In Java/JS/C#, you launch a task then get a future/promise/task object that represents the async task. The task will run regardless whether you discard the future. But in Rust, when you create a future, the task is not yet launched. It will be launched when the future is firstly polled.

In Rust, an async function may suddenly stop executing in an `await` point.

In Rust, futures are not background tasks. Futures only progress when polled. There are two kinds of async cancellation in Rust:

- The future won't be polled, but it's not yet dropped. (This is prone to deadlock.)
- The future is dropped. It obviously won't be polled.

In either case, the async function will suddenly stop executing in an `await` point.

This behavior of async Rust is very different to other languages. In Golang a goroutine can suddenly stop executing if it panics. In Java a thread can suddenly stop executing if there is an exception. It's easier to debug in Golang and Java because panics and exceptions are usually logged. But in Rust the dropping of future is not logged so it's harder to debug.

Although it's called "cancellation", the already-done IO operations won't be cancelled (written files won't be rolled back, sent packets won't be magically withdrawn). The cancellation here just stops the async function from continuing.

In tokio, these are the main ways of cancellation:

- `tokio::select!`
- `JoinHandle::abort`
- `tokio::time::timeout`

Once a parent future is cancelled, its children futures are also dropped.

`tokio::select` the value of non-selected cases will be dropped. `tokio::select` very different to Golang select. In Golang select, the non-selected cases just don't consume data from channel.

In `tokio::select`, for each case, you can:

- Pass an owned future. If that case is not selected, the future will be dropped. This cause uncorporative cancellation.
- Pass a pinned mutable borrow of future. If that case is not selected, `select` will proceed without further polling that future, but the future itself won't be dropped.
- Pass a `JoinHandle`. If that case is not selected, the `JoinHandle` will be dropped but the task of `JoinHandle` won't be cancelled. This way can avoid cancellation.




[Tokio document 1](https://tokio.rs/tokio/tutorial/select#cancellation), [Tokio document2](https://docs.rs/tokio/latest/tokio/macro.select.html#cancellation-safety)

See also: [Making Async Rust Reliable - Tyler Mandry](https://tmandry.gitlab.io/blog/posts/making-async-reliable/)  [400 - Dealing with cancel safety in async Rust / RFD / Oxide](https://rfd.shared.oxide.computer/rfd/400)   [609 - Futurelock / RFD / Oxide](https://rfd.shared.oxide.computer/rfd/0609)  [FuturesUnordered and the order of futures](https://without.boats/blog/futures-unordered/) [Alan tries to cache requests, which doesn't always happen - wg-async](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/alan_builds_a_cache.html) [Barbara gets burned by select](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_gets_burned_by_select.html) [Cancelling async Rust](https://sunshowers.io/posts/cancelling-async-rust/)


## Stackoverflow caused by large future

TODO

https://github.com/rust-lang/rust-project-goals/blob/main/src/2026/async-future-memory-optimisation.md

## No parallelism by default

Example

```rust
#[tokio::main]  
async fn main() {  
    let mut result: Vec<_> = futures::stream::iter(1..=65535)  
        .map(|port| {  
            let host_port = format!("127.0.0.1:{port}");  
            async move {  
                println!("Testing port: {} {:?}", port, thread::current().id());  
                if let Ok(Ok(_)) =  
                    tokio::time::timeout(Duration::from_millis(200), TcpStream::connect(host_port))  
                        .await  
                {  
                    Some(port)  
                } else {  
                    None  
                }  
            }  
        })  
        .buffer_unordered(999999999)  
        .filter_map(|port| async move { port })  
        .collect()  
        .await;  
}
```

It will print 

```
Testing port: 1 ThreadId(1)
Testing port: 2 ThreadId(1)
Testing port: 3 ThreadId(1)
Testing port: 4 ThreadId(1)
...
```

All of them execute on main thread. There is no parallelism. The parallelism can be enabled by using `tokio::spawn`. But without `tokio::spawn` it has no parallelism by default.

This trap doesn't exist in Golang. In Golang, goroutines are parallel.

---

https://without.boats/blog/why-async-rust/

