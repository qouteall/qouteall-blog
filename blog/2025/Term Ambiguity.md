---
date: 2025-07-06
tags:
  - Programming
---
# Term Ambiguity

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
- API. Restful APIs / functions / other forms.
- Synchronization. In some places, specifying memory ordering and accessing Java volatile are called "synchronization". In some places these are not called synchronization.
- In English, synchronzied can mean "happen at the same time", which contradicts the fact that caller waiting for the service working. Asynchronous can mean "not happening at the same time", which contradicts the fact that the caller calling an asynchronous interface can run with the called service at the same time.
- "Low-level". Normally "low-level" usually means entry-level, junior-level. But in programming "low-level" can mean very deep things involving things like OS and hardware internal, which require high-level skill. We can use "deep-level", "infrastructure-level" instead of "low-level" to avoid misunderstanding.
- Predict. Normally "predict" means figuring out what happens in the future. But in AI, "predict" means estimating something, not necessarily the things in future. For example: "predict masked token", "predict noise".
- KB, MB, GB. 
  - Most commonly, 1 KB = 1024 bytes, 1MB = 1024 KB, 1GB = 1024 MB. (Formally they should be written as KiB, MiB, GiB.)
  - In disk manufactuers' descriptions, 1 KB = 1000 bytes, 1MB = 1000 KB, 1GB = 1000 MB. 
  - In networking speed, 1 Kbps = 1000 bits per second, 1Mbps = 1000 Kbps, 1Gbps = 1000 Mbps.
- Verbal. Sometimes mean spoken words. Sometimes includes both written text and spoken words.
- "Last" can mean "previous" or "final".
- Immutable. There are different kinds of "immutable":
  - The referenced object is immutable, and the reference is also immutable.
  - The referenced object is immutable, but the reference itself is mutable.
  - The referenced object is mutable, but the reference itself is immutable.
  - Read-only is not necessarily immutable.
- Character. A character in GUI is a grapheme cluster. Sometimes it mean a code point. In C, a `char` is a byte. In Java a `char` is two bytes.
- Artificial nerual network are "Black Box". All the matrix computations and weights involved in inference and training are white-box. The "Black Box" here means the mechanism of why it produce specific output is not clear. Although human can view the weight numbers, it's hard to understand how these weights correspond to what "thinking" and "decision making".
- RAG (retrieval augmented generation). Sometimes it must involve vector database. Sometimes it involves all kinds of information retrieval methods.
- Unsafe/safe. "Unsafe" has these nuanced intepretations: 1. it can potentially cause problems 2. it will definitely cause problems. 3. it may be safe in short run but will eventually cause problems if you keep doing it 4. it will only cause problems if you use it wrongly [^rust_unsafe]
- Routing. Router determine which interface to relay packet to. / Determine which web page based on URL (and other things). / Determine which Restful API by URL (and other things).

[^rust_unsafe]: The meaning of `unsafe` in Rust is close to the 4th interpretation. `unsafe` Rust code can be safe. But some people understand "unsafe" as 2nd interpretation. [See also](https://github.com/rust-lang/rfcs/pull/117)

