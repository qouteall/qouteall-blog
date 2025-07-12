---
date: 2025-07-06
---


A lot of debate happen because same word has different meanings to different people. Some ambiguities related to programming:

<!-- truncate -->

- Encryption. Calculating hash code or signing is/isn't encryption.
- Linear regression is / isn't machine learning.
- AGI. Average-human-level / super-human AI.
- Pass-by-value. In some places, passing a reference is techincally called "pass-by-value". In some places, pass-by-value means pass object content instead of object reference.
- Compile. Turn source code into machine code / turn one kind of code into another kind of code (IR) / optimize code without changing code format (React compiler).
- Render. Generate image / generate HTML / generate video / generate other things.
- Parse. Parse contains / doesn't contain validation.
- Garbage collection. In some places, it means only tracing garbage collection. In some places, it also includes reference counting. GC includes epoch-based memory reclamation.
- In distributed system, "availability" means can process read requests / can process both read and write requests. [Let's Consign CAP to the Cabinet of Curiosities - Marc's Blog (brooker.co.za)](https://brooker.co.za/blog/2024/07/25/cap-again.html)
- Negative feedback loop. In some places, it means self-regulating process (like thermostat). In some places, it means self-reinforcing negative effect (such as self-reinforcing asset price drop in a financial crisis).
- Forward and backward in time. Sometimes "forward" is future-oriented, analogous to walking. Sometimes "forward" is past-oriented, when talking about history.
- MVC. There are two kinds of MVCs. One is for client GUI applications, where controller is the mediator between view and model. One is for server-side web applications, where the model accesses database, the view generates HTML and the controller calls the previous two and handle RESTful APIs. [MVC Isn’t MVC — Collin Donnell](https://collindonnell.com/mvc-isnt-mvc)
- Synchronization. In some places, specifying memory ordering and accessing Java volatile are called "synchronization". In some places these are not called synchronization.
- In English, synchronzied can mean "happen at the same time", which contradicts the fact that caller waiting for the service working. Asynchronous can mean "not happening at the same time", which contradicts the fact that the caller calling an asynchronous interface can run with the called service at the same time.
- "Low-level". Normally "low-level" usually means entry-level, junior-level. But in programming "low-level" can mean very deep things involving things like OS and hardware internal, which require high-level skill. We can use "deep-level", "infrastructure-level" instead of "low-level" to avoid misunderstanding.
- Predict. Normally "predict" means figuring out what happens in the future. But in AI, "predict" means estimating something, not necessarily the things in future. For example: "predict masked token", "predict noise".



Oxymoron naming:

- [Serverless servers](https://vercel.com/blog/serverless-servers-node-js-with-in-function-concurrency).
- Constant variable. (a constant. but treated as a special kind of variable in compiler)
- Unnamed namespaces. (C++ namespace without a name)
- Safe unsafe Rust code. (unsafe Rust code wrapped in a safe way)
- Asynchronous synchronization. (non-blocking replication)
- Lock-free deadlock. (deadlock can happen in message-passing systems, without any explicit lock)
- Transparent opacity. (an opacity value that's less than 1.0, making it transparent)
- Static animation. (the animation data is hard-coded)
- Unlogged log in. (the log in event is not written to record)


