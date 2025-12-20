---
date: 2025-11-09
tags:
  - Programming
unlisted: true
---

# Error Handling and Logging

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

lock poisoning [Adding locks disregarding poison · Issue #169 · rust-lang/libs-team](https://github.com/rust-lang/libs-team/issues/169)

when is poisoning useful: use lock to protect mutable data structure. if there is a panic within operation, the data structure may be in an invalid state. 

when poisoning is harmful: for a web server, when there is already database transaction that do proper rollback, so panicking doesn't make the data be invalid. in this case poisoning is not only useless but also harmful. the poisoned mutex in memory prevent new requests from accessing data, which hurts web server reliability.

sometimes panic when holding lock cause data corruption. but there are also many cases where panicking when holding lock doesn't cause data corruption.

panic propagation doesn't work well with web server. panic propagation make requests that use poisoned mutext keep failing (without restart). if during panic unwind the drop uses mutex and use `.unwrap` it may second-panic which crashes whole process.

tokio mutex cancel issue

## Golang error handling

TODO
### IDE can "fix" the language

TODO

## Just error code

TODO

C and Zig commonly use error code. 

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

Zig error code is better than C as Zig doesn't allow implicitly ignoring error and proceed. 

But Zig doesn't allow attaching data with error so error data can only be passed via side channel (in Zig there is no uniform way of passing error data so each library have their own way that don't work together.)

## Logging

TODO
### Log spam

TODO
### Structural logging

TODO

