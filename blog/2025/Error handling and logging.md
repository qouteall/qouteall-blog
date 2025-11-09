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

The bad design: fail-silent. 

## OOP exception

TODO

## Rust error handling

TODO
## Golang error handling

TODO
### IDE can "fix" the language

TODO

## Just error code

TODO
## Logging

TODO
### Log spam

TODO
### Structural logging

TODO

