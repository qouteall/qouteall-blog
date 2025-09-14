
# WebAssembly Limitations

<!-- truncate -->

Background:

- [WebAssembly](https://webassembly.org/) is an execution model and a code format.
- It's designed with performance concern. It can achieve higher performance than JS [^wasm_js_perf].
- It's designed with safety concern. Its execution is sandboxed.
- It can be run in browser. 
- It's close to native assembly (e.g. X86, ARM) but abstracts in a cross-platform way, so that many C/C++/Rust/etc. applications can be compiled to Wasm (but with limitations).
- Although its name has "Web", it's is not just for Web. It can be used outside of browser.
- Although its name has "Assembly", it has features (e.g. [GC](https://github.com/WebAssembly/gc)) that are in a higher abstraction layer than native assembly. WebAssembly can be seen as a programming language.
- In browsers, it's the same engine that runs JS and Wasm. For example, Chromium V8 executes both JS and Wasm.

[^wasm_js_perf]: WebAssembly is not always faster than JS, depending on how optimization efforts are put in, and browser's limitations. But WebAssembly has higher potential for computation performance than JS. JS has a lot of flexibility. Flexibility costs performance. JS runtime often use runtime statistics to find unused flexibility and optimize accordingly. But statistics cannot be really sure so JS runtime still have to "prepare" for flexibility. The runtime statistics and "prepare for flexibility" all costs performance, in a way that cannot be optimized without changing code format.


## Wasm runtime data

The data that Wasm program works on:

- Runtime-managed stack. It has local variables, function arguments, function pointers, etc. It's managed by the runtime. It's not in linear memory.
- Linear memory. 
  - A linear memory is an array of bytes. Can be read/written by address (address is the index in array). 
  - Wasm supports one module to have multiple linear memories.
  - A linear memory's size can grow. But currently a linear memory's size cannot shrink.
  - A linear memory can be shared by multiple Wasm instances, see multi-threading section below.
- Table. Each table is a (growable) array that can hold:
  - Function references.
  - Extern value references. Extern value can be JS value or other things, depending on environment.
  - Exception references.
  - GC value references.
- Heap. Holds GC values. Explained later.
- Globals. A global can hold a number (`i32`, `i64`, `f32`, `f64`), an `i128` or a reference (including function reference, GC value reference, extern value reference, etc.). The globals are not in linear memory.

The linear memory doesn't hold these things:

- Linear memory doesn't hold the stack. The stack managed by runtime and cannot be read/written by address.
- The linear memory doesn't hold function references. Unlike function pointers in C, Wasm function references cannot be converted to and from integers. This design can improve safety. A function reference can be on stack or on table or in global, and can be called by special instructions [^function_call_instructions].

[^function_call_instructions]: `call_ref` calls a function reference on stack. `call_indirect` calls a function reference in a table in an index. `return_call_ref`, `return_call_indirect` are for tail call.

## Stack is not in linear memory

Normally program runs with a stack. For native programs, the stack holds:

- Local variables. (not all local variables are on stack. some are in registers)
- Call arguments. (similar to the above, some are in registers)
- Return code address. It's the machine code address to jump to when function returns. (Function can be inlined, machine code can be optimized, so this don't always correspond to code.)
- Other things. (e.g. C# `stackalloc`, Golang `defer` metadata) 

In Wasm, the main stack is managed by Wasm runtime. The main stack is not in lineary memory, and cannot be read/written by address.

It has benefits:

- It avoids security issues related to control flow hijacking. A native application's stack is in memory. An out-of-bound write can change the return function address on stack, causing it to execute wrong code. Mitigations such as [data execution prevention](https://en.wikipedia.org/wiki/Executable_space_protection) (DEP) and [stack smashing protection](https://en.wikipedia.org/wiki/Buffer_overflow_protection#Random_canaries) (SSP) are not needed in Wasm. [See also](https://webassembly.org/docs/security/)
- It allows the runtime to optimize stack layout without changing program behavior.

But it also have downsides:

- Some local variables need to be taken address to. They need to be in linar memory.
- GC needs to scan the references (pointers) on stack. If the Wasm app use application-managed GC (not Wasm built-in GC) (for reasons explained below), then the on-stack references (pointer) need to be "spilled" to linear memory.
- Stack switching cannot be done. Golang use stack switching for goroutine scheduling. There is a [proposal](https://github.com/WebAssembly/stack-switching) for adding this functionality.
- Dynamic stack resizing cannot be done. Golang does dynamic stack resizing so that new goroutines can be initialized with small stacks, reducing memory usage.

The common solution is to have a **shadow stack** that's in linear memory. That stack is managed by Wasm code.

Summarize 2 different stacks:

- The main execution stack, that holds local variable, call arguments, function pointers, and possibly operands (in wasm stack machine). It's managed by Wasm runtime and not in linear memory. It cannot be directly read and written by Wasm code.
- The shadow stack. It's in linear memory. Holds the local variables that need to be in linear memory. Managed by Wasm code, not Wasm runtime.

## Memory deallocation

The Wasm linear memory can be seen as a large array of bytes. Address in linear memory is the index into the array.

Instruction `memory.grow` can grow a linear memory. 

However, there is no way to shrink the linear memory. There is no way to return allocated memory back to Wasm runtime then back to OS. 

The allocator (in Wasm application) can free regions of linear memory for future use. In native applications, the memory block freed by allocator are often returned to OS. But in Wasm, the freed memory regions cannot be returned to OS.

Mobile platforms (iOS, Android, etc.) often kill background process that has large memory usage, so not returning memory to OS is an important issue. See also: [Wasm needs a better memory management story](https://github.com/WebAssembly/design/issues/1397). 

There is a [memory control propsal](https://github.com/WebAssembly/memory-control) that addresses this issue.

## Wasm GC

When compiling non-GC languages (e.g. C/C++/Rust/Zig) to Wasm, they use the linear memory and implement the allocator in Wasm code.

For GC langauges (e.g. Java/C#/Python/Golang), they need to make GC work in Wasm. There are two solutions:

- Still use linear memory to hold data. Implement GC in Wasm code.
- Use Wasm's built-in GC functionality.

The first solution, manually implementing GC encounters difficulties:

- GC requires scanning GC roots (pointers). Some GC roots are on stack. But the Wasm main stack is not in linear memory and cannot be read by address. One solution is to "spill" the pointers to the shadow stack in linear memory. Having the shadow stack increases binary size and costs runtime performance.
- Multi-threaded GC often need to pause the execution to scan the stack correctly. In native applications, it's often done using safepoint mechanism [^safepoint_mechanism]. It also increases binary size and costs runtime performance.
- Multi-threded GC often use store barrier or load barrier to ensure scanning correctness. It also increases binary size and costs runtime performance.
- Cannot collect a cycle where a JS object and an in-Wasm object references each other.

[^safepoint_mechanism]: Safepoint mechanism allows a thread to pause at specific points. It can force the paused thread to expose all local variables on stack. When a thread is running, a local variable may be in register that cannot be scanned by another thread. And scanning a running thread's stack is not reliable due to memory order issues and race conditions. One way to implement safepoint is to have a global safepoint flag. The code frequently reads the safepoint flag and pause if flag is true. There exists optimizations such as using OS page fault handler.

What about using Wasm's built-in GC functionality? It requires mapping the data structure to Wasm GC data structure. Wasm's GC data structure allows Java-like class (with object header), Java-like prefix subtyping, and Java-like arrays. But it doesn't support:

- Use fat pointer to avoid object header. (Golang does it)
- Add custom fields at the head of an array object. (C# supports it)
- Compact sum type memory layout.
- Interior pointer. (Golang supports interior pointer)
- Weak reference.
- Finalizer (the code that run when an object is collected by GC).

## Multi-threading

The execution model of web code:

- The main thread runs in an event loop, with an event queue.
- Each calling to JS adds one event to queue.
- The event loop executes all events in queue, until queue is empty, then browser controls the main thread, until new event arrives.
- The web page rendering and interaction is blocked by main thread JS code running. It's not recommended to make main thread JS code block for long time.
- There are web workers. Each web worker also has its own event loop and event queue. Each web worker is single-threaded.
- Web workers don't share memory (exception: `SharedArrayBuffer`). JS values can be sent to another web worker, but it's deep-copied after being sent.

WebAssembly multithreading relies on web workers and `SharedArrayBuffer`.

### Security issue of `SharedArrayBuffer`

[Spectre vulnerability](https://meltdownattack.com/) is a vulnearbility that allows JS code running in browser to read browser memory. Exploiting it requires accurately measuring memory access latency to test whether a piece of data is in cache. 

Modern browsers reduced `performance.now()`'s precision to make it not usable for exploit. But there is another way of accurately measuring (relative) latency: multi-threaded counter timer. One thread (web worker) keeps incrementing a counter in `SharedArrayBuffer`. Another thread can read that counter, treating it as "time". Subtracting two "time" gets accurate relative latency.

[Spectre vulneability explanation below](#spectre-vulnerability-explanation)

The solution to that security issue is [**cross-origin isolation**](https://web.dev/articles/cross-origin-isolation-guide). Cross-origin isolation make the browser to use different processes for different websites. One website exploiting Spectre vulnearbility can only read the memory in the process of their website, not other websites.

Cross-origin isolation can be enabled by the HTML loading response having these headers:

- `Cross-Origin-Opener-Policy` be `same-origin`
- `Cross-Origin-Embedder-Policy` be `require-corp` or `credentialless`. 
  - If it's `require-corp`, all resources loaded from other websites (origins) must have response header contain `Cross-Origin-Resource-Policy: cross-origin` (or CORS headers, like `Access-Control-Allow-Origin: *`). Otherwise resource loading will fail.
  - If it's `credentialless`

After cross-origin isolation is enabled:

- For the resources loaded from other websites (origins), the resource loading response header must have 
- The `iframe` that the website embeds also need to be cross-origin isolated.

### Cannot block on main thread

## Cannot directly call Web APIs

## JS Interop

### Passing string


## Dunamically loading new code


## Appendix

### Spectre vulnerability explanation

Background:

- CPU has a cache for accelerating memory access. Some parts of memory are put into cache. Accessing these memory can be done by accessing cache, which is faster. 
- The cache size is limited. Accessing new memory can evict existing data in cache.
- Whether a content of memory is in cache can be tested by memory access latency.
- CPU does speculative execution and branch prediction. CPU tries to execute as many as possible instructions in parallel. When CPU sees a branch (e.g. `if`), it tries to predict the branch and speculatively execute code in branch. 
- If CPU later find branch prediction to be wrong, the effects of speculative execution (e.g. written registers, written memory) will be rolled back. However, memory access leads side effect on cache, and that side effect won't be cancelled by rollback. 
- The branch predictor relies on statistical data, so it can be "trained". If one branch keeps go to first path for many times, the branch predictor will predict it will always go to the first path.

Specture vulneability (Variant 1) core exploit JS code ([see also](https://spectreattack.com/spectre.pdf)):

```js
...
if (index < simpleByteArray.length) {
    index = simpleByteArray[index | 0];
    index = (((index * 4096)|0) & (32*1024*1024-1))|0;
    localJunk ˆ= probeTable[index|0]|0;
}
...
```

The `|0` is for converting value to 32-bit integer, helping JS runtime to optimize it into integer operation (JS is dynamic, without that the JITed code may do other things). The `localJunk` is to prevent these read opearations from being optimized out.

- The attacker firstly execute that code many times with in-bound `index` to "train" branch predictor. 
- Then the attacker access many other different memory locations to invalidate the cache.
- Then attacker executes that code using a specific out-of-bound `index`:
  - CPU speculatively reads `simpleByteArray[index]`. It's out-of-bound. That result is the secret in browser process's memory.
  - Then CPU speculatively reads `probeTable`, using an index that's computed from that secret.
  - One specific memory region in `probleTable` will be loaded into cache. Accessing that region is faster.
  - CPU found that branch prediction is wrong and rolls back, but doesn't rollback side effect on cache.
- The attacker measures memory read latency in `probeTable`. Which place access faster correspond to the value of secret.
- To accurately measure memory access latency, `performance.now()` is not accurate enough. It need to use a multi-threaded counter timer: One thread (web worker) keeps increasing a shared counter in a loop. The attacking thread reads that counter to get "time". The cross-thread counter sharing requires `SharedArrayBuffer`. Although it cannot measure time in standard units (e.g. nanosecond), it's accurate enough to distinguish latency difference between fast cache access and slow RAM access.

Related: Another vulnerability related to cache side channel: [GoFetch](https://gofetch.fail/). It exploits Apple processors' cache prefetching functionality.

