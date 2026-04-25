---
unlisted: true
---

# Rust Async Traps

<!-- truncate -->



## Rust future is just data by default

In Rust, if you call an async function, it returns a future. But the future is just data by default. If you don't await it or spawn a it, its async code won't run. 

The word "future" has very different meaning in Java. In Java, when obtaining a `CompletableFuture`, the task should be already running.

## Blocking scheduler thread

Async runtime schedules async tasks on threads. When an async task suspends, the thread can run other async tasks. 

But it requires the async task to cooperatively suspend (`.await`). An async task can keep running without `.await` for long time, and the async runtime cannot force-suspend it. Then a scheduler thread will be kept occupied. This is called **blocking the scheduler thread**. 

When a scheduler thread is blocked, it reduces overall concurrency and reduces overall performance. And it may cause deadlock.

The normal sleep `std::thread::sleep` and normal locking `std::sync::Mutex` will block thread using OS functionality. When a thread is blocked by OS, async runtime don't know about it. In Tokio, use `tokio::sync::Mutex` for mutex and `tokio::time::sleep` and sleep. They will coorporatively pause and avoid that issue.

That issue is not limited to only locking and sleep. It also involves networking and all kinds of IOs. So Tokio provides its own set of IO functionalities, and you have to use them when using Tokio for max performance.

Also, heavy computation work without `.await` point is also blocking. The async runtime cannot force-suspend the heavy computation if it doesn't cooperatively `.await`. 

