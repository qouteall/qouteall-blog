---
date: 2025-11-09
tags:
  - Programming
unlisted: true
---

# Error Handling

<!-- truncate -->


## Error handling strategies

- Fail-fast.
  - Better for finding bugs.
  - Better for debugging as long as it keeps the context of error.
  - Easier to do correctly than cleaning up the invalid state.
  - Can be limited to one operation. No need to crash the whole server.
- Fail-safe.
  - Try to recover from error. Clean up the invalid state. It's hard to do correctly.
  - If one way fails, try another way.
  - Retry (with backoff, circuit breaker).
  - Higher availability to user.

The bad design: fail-silent. It hide problems. It may cause data corruption unnoticed.

TODO fail open fail close

## OOP exception

TODO

## Rust error handling

TODO

In Rust the error is often type-erased for convenience. The common solution is [anyhow](https://docs.rs/anyhow/latest/anyhow/). Without type erasure, you need to write a lot of error types and do conversions between.

Type erasuring error requires putting error into heap allocation. Different error types have different sizes. Turning them into heap allocation makes the memory layout be uniform (all become fat pointer). If error don't occur in hot path then it won't have visible performance cost.

`anyhow` requires error to be `Send + Sync + 'static`. The `'static` requires that if it contains a string, the string need to be copied (cannot reference other places). This makes it safe to pass error to outer scope, but also add copying cost.

Rust is not good at handling out-of-memory error. when OS over-commit is enabled, when memory is used up it errors when accessing memory, not when allocating. so doing correct error handling of OOM with over-commit is hard.

### Lock poisoning

lock poisoning [Adding locks disregarding poison · Issue #169 · rust-lang/libs-team](https://github.com/rust-lang/libs-team/issues/169)

when is poisoning useful: use lock to protect mutable data structure. if there is a panic within operation, the data structure may be in an invalid state. 

when poisoning is harmful: for a web server, when there is already database transaction that do proper rollback, so panicking doesn't make the data be invalid. in this case poisoning is not only useless but also harmful. the poisoned mutex in memory prevent new requests from accessing data, which hurts web server reliability.

sometimes panic when holding lock cause data corruption. but there are also many cases where panicking when holding lock doesn't cause data corruption.

tokio mutex cancel issue

## Golang error handling

TODO
### IDE can "fix" the language

TODO

## C error code


For C error code, the error code can be ignored. Sometimes the developer forget that error is possible and ignore the error.

> https://x.com/ErrataRob/status/2001872046515507372
> 
> One cause is the "truncated send()" bug. The Sockets send() function queues up the data in kernel buffers. When the kernel runs out of buffers, send() will return having queued only part of the data, returning the number of bytes queued.
> 
> It works like this according to the spec on all systems (Windows, Linux, macOS, etc.). You are supposed to check the return value, most don't.
> 
> Thus, you may see it send the data and then close the connection, not understanding that not all the data was actually stored in the kernel buffers.
> 
> It's a far more common bug than people realize because it'll pass through almost all automated testing. You have to heavily load a server to the point where kernel buffers are all in use before you can detect it, which is beyond the scale of typical Continuous Integration unit/regression tests.
> 
> It can continue right through production, because a lot of things correct it, such as automatically restarting a transaction if there's an intermittent failure.
> 
> You can see the problem easily looking at the outer SSL headers, by tracking record length fields and error messages.
> 
> But if it's the last record before terminating the connection, then you can easily diagnose it.
> 
> The problem described is confusing. It talks about half a request when the rest of the text sounds like half a response.

> https://man7.org/linux/man-pages/man2/close.2.html
> 
> A careful programmer will check the return value of **close**(), since it is quite possible that errors on a previous [write(2)](https://man7.org/linux/man-pages/man2/write.2.html) operation are reported only on the final **close**() that releases the open file description.  Failing to check the return value when closing a file may lead to _silent_ loss of data.  This can especially be observed with NFS and with disk quota.

How does C apps pass error data when it only returns an error code? One common pattern is to put error as global variable. Provide an API for getting last error. But global variables are more error-prone (e.g. data race risk). And if the library user forget to get last error the error is lost. Generally not recommended in modern applications.

## Zig error code

Zig error code is better than C as Zig doesn't allow implicitly ignoring error and proceed. 

But Zig doesn't allow attaching data with error. (In Rust and Golang, you can easily attach data to error.)

The error code itself only tells very little information. Getting detailed error information is important. There are 100 things that possibly correlate with the same error code. Not knowing error detail may waste developer hours debugging the error.

The common pattern is to pass in a diagnostics object, then write error data into diagnostics object:

```
pub fn someFunc(
    ...,
    diag: ?*Diagnostics,
) error{ SomeError }!ReturnType {
    // when error happens, write error data into diag and return error code
}
```

See also: [Error Codes for Control Flow](https://matklad.github.io/2025/11/06/error-codes-for-control-flow.html)

This is **less convenient** than Rust and Golang. In Rust and Golang, the error itself contains error data. But in Zig the two things are separated. 

Related:

> I just spent way longer than I should have to debugging an issue of my project's build not working on Windows given that all I had to work with from the zig compiler was an `error: AccessDenied` and the build command that failed. 
> 
> ......
> 
> While the obvious answer here is "The Zig compiler is a work in progress and eventually we will improve our error messages using the diagnostic pattern [proposed above](https://github.com/ziglang/zig/issues/2647#issuecomment-589829306)..." (or perhaps that this is some Windows specific issue, etc), **I think the fact that even the compiler can't consistently implement this pattern points to it perhaps being too manual/tedious/unergonomic/difficult to expect the Zig ecosystem at large to do the same**.
> 
> [Link](https://github.com/ziglang/zig/issues/2647#issuecomment-1444790576)

Sometimes having a separated diagnostics system is useful for showing user-friendly error message. Of course it requires extra efforts of developers.

Currently, Zig doesn't provide an unified interface for diagnostics information. So different libraries tend to have their own diagnostics types which are incompatible with each other. You cannot easily compose the diagnostics of different libraries. This makes it overall less convenient.

The problem is that just returning error code is easy in Zig, but providing diagnoistics information takes more efforts. There is [Principle of least effort](https://en.wikipedia.org/wiki/Principle_of_least_effort). If providing diagnostics is hard then very few libraries in ecosystem will do that.

If the purpose is just to help developer debugging, then logging should be enough. And it takes fewer efforts than maintaining a diagnostics object. But logging then faces log spam issue.


## Logging

TODO
### Log spam

TODO
### Structural logging

TODO


## Silent errors in AI

In deep learning and RL, bugs often don't give error messages. They just cause model to perform worse. 

Sometimes a bug in evaluation or RL reward make people wrongly think model has high capability.

Sometimes the dataset has many wrong data.

Sometimes even if there is an error model can adapt to it.

After turning data into matrix it's hard to visualize. So issues like flipping image is hardly noticed.

