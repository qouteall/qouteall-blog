---
unlisted: false
date: 2026-05-01
tags:
  - Programming
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

In Rust, a future can be dropped. When it's dropped, its async code stops executing in an await point. This is called cancellation. It's a **implicit exit** mechanism. The control flow of it is not obvious in code.

Note it cancels the future, not the IO. Cancelling a future just stops the async code from running (and drop related data). The already-done IO operations won't be cancelled. (The written files won't be magically rolled back. The sent packets won't be magically withdrawn.)

Cancellation not the only implicit exit mechanism. Panic is another implicit exit mechanism. And in the languages that have exceptions (Java, JS, Python, etc.), exception is another implciit exit mechanism.

However, **exceptions and panics are often logged, but future cancel is often not logged**. Although panic is implicit code control flow, it's often explicit in logs. It's easy to debug because it's visible in log. But a future cancel by default logs nothing. Debugging future cancel issue is much harder than debugging panics.

| Exception and panic                                | Rust future cancellation                                               |
| -------------------------------------------------- | ---------------------------------------------------------------------- |
| Implicit control flow of exiting function.         | Implicit control flow of exiting async code.                           |
| Often logged. Easy to notice.                      | Not logged by default. Hard to notice.                                 |
| Propagates from inside to outside. Can be catched. | Propagates from outside to inside. Can be "catched" by `tokio::spawn`. |

The cancellation "catch": normally when the parent future cancels, the inner futures are also cancelled. It propagates from outside to inside. The `tokio::spawn` can stop that propagation. Although `JoinHandle` is `Future`, dropping it won't cancel the spawned task. So if you want to avoid cancellation, wrap it in `tokio::spawn` (and don't call `JoinHandle::abort`).

In Golang, there is panic, but there is no implcit cancellation. All cancellation need to be explicit. (However managing context cancellation in Golang still has traps, just different to async Rust.)

Cancellation is indeed useful in some cases. But there are also many cases that cancellation is harmful. The problem is that async Rust made cancellation implicit. There is no type-level annotation ensuring an async function cannot cancel. This creates traps.

Two examples of cancellation issues: [Alan tries to cache requests, which doesn't always happen](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/alan_builds_a_cache.html), [Barbara gets burned by select](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_gets_burned_by_select.html)

See also: [Dealing with cancel safety in async Rust](https://rfd.shared.oxide.computer/rfd/400), [Cancelling async Rust](https://sunshowers.io/posts/cancelling-async-rust/)

There is another kind of "cancel": doesn't drop the future but does not `poll` the future. This is also dangerous. Elaborated below.

### Common sources of cancellation in Tokio

- `tokio::select!`. When one branch is selected, the futures of other branches are cancelled.
- `JoinHandle::abort`. Explcitly cancel a task.
- `tokio::time::timeout`. When timeout is reached but the future hasn't finished, it's cancelled.

Tokio documentation about cancellation safety: [1](https://docs.rs/tokio/latest/tokio/macro.select.html#cancellation-safety), [2](https://tokio.rs/tokio/tutorial/select#cancellation)

Using `tokio::spawn(future).await` rather than `future.await` prevents propagation of both cancellation and panic.

### Debugging cancellation

Cancelling does not log by default. You can use a future wrapper to make it log if it cancels before completion. Example:

```rust
use std::backtrace::Backtrace;
use std::time::Duration;

use pin_project::pin_project;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

/// A Future wrapper for debugging async cancellation.
#[pin_project(PinnedDrop)]
pub struct CancelDebug<F> {
    #[pin]
    inner: F,
    completed: bool,
    name: String,
    created_at: Backtrace,
}

impl<F: Future> CancelDebug<F> {
    pub fn new(name: impl Into<String>, inner: F) -> Self {
        Self {
            inner,
            completed: false,
            name: name.into(),
            created_at: Backtrace::force_capture(),
        }
    }
}

impl<F: Future> Future for CancelDebug<F> {
    type Output = F::Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();
        match this.inner.poll(cx) {
            Poll::Ready(v) => {
                *this.completed = true;
                Poll::Ready(v)
            }
            Poll::Pending => Poll::Pending,
        }
    }
}

#[pin_project::pinned_drop]
impl<F> PinnedDrop for CancelDebug<F> {
    fn drop(self: Pin<&mut Self>) {
        if !self.completed {
            let dropped_at = Backtrace::force_capture();
            eprintln!(
                "Future '{}' was cancelled!\nCreated at:\n{}\nDropped at:\n{}",
                self.name, self.created_at, dropped_at
            );
        }
    }
}

async fn some_work() {
    println!("Begin");
    tokio::time::sleep(Duration::from_secs(2)).await;
    println!("End");
}

#[tokio::main]
async fn main() {
    let f = CancelDebug::new("some work", some_work());
    tokio::time::timeout(Duration::from_secs(1), f).await;
}
```

### How cancellation interacts with io_uring

Cancellation also causes problems with io_uring.

- In epoll, the OS notifies app that an IO can be done, then the app does another system call to do IO. It involves context switching from kernel to app (receive notification), then to kernel (do the IO syscall) then to app (finishing IO).
  - The app can choose to not do the IO after receiving notification. This works well with Rust future cancellation.
- In io_uring, the OS directly finish IO (write to buffer) then tell the app. It's just a context switch from kernel to app (it's faster than epoll's kernel-to-app-to-kernel-to-app).
  - The app cannot choose to "receive notification but not do IO". When app receives notification, the IO has already been done by kernel.

The [`futures::io::AsyncRead`](https://docs.rs/futures/latest/futures/io/trait.AsyncRead.html) works on read operations that write into caller-owned buffer:

```rust
pub trait AsyncRead {
    fn poll_read(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &mut [u8], // Note: caller passes mutable borrow of buffer
    ) -> Poll<Result<usize, Error>>;
    ...
}
```

When the buffer is owned by caller, the buffer will be dropped when caller future cancels. Cancelling Rust future doesn't cancel io_uring operation, so the kernel may write into the freed buffer. This is not memory-safe. 

One workaround is to allocate an extra internal buffer for io_uring, then copy that buffer to caller's buffer after IO completes, but this workaround costs performance. With io_uring, the `AsyncRead` cannot be used without performance cost.

In [`monoio::io::AsyncReadRent`](https://docs.rs/monoio/0.2.4/monoio/io/trait.AsyncReadRent.html), the caller transfer ownership of buffer to it. When IO finishes, it transfers buffer ownership back:

```rust
pub trait AsyncReadRent {
    fn read<T: IoBufMut>(
        &mut self,
        buf: T, // Note: caller transfers buffer ownership
    ) -> impl Future<Output = BufResult<usize, T>>;
    ...
}
```

Having to pass buffer ownership doesn't mean each small buffer have to be separately allocated. It can allocate a large buffer then split it into many small buffers, and use reference counting internally.

See also: [Notes on io-uring](https://without.boats/blog/io-uring/)

## Un-`poll`-ed futures

As previously mentioned, dropping a future cancels it. There is another kind of "cancellation": just not `poll` the future, without dropping the future.

It's also dangerous. It may cause deadlock or weird delaying.

### Futurelock

In `tokio::select!` you can pass ownership of a future, but you can also pass a future borrow. When a future borrow is passed, one dangerous case can happen. 

If the select goes into one branch, the future of other branches are dropeed. If you pass a future borrow to it, the borrow itself is dropped, but the borrowed future is not dropped. However, the borrowed future will not be polled again (you can explicit await it after the `select!`, but it doesn't `poll` before `select!` finishing).

This creates a temporaily un-`poll`-ed future. This is dangerous when async lock is involved. After acquiring lock, the returned future holds lock. If the future holding lock is dropped, it released lock. But if the future holds lock but not dropped and not polled, it's likely to deadlock. This is the mechanism behind [futurelock](https://rfd.shared.oxide.computer/rfd/0609).

### Buffered stream issue

When using buffered stream, some futures in buffer may be temporarily un-`poll`-ed. This can cause weird delaying or deadlock.

See also:

- [Barbara battles buffered streams](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_battles_buffered_streams.html)
- [`for await` and the battle of buffered streams](https://tmandry.gitlab.io/blog/posts/for-await-buffered-streams/)
- [poll_progress](https://without.boats/blog/poll-progress/)

## Stack overflow caused by large future

Rust currently have no in-place initialization. Heap-allocating one thing requires firstly creating it on stack then move it to heap. In release mode, it can be optimized to directly initializing on heap. But in debug mode it still involves creating on stack.

Some futures may be very large. Creating a large future on stack can cause stack overflow.

Sometimes it stack overflows in debug mode but not release mode, because in release mode it directly writes to heap.

In Windows the default stack size is smaller so it's more likely to stackoverflow.

There is currently some inefficiency in future size. See [Async Future Memory Optimisation](https://github.com/rust-lang/rust-project-goals/blob/main/src/2026/async-future-memory-optimisation.md)
How to reduce future size:

- Avoid creating an in-place buffer like `let buf: [u8; 1024]`. The buffer will directly be in the future.
- When calling another async function, firstly box that future then await on it. If not boxed, the sub-future will be directly put inside parent future.

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

## Converting between async and sync

- Making async code call sync code is easy, but has risk of blocking scheduler thread, as mentioned previously.
- Making sync code call async is not easy. It requires using async runtime's API. But it's less risky.

Async-sync-async sandwitch: Async function call sync function that blocks on another async function. Its async-to-sync calling blocks scheduler thread. It's very prone to deadlock.

[The bane of my existence: Supporting both async and sync code in Rust](https://nullderef.com/blog/rust-async-sync/)

## Mixing multiple async runtimes is hard

Using multiple async runtimes together is possible but is hard and error-prone. And there are many async-runtime-specific types. So async runtime naturally has exclusion. That's why Tokio has monopoly.

In Golang you can only use one official goroutine scheduler. In Rust, although Tokio has monopoly, you have choices of using other async runtimes, including thread-per-core async runtimes:

## Thread-per-core async runtimes

Tokio uses work stealing. One async task submitted in one thread can run in another thread, so `tokio::spawn` requires future to be `Send`. There are thread-per-core async runtimes (e.g. [monoio](https://github.com/bytedance/monoio), [glommio](https://github.com/DataDog/glommio)). Thread-per-core async runtimes don't require `Send`. 

Thread-per-core runtimes often relies on the kernel to distribute work between threads evenly. Linux has a mechanism of distributing networking work by hash of local ip+local port+remote ip+remote port, [see also](https://lwn.net/Articles/542629/). If there are many different clients and all requests are processed quickly, then work can be distributed to threads evenly, and thread-per-core can be more efficient than work stealing. But if a small amount of remote ip+remote port combinations have request that requires fat-tailed large processing work, then work distribution will not be even and thread-per-core will be not efficient. Work stealing is more adaptive in that fail-tail workloads.

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


## See also

[Why async Rust?](https://without.boats/blog/why-async-rust/)

[Async Rust can be a pleasure to work with (without `Send + Sync + 'static`)](https://emschwartz.me/async-rust-can-be-a-pleasure-to-work-with-without-send-sync-static/)

[Making Async Rust Reliable - Tyler Mandry](https://tmandry.gitlab.io/blog/posts/making-async-reliable/) 

[FuturesUnordered and the order of futures](https://without.boats/blog/futures-unordered/) 