Tokio also supports an "escape hatch". The task spawned by [`spawn_blocking`](https://dtantsur.github.io/rust-openstack/tokio/task/fn.spawn_blocking.html) runs in another thread pool and won't block the normal scheduler thread. The code that does non-async blocking or heavy compute work should be ran in `spawn_blocking`.

### Deadlock caused by blocking scheduler thread

[How to deadlock Tokio application in Rust with just a single mutex](https://turso.tech/blog/how-to-deadlock-tokio-application-in-rust-with-just-a-single-mutex)

[Why do I get a deadlock when using Tokio with a std::sync::Mutex?](https://stackoverflow.com/questions/63712823/why-do-i-get-a-deadlock-when-using-tokio-with-a-stdsyncmutex)


## Cancellation safety

In Rust, a future can be cancelled. When it's cancelled, the async task stops executing in an await point. It's a **implicit exit** mechanism. The control flow of it is not obvious in code.

It's not the only implicit exit mechanism. Panic is another implicit exit mechanism. And in the languages that have exceptions (Java, JS, Python, etc.), exception is another implciit exit mechanism.

However, **exceptions and panics are often logged, but future cancel is often not logged**. Although panic is implicit code control flow, it's often explicit in logs. It's easy to debug because it's visible in log. But a future cancel by default logs nothing. Debugging future cancel issue is much harder than debugging panics.

| Exception and panic                                | Rust future cancellation                                               |
| -------------------------------------------------- | ---------------------------------------------------------------------- |
| Implicit control flow of exiting function.         | Implicit control flow of exiting async function.                       |
| Often logged. Easy to notice.                      | Not logged by default. Hard to notice.                                 |
| Propagates from inside to outside. Can be catched. | Propagates from outside to inside. Can be "catched" by `tokio::spawn`. |

The cancellation "catch": normally when the parent future cancels, the inner futures are also cancelled. It propagates from outside to inside. The `tokio::spawn` can stop that propagation. Although `JoinHandle` is `Future`, dropping it won't cancel the spawned task. So if you want to avoid cancellation, wrap it in `tokio::spawn` (and don't call `JoinHandle::abort`).

Although it's called "cancellation", the already-done IO operations won't be cancelled. The written files won't be rolled back. The sent packets won't be magically withdrawn. The cancellation here just stops the async function from continuing in an await point.

In Golang, there is panic, but there is no implcit cancellation. All cancellation need to be explicit. However managing context cancellation in Golang still has traps, just different to async Rust.

Two examples of cancellation issues: [Alan tries to cache requests, which doesn't always happen - wg-async](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/alan_builds_a_cache.html), [Barbara gets burned by select](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_gets_burned_by_select.html)

See also: [Dealing with cancel safety in async Rust / RFD / Oxide](https://rfd.shared.oxide.computer/rfd/400), [Cancelling async Rust](https://sunshowers.io/posts/cancelling-async-rust/)

### Common sources of cancellation in Tokio

- `tokio::select!`. When one branch is selected, the futures of other branches are cancelled.
- `JoinHandle::abort`. Explcitly cancel a task.
- `tokio::time::timeout`. When timeout is reached but the future hasn't finished, it's cancelled.

Tokio documentation about cancellation safety: [1](https://docs.rs/tokio/latest/tokio/macro.select.html#cancellation-safety), [2](https://tokio.rs/tokio/tutorial/select#cancellation)

### io_uring issue

- In epoll, the OS notifies app that an IO can be done, then the app does another system call to do IO. It involves context switching from kernel to app then to kernel.
  - The app can choose to not do the IO after receiving notification. This works well with Rust async cancellation.
- In io_uring, the OS directly finish IO (write to buffer) then tell the app. It avoids that context switching from kernel to app then to kernel.
  - The whole process is done by kernel. The app cannot choose to "receive notification but not do IO". When app receives notification, the IO has been done. This doesn't work well with Rust async cancellation.

Nuance of "cancel": cancelling a Rust future just drops the future object (and un-tracked by async runtime). It doesn't cancel the IO operation.

With epoll, the buffer can be directly put inside future, with no extra allocation. If the Rust future is dropped, it just don't do the IO after being notified.

With io_uring, dropping the future doesn't cancel the io_uring's IO process. So putting buffer into future in io_uring is not memory-safe on cancellation (kernel will write into freed memory). Two solutions:

- Make the future non-cancellable. Rust doesn't yet have linear type (must-move type) so this cannot be guaranteed by language.
- Make the buffer heap-allocated. When future is dropped, the buffer can still exist, kernel can write to it without violating memory safety.

See also: [Notes on io-uring](https://without.boats/blog/io-uring/)

### Abandoned future and futurelock

In `tokio::select!` you can pass ownership of a future, but you can also pass a future borrow. When a future borrow is passed, one dangerous case can happen. 

If the select goes into one branch, the future of other branches are dropeed. If you pass a future borrow to it, the borrow itself is dropped, but the borrowed future is not dropped. However, the borrowed future will not be polled again before the `select!` finishes, even if you explicit await it after the `select!`.

This creates a temporaily abandoned future. It's not polled but not dropped. This is dangerous when async lock is involved. After acquiring lock, the future holds lock. If the future holding lock is dropped, it released lock. But if the future holds lock but not dropped and not polled, it's likely to deadlock. This is the mechanism behind [futurelock](https://rfd.shared.oxide.computer/rfd/0609).

## Stack overflow caused by large future

TODO

https://github.com/rust-lang/rust-project-goals/blob/main/src/2026/async-future-memory-optimisation.md

heap-allocating the future can avoid it. but rust currently has no in-place initialization. conceptually, it firstly creates future on stack then move to heap. in release mode it can be optimized to directly creating on heap. but in debug it still involves creating on stack first.

## No parallelism without `spawn`

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

This is different in Golang. In Golang, goroutines are parallel.

## Buffered stream issue

https://tmandry.gitlab.io/blog/posts/for-await-buffered-streams/

https://without.boats/blog/poll-progress/

## Converting between async and sync

- Making async code call sync code is easy, but has risk of blocking scheduler thread, as mentioned previously.
- Making sync code call async is not easy. It requires using async runtime's API. But it's less risky.

Async-sync-async sandwitch: Async function call sync function that blocks on another async function. Its async-to-sync calling blocks scheduler thread. It's very prone to deadlock.

## Rust `Send` `Sync` limitations

Tokio does multi-thread work-stealing scheduling. Its purpose is very similar to OS scheduling. And an async task's purpose is very similar to OS thread.

The duality of the two:

| OS thread scheduling                                          | Tokio async task scheduling                                                        |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Schedules threads on CPU cores                                | Schedules async tasks on threads                                                   |
| Spawn a thread                                                | Spawn an async task                                                                |
| Join on a thread                                              | Await on a `JoinHandle`                                                            |
| Can do forced scheduling (using hardware interrupt)           | If the async function don't suspend cooperatively, the scheduler cannot suspend it |
| As long as the data is owned by a thread, it's data-race free | As long as the data is owned by an async task, it's data-race free                 |

As long as the data is owned by a thread, it's data-race free. The correspondence: as long as the data is owned by an async task, it's data-race free.

Tokio `spawn` requires the future to be `Send + Sync`. This can create some troubles. It requires `Send + Sync` because Tokio does work stealing. An async task in one thread could be then scheduled to another async task. However if async task is analogous to thread, then if we ensure that the data is owned by async task, it can also achieve data-race free, even if the data is not `Send + Sync`.

However Rust doesn't check "async task boundary". An async task can pass data out. Then the data is no longer owned by async task. There is no language mechanism that ensures that the data is tied within async task. So you still have to satisfy `Send + Sync` even for the data that's only used with one async task.

The `Send + Sync` constraint can be avoided for thread-per-core async runtimes.

## Mixing multiple async runtimes is hard

Using multiple async runtimes together is possible but is hard and error-prone. And there are many async-runtime-specific types. So async runtime naturally has exclusion. That's why Tokio has monopoly.

In Golang you can only use one official goroutine scheduler. In Rust, although Tokio has monopoly, you have choices of using other async runtimes.

## Unbound concurrency use up resources

This trap is not Rust-specific. When using thread pool, it often has thread count limit, which limits concurrency. But in async, there is no concurrency limit by default. This is good for high-performance web server. But it has downsides:

- For scraper, if concurrency is too high, it may use too much memory then OOM.
- If it sends too many concurrent requests to a remote server, it may trigger rate limit then most requests fail.

One solution is to add a semaphore to limit concurrency.

## About structural concurrency

Structural concurrency force all concurrent tasks to be scoped. Then the tasks form a tree-shaped structure.

Structural concurrency can borrow data from parent. There is no need to make the future `'static`. There is no need to wrap things in `Arc`.

The tree shape is free of cycles, so awaiting on child tasks alone cannot deadlock (but it can deadlock if other kinds of waits are involved).

But there are cases that structural concurrency cannot handld. One is background tasks. For example, a web server provides a Restful API that launches a background task. The background task keeps running after the request that launch task finishes.


## Duplicated APIs

[The bane of my existence: Supporting both async and sync code in Rust](https://nullderef.com/blog/rust-async-sync/)

## See also

[Why async Rust?](https://without.boats/blog/why-async-rust/)

https://emschwartz.me/async-rust-can-be-a-pleasure-to-work-with-without-send-sync-static/

[Making Async Rust Reliable - Tyler Mandry](https://tmandry.gitlab.io/blog/posts/making-async-reliable/) 

[FuturesUnordered and the order of futures](https://without.boats/blog/futures-unordered/) 




