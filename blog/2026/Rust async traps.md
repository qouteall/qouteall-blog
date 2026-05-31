---
unlisted: false
date: 2026-05-01
tags:
  - Programming
---

# Rust Async Traps

<!-- truncate -->



## Blocking scheduler thread

Async runtime schedules async tasks on threads. When an async task suspends, the thread can run other async tasks.

But it requires the async task to cooperatively suspend (`.await`). An async task can keep running without `.await` for long time, and the async runtime cannot force-suspend it. Then a scheduler thread will be kept occupied. This is called **blocking the scheduler thread**. 

When a scheduler thread is blocked, it reduces overall concurrency and reduces overall performance. And it may cause deadlock.

The normal sleep `std::thread::sleep` and normal locking `std::sync::Mutex` will block thread using OS functionality. When a thread is blocked by OS, async runtime don't know about it. In Tokio, use `tokio::sync::Mutex` for mutex and `tokio::time::sleep` and sleep. They will cooperatively pause and avoid that issue.

That issue is not limited to only locking and sleep. It also involves networking and all kinds of IOs. So Tokio provides its own set of IO functionalities, and you have to use them when using Tokio for max performance.

Also, heavy computation work without `.await` point is also blocking. The async runtime cannot force-suspend the heavy computation if it doesn't cooperatively `.await`. 

