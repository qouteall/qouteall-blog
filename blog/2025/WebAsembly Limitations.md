
Background:

- [WebAssembly](https://webassembly.org/) is an execution model and a code format.
- It's designed with performance concern. It can achieve higher performance than JS [^wasm_js_perf].
- It's designed with safety concern. Its execution is sandboxed. It can be run in browser. 
- It's close to native assembly (e.g. X86, ARM) but abstracts in a cross-platform way, so that many C/C++/Rust/etc. applications can be compiled to Wasm (but with limitations).
- Although its name has "Web", it's is not just for Web. It can be used outside of browser.
- Although its name has "Assembly", it has features (e.g. [GC](https://github.com/WebAssembly/gc)) that are in a higher abstraction layer than native assembly. WebAssembly can be seen as a programming language.
- In browsers, the Wasm runtime is JS runtime. For example, Chromium V8 executes both JS and Wasm.

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

Two vulnearbilities: [Spectre vulnerability](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)) and [Meltdown vulnerability](https://en.wikipedia.org/wiki/Meltdown_(security_vulnerability).

Background:

- CPU does speculative execution and branch prediction. CPU can execute many instructions in parallel. When CPU sees a branch, it tries to predict the branch and speculatively execute it. If CPU later find branch prediction to be false, the effects of speculative execution will be rolled back, but the side effects on cache won't rollback. 
- CPU also has memory access permission sytem. The memory access security check is deferred. When CPU sees memory access permission issue, it also rolls back side effects, but the side effects on cache won't rollback.
- CPU has a cache for accelerating memory access. Some parts of memory are put into cache. Accessing these memory can be done by accessing cache which is faster. 
- The cache size is limited. Accessing new memory can evict existing data in cache.
- Whether a content of memory is in cache can be tested by memory access time.
- Measuring memory access time requires a high precision timer. The web API of getting time (e.g. `performance.now()`) is not accurate enough. One more accurate way is to have another web worker keep incrementing an integer in `SharedArrayBuffer`.
- Spectre vulnearbility:    The memory access address contains content of `array1[x]`. The attacker measure 

Specture vulneability (Variant 1):

- It requires there exists code like `if (x < array1_size) { y = array2[array1[x] * 4096]; ... }` , and the attacker can call that code with specified `x`. These code are common in browsers.
- The attacker firstly call it with many in-bound `x` to make branch predictor to assume `x < array1_size` is likely true.
- Attacker then calls it using an out-of-bound `x`, then speculative execution will do memory access to `array2[array1[x] * 4096]`. It affects cache before CPU found branch prediction failure and rollback.

### Cannot block on main thread

## Cannot directly call Web APIs

## JS Interop

### Passing string


## Dunamically loading new code