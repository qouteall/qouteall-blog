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