Tokio also supports an "escape hatch". The task spawned by [`spawn_blocking`](https://dtantsur.github.io/rust-openstack/tokio/task/fn.spawn_blocking.html) runs in another thread pool and won't block the normal scheduler thread. The code that does non-async blocking or heavy compute work should be run in `spawn_blocking`.

### Deadlock caused by blocking scheduler thread

[How to deadlock Tokio application in Rust with just a single mutex](https://turso.tech/blog/how-to-deadlock-tokio-application-in-rust-with-just-a-single-mutex)

[Why do I get a deadlock when using Tokio with a std::sync::Mutex?](https://stackoverflow.com/questions/63712823/why-do-i-get-a-deadlock-when-using-tokio-with-a-stdsyncmutex)


## Cancellation safety

In Rust, a future can be dropped. When it's dropped, its async code stops executing in an await point. This is called cancellation. It's a **implicit exit** mechanism. The control flow of it is not obvious in code.

Note it cancels the future, not the IO. Cancelling a future just stops the async code from running (and drop related data). The already-done IO operations won't be cancelled. (The written files won't be magically rolled back. The sent packets won't be magically withdrawn.)

Cancellation is not the only implicit exit mechanism. Panic is another implicit exit mechanism. And in the languages that have exceptions (Java, JS, Python, etc.), exception is another implicit exit mechanism.

However, **exceptions and panics are often logged, but future cancel is often not logged**. Although panic is implicit code control flow, it's often explicit in logs. It's easy to debug because it's visible in log. But a future cancel by default logs nothing. Debugging future cancel issue is much harder than debugging panics.

| Exception and panic                                | Rust future cancellation                                               |
| -------------------------------------------------- | ---------------------------------------------------------------------- |
| Implicit control flow of exiting function.         | Implicit control flow of exiting async code.                           |
| Often logged. Easy to notice.                      | Not logged by default. Hard to notice.                                 |
| Propagates from inside to outside. Can be catched. | Propagates from outside to inside. Can be "catched" by `tokio::spawn`. |

The cancellation "catch": normally when the parent future cancels, the inner futures are also cancelled. It propagates from outside to inside. The `tokio::spawn` can stop that propagation. Although `JoinHandle` is `Future`, dropping it won't cancel the spawned task. So if you want to avoid cancellation, wrap it in `tokio::spawn` (and don't call `JoinHandle::abort`).

Cancellation is indeed useful in some cases. But there are also many cases that cancellation is harmful. The problem is that async Rust made cancellation implicit. There is no type-level annotation ensuring an async function cannot cancel. This creates traps.

Two examples of cancellation issues: [Alan tries to cache requests, which doesn't always happen](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/alan_builds_a_cache.html), [Barbara gets burned by select](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_gets_burned_by_select.html)

See also: [Dealing with cancel safety in async Rust](https://rfd.shared.oxide.computer/rfd/400), [Cancelling async Rust](https://sunshowers.io/posts/cancelling-async-rust/)

There is another kind of "cancel": doesn't drop the future but does not `poll` the future. This is also dangerous. Elaborated below.

Related: [cancel_safe_futures](https://docs.rs/cancel-safe-futures/latest/cancel_safe_futures/) crate

### Rust future is just data by default

If you calls an async function, but don't do anything to the future (don't `.await`, don't `spawn` etc.), it's also cancellation. In Rust, futures are just data.

The word "future" has very different meaning in Java. In Java, when obtaining a `CompletableFuture`, the task should be already running.

### Common sources of cancellation in Tokio

- `tokio::select!`. When one branch is selected, the futures of other branches are dropped.
- `JoinHandle::abort`. Explicitly cancel a task.
- `tokio::time::timeout`. When timeout is reached and the future awaits, it's cancelled.

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

When the buffer is owned by caller, the buffer will be dropped when caller future cancels. Cancelling Rust future doesn't necessarily cancel io_uring operation, so the kernel may write into the freed buffer. This is not memory-safe. 

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

Having to pass buffer ownership doesn't mean each small buffer have to be separately allocated. It can allocate a large buffer then split it into many small buffers that have separate ownerships, and use reference counting internally.

See also: [Notes on io-uring](https://without.boats/blog/io-uring/)

### Holding mutex across await point

Holding `std::sync::Mutex` across await point will not compile when using Tokio. Tokio require future to be `Send`, but `std::sync::MutexGuard` is not `Send`.

In Tokio, `tokio::sync::Mutex` can be held across await point. And it doesn't block scheduler, unlike std mutex. But the future can be cancelled in await point when lock is being held. Then the invariants that the lock aim to protect may break, and there is no related logging by default.

The `tokio::sync::Mutex` doesn't have poisoning mechanism. Poisoning mechanism cannot fix broken invariant, but it makes broken invariant more salient, helping debugging. The cancel_safe_futures provides [`RobustMutex`](https://docs.rs/cancel-safe-futures/latest/cancel_safe_futures/sync/struct.RobustMutex.html) which has poisoning mechanism.

## Un-`poll`-ed futures

As previously mentioned, dropping a future cancels it. There is another kind of "cancellation": just not `poll` the future, without dropping the future.

It's also dangerous. It may cause deadlock or weird delaying.

### Futurelock

In `tokio::select!` you can pass ownership of a future, but you can also pass a future borrow. When a future borrow is passed, the future could become un-`poll`-ed future. When the un-`poll`-ed future holds lock it will deadlock. This is called [futurelock](https://rfd.shared.oxide.computer/rfd/0609).

Here is a simpler example of futurelock:

```rust
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::Mutex;
use tokio::time::sleep;
use futures::FutureExt;

#[tokio::main]
async fn main() {
    let lock = Arc::new(Mutex::new(()));

    tokio::spawn(simulate_contention(lock.clone()));

    // let `simulate_contention` take lock
    sleep(Duration::from_millis(100)).await;

    let mut xxx_future = do_xxx(lock.clone()).boxed();

    tokio::select! {
        _ = &mut xxx_future => { // Note: it passes future borrow to select
            println!("first branch has been run");
        }
        _ = sleep(Duration::from_millis(500)) => {
            println!("second branch has been run");
            do_yyy(lock.clone()).await;
        }
    };
}

async fn do_xxx(lock: Arc<Mutex<()>>) {
    println!("xxx started");
    let _guard = lock.lock().await;
    // ...
    println!("xxx done");
}

async fn do_yyy(lock: Arc<Mutex<()>>) {
    println!("yyy started");
    let _guard = lock.lock().await;
    // ...
    println!("yyy done");
}

async fn simulate_contention(lock: Arc<Mutex<()>>) {
    let _guard = lock.lock().await;
    sleep(Duration::from_secs(5)).await;
    println!("simulate contention done");
}
```

It outputs this then deadlock

```
xxx started
second branch has been run
yyy started
simulate contention done
```

In [`tokio::select!`](https://docs.rs/tokio/latest/tokio/macro.select.html), for each branch, you can pass a future ownership, you can also pass a borrowed future. 

`tokio::select!` will keep `poll`-ing the futures of each branch, until one future finishes [^select_else]. When finishing, `tokio::select!` will drop (cancel) the futures of all branches, as mentioned previously. However, if you pass a future borrow, it will only drop the borrow, but the borrowed future will not be dropped when `tokio::select!` finishes. The borrowed future `xxx_future` then becomes alive but un-`poll`-ed.

Note that `tokio::select!` can run the futures of all branches concurrently, but only one branch's result will be selected.

[^select_else]: If there is an `else` branch, then the `else` branch will be taken if all other futures are not ready, then `tokio::select!` finishes.

In that example, the un-`poll`-ed future (`xxx_future`) holds lock. Its held lock will only release in two cases: 1. the future is polled and progresses, 2. the future is dropped. But that future is not `poll`-ed and also not dropped, thus lock cannot release.

Using `select!` correctly is not easy. For timeouts, it's recommended to just use  `tokio::timeout` (if you don't want cancellation, wrap inner future within `tokio::spawn`).

### Buffered stream issue

When using buffered stream, some futures in buffer may be temporarily un-`poll`-ed. This can cause weird delaying or deadlock.

See also:

- [Barbara battles buffered streams](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_battles_buffered_streams.html)
- [`for await` and the battle of buffered streams](https://tmandry.gitlab.io/blog/posts/for-await-buffered-streams/)
- [poll_progress](https://without.boats/blog/poll-progress/)
- [Never snooze a future](https://jacko.io/snooze.html)

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


## See also

[Why async Rust?](https://without.boats/blog/why-async-rust/)

[Async Rust can be a pleasure to work with (without `Send + Sync + 'static`)](https://emschwartz.me/async-rust-can-be-a-pleasure-to-work-with-without-send-sync-static/)

[Making Async Rust Reliable - Tyler Mandry](https://tmandry.gitlab.io/blog/posts/making-async-reliable/) 

[FuturesUnordered and the order of futures](https://without.boats/blog/futures-unordered/) 

[The bane of my existence: Supporting both async and sync code in Rust](https://nullderef.com/blog/rust-async-sync/)


