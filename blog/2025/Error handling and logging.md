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

## OOP exception

TODO

## Rust error handling

TODO

In Rust the error is often type-erased for convenience. The common solution is [anyhow](https://docs.rs/anyhow/latest/anyhow/). Without type erasure, you need to write a lot of error types and do conversions between.

Type erasuring error requires putting error into heap allocation. Different error types have different sizes. Turning them into heap allocation makes the memory layout be uniform (all become fat pointer). If error don't occur in hot path then it won't have visible performance cost.

`anyhow` requires error to be `Send + Sync + 'static`. The `'static` requires that if it contains a string, the string need to be copied (cannot reference other places). This makes it safe to pass error to outer scope, but also add copying cost.

Rust is not good at handling out-of-memory error.

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

## Logging

TODO
### Log spam

TODO
### Structural logging

TODO

