---
date: 2025-09-23
tags:
  - Programming
---

# WebAssembly Limitations

<!-- truncate -->

Background:

- [WebAssembly](https://webassembly.org/) is an execution model and a code format.
- It's designed with performance concern. It can achieve higher performance than JS [^wasm_js_perf].
- It's designed with safety concern. Its execution is sandboxed.
- It can be run in browser. 
- It's close to native assembly (e.g. X86, ARM) but abstracts in a cross-platform way, so that many C/C++/Rust/etc. applications can be compiled to Wasm (but with limitations).
- Although its name has "Web", it's is not just for Web. It can be used outside of browser.
- Although its name has "Assembly", it has features (e.g. [GC](https://github.com/WebAssembly/gc)) that are in a higher abstraction layer than native assembly, similar to JVM bytecode.
- In browsers, it's the same engine that runs JS and Wasm. Chromium V8 executes both JS and Wasm. Wasm GC use the same GC as JS.

[^wasm_js_perf]: WebAssembly is not always faster than JS, depending on how much optimization efforts are put in, and browser's limitations. But WebAssembly has higher potential of performance than JS. JS has a lot of flexibility. Flexibility costs performance. JS runtime often use runtime statistics to find unused flexibility and optimize accordingly. But statistics cannot be really sure so JS runtime still have to "prepare" for flexibility. The runtime statistics and "prepare for flexibility" all costs performance, in a way that cannot be optimized without changing code format.

This article focuses on in-browser Wasm.

## Wasm runtime data

The data that Wasm program works on:

- Runtime-managed stack. It has local variables, function arguments, return code addresses, etc. It's managed by the runtime. It's not in linear memory.
- Linear memory. 
  - A linear memory is an array of bytes. Can be read/written by address (address can be seen as index in array). 
  - Wasm supports having multiple linear memories.
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
- The linear memory doesn't hold function references. Unlike function pointers in C, Wasm function references cannot be converted to and from integers. This design can improve safety. A function reference can be on stack or on table or in global, and can be called by special instructions [^function_call_instructions]. Function pointer becomes integer index corresponding to a function reference in table.

[^function_call_instructions]: `call_ref` calls a function reference on stack. `call_indirect` calls a function reference in a table in an index. `return_call_ref`, `return_call_indirect` are for tail call.

## Stack is not in linear memory

Normally program runs with a stack. For native programs, the stack holds:

- Local variables and call arguments. (not all of them are on stack. some are in registers)
- Return code address. It's the machine code address to jump to when function returns. (Function can be inlined, machine code can be optimized, so this don't always correspond to code.)
- Other things. (e.g. C# `stackalloc`, Golang `defer` metadata) 

In Wasm, the main stack is managed by Wasm runtime. The main stack is not in lineary memory, and cannot be read/written by address.

It has benefits:

- It avoids security issues related to control flow hijacking. A native application's stack is in memory, so out-of-bound write can change the return code address on stack, causing it to execute wrong code. There are protections such as [data execution prevention](https://en.wikipedia.org/wiki/Executable_space_protection) (DEP) and [stack canary](https://en.wikipedia.org/wiki/Buffer_overflow_protection#Random_canaries) and [address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization) (ASLR). These are not needed in Wasm. [See also](https://webassembly.org/docs/security/)
- It allows the runtime to optimize stack layout without changing program behavior.

But it also have downsides:

- Some local variables need to be taken address to. They need to be in linar memory. For example:

```c
void f() {
    int localVariable = 0;
    int* ptr = &localVariable;
    ...
}
```

  The `localVariable` is taken address to, so it must be in linear memory, not Wasm execution stack (unless the compiler can optimize out the pointer).

- GC needs to scan the references (pointers) on stack. If the Wasm app use application-managed GC (not Wasm built-in GC) (for reasons explained below), then the on-stack references (pointer) need to be "spilled" to linear memory.
- Stack switching cannot be done. Golang use stack switching for goroutine scheduling. There is a [proposal](https://github.com/WebAssembly/stack-switching) for adding this functionality.
- Dynamic stack resizing cannot be done. Golang does dynamic stack resizing so that new goroutines can be initialized with small stacks, reducing memory usage.

The common solution is to have a **shadow stack** that's in linear memory. That stack is managed by Wasm code. (Sometimes shadow stack is called aux stack.)

Summarize 2 different stacks:

- The main execution stack, that holds local variable, call arguments, function pointers, and possibly operands (in wasm stack machine). It's managed by Wasm runtime and not in linear memory. It cannot be directly read and written by Wasm code.
- The shadow stack. It's in linear memory. Holds the local variables that need to be in linear memory. Managed by Wasm code, not Wasm runtime.

There is a [stack switching proposal](https://github.com/WebAssembly/stack-switching) that aim to allow Wasm to do stack switching. This make it easier to implement lightweight thread (virtual thread, goroutine, etc.), without transforming the code and add many branches.

Using shadow stack involves issue of reentrancy explained below.

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
- Multi-threaded GC often use store barrier or load barrier to ensure scanning correctness. It also increases binary size and costs runtime performance.
- Cannot collect a cycle where a JS object and an in-Wasm object references each other.

[^safepoint_mechanism]: Safepoint mechanism allows a thread to cooporatively pause at specific points. Scanning a running thread's stack is not reliable, due to memory order issues and race conditions, and some pointers may be in register, not stack. If the thread is suspended using OS functionality, some local variable may be in register, and it's hard to tell whether data in register is pointer or other data (treating integer as pointer may cause memory safety issue or memory leak). If a thread is coorporatively paused in specific places, the references can be reliably scanned. One way to implement safepoint is to have a global safepoint flag. The code frequently reads the safepoint flag and pause if flag is true. There exists optimizations such as using OS page fault signal handler.

What about using Wasm's built-in GC functionality? It requires mapping the data structure to Wasm GC data structure. Wasm's GC data structure allows Java-like class (with object header), Java-like prefix subtyping, and Java-like arrays. 

The important memory management features that it doesn't support:

- No weak reference.
- No finalizer (the code that runs when an object is collected by GC).
- No interior pointer. (Golang supports interior pointer)
- **GC values cannot be shared across threads**

It doesn't support some memory layout optimizations:

- No array of struct type.
- No per-object locking (in Java and C# every object can be a lock)
- Cannot use fat pointer to avoid object header. (Golang does it)
- Cannot add custom fields at the head of an array object. (C# supports it)
- Don't have compact sum type memory layout.

See also: [C# Wasm GC issue](https://github.com/dotnet/runtime/issues/94420), [Golang Wasm GC issue](https://github.com/golang/go/issues/63904)

## Multi-threading

Web code runs in event loop:

- The main thread runs in an event loop, with an event queue.
- Each time browser calls JS code (e.g. event handling), it adds an event to queue.
- If JS code awaits on an unresolved promise, the event handling finishes. When that promise resolves, a new event is added into queue.
- The event loop executes all events in queue, until queue is empty. It waits until new event arrives.
- The web page rendering and interaction is blocked by main thread JS code and Wasm code running. It's not recommended to make main thread keep executing JS/Wasm for long time.
- There are web workers that can run in parallel. Each web worker also has its own event loop and event queue. Each web worker is single-threaded.
- Web workers don't share memory (except `SharedArrayBuffer`). JS values sent to another web worker are deep-copied. Sending an `ArrayBuffer` across thread will make `ArrayBuffer` to detach with its binary data.
- Only the main thread can do DOM operations.

WebAssembly multithreading relies on web workers and `SharedArrayBuffer`.

### Security issue of `SharedArrayBuffer`

[Spectre vulnerability](https://meltdownattack.com/) is a vulnearbility that allows JS code running in browser to read browser memory. Exploiting it requires accurately measuring memory access latency to test whether a region of memory is in cache. 

Modern browsers reduced `performance.now()`'s precision to make it not usable for exploit. But there is another way of accurately measuring (relative) latency: multi-threaded counter timer. One thread (web worker) keeps incrementing a counter in `SharedArrayBuffer`. Another thread can read that counter, treating it as "time". Subtracting two "time" gets accurate relative latency.

[Spectre vulneability explanation below](#spectre-vulnerability-explanation)

### Cross-origin isolation

The solution to that security issue is [**cross-origin isolation**](https://web.dev/articles/cross-origin-isolation-guide). Cross-origin isolation make the browser to use different processes for different websites. One website exploiting Spectre vulnearbility can only read the memory in the process of their website, not other websites.

Cross-origin isolation can be enabled by the HTML loading response having these headers:

- `Cross-Origin-Opener-Policy` be `same-origin`. [See also](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cross-Origin-Opener-Policy)
- `Cross-Origin-Embedder-Policy` be `require-corp` or `credentialless`. [See also](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cross-Origin-Embedder-Policy)
  - If it's `require-corp`, all resources loaded from other websites (origins) must have response header contain `Cross-Origin-Resource-Policy: cross-origin` (or differently if in CORS mode).
  - If it's `credentialless`, requests sent to other websites won't contain credentials like cookies.

### Cannot block on main thread

The threads proposal adds `memory.atomic.wait32`, `memory.atomic.wait64` instructions for suspending a thread, which can be used for implement locks (and conditional variables, etc.). [See also](https://github.com/WebAssembly/threads/blob/main/proposals/threads/Overview.md)

However, the main thread cannot be suspended by these instructions. This was due to some concerns about web page responsiveness.

[Related 1](https://github.com/WebAssembly/threads/issues/106) [Related 2](https://github.com/WebAssembly/threads/issues/177) [Related 3](https://github.com/WebAssembly/threads/issues/174)

This restriction makes porting native multi-threaded code to Wasm harder. For example, locking in web worker can use normal locking, but locking in main thread must use spin-lock. Spin-locking for long time costs performance.

The main thread can be blocked using [JS Promise integration](https://github.com/WebAssembly/js-promise-integration). That blocking will allow other code (JS code and Wasm code) to execute when blocking. This is called **reentrancy**. 

Wasm applications often use **shadow stack**. It's a stack that's in linear memory, managed by Wasm app rather than Wasm runtime. Shadow stack must be properly switched when Wasm code suspends and resumes using JS Promise integration. Otherwise, the shadow stack parts of different execution can be mixed and messed up. Other things may also break under reentrancy and need to be taken care of.

What's more, if the canvas drawing code suspends using JS Promise integration, the half-drawn canvas will be presented in web page. This can be workarounded by using [offscreen canvas](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas), drawn in web worker. 

### Recreating Wasm instance

Multi-threading in Web relies on web workers. Currently there is no way to directly launch a Wasm thread in browser.

Launching a multi-threaded Wasm application is done by passing shared `WebAssembly.Memory` (that contains a `SharedArrayBuffer`) to another web worker. That web worker need to **separately create a new Wasm instance**, using the same `WebAssembly.Memory` object (and `WebAssembly.Module` object.

The **Wasm globals are thread-local** (not actually global). Mutate a mutable Wasm global in one thread don't affect other threads. Mutable globals variables need to be placed in linear memory.

Another important limitation: **The Wasm tables cannot be shared**.

That creates trouble when **loading new Wasm code during running** (dynamic linking). To make existing code call new function, you need indirect call via function reference in table. However, tables cannot be shared across Wasm instances in different web workers.

The current workaround is to notify the web workers to make them proactively load the new code and put new function references to table. One simple way is to send a message to web worker. But that doesn't work when web worker's Wasm code is still running. For that case, some other mechanisms (that costs performance) need to be used.

> While load-time dynamic linking works without any complications, runtime dynamic linking via `dlopen`/`dlsym` can require some extra consideration. The reason for this is that keeping the indirection function pointer table in sync between threads has to be done by emscripten library code. Each time a new library is loaded or a new symbol is requested via `dlsym`, table slots can be added and these changes need to be mirrored on every thread in the process.
> 
> Changes to the table are protected by a mutex, and before any thread returns from `dlopen` or `dlsym` it will wait until all other threads are sync. In order to make this synchronization as seamless as possible, we hook into the low level primitives of *emscripten_futex_wait* and *emscripten_yield*.
> 
> [Dynamic Linking — Emscripten](https://emscripten.org/docs/compiling/Dynamic-Linking.html)

There is [shared-everything threads proposal](https://github.com/WebAssembly/shared-everything-threads) that aim to fix that.

If all web workers utitlize browser's event loop and don't block for long time in each execution, then they can coorporatively load new Wasm code by processing web worker message, without much delay.

## Wasm-JS passing

Numbers (`i32`, `i64`, `f32`, `f64`) can be directly passed between JS and Wasm (`i64` maps to `BigInt` in JS, other 3 maps to `number`).

Passing a JS string to Wasm requires:

- transcode (e.g. passing to Rust need to convert WTF-16 to UTF-8),
- allocate memory in Wasm linear memory,
- copy transcoded string into Wasm linear memory,
- pass address and length into Wasm code,
- Wasm code need to care about deallocating the string.

Similarily passing a string in Wasm linear memory to JS is also not easy. 

Passing strings between Wasm and JS can be a performance bottleneck. If your application involve frequent Wasm-JS data passing, then replacing JS by Wasm may actually reduce performance.

Modern Wasm/JS runtime (including V8) can JIT and inline the cross calling between Wasm and JS. But the copying cost still cannot be optimized out.

The [Wasm Component Model](https://component-model.bytecodealliance.org/introduction.html) aim to solve this. It allows passing higher-level types such as string, record (struct), list, enum in interface. But different components cannot share memory, and the passed data need to be copied.

There are [Wasm-JS string builtins](https://webassembly.github.io/spec/js-api/index.html#builtins-js-string) that aim to reduce the cost of string passing between Wasm and JS.

## Cannot directly call Web APIs

Wasm code cannot directly call Web APIs. Web APIs must be called via JS glue code.

Although all Web's JS APIs have [Web IDL](https://webidl.spec.whatwg.org/) specifications, it involves GCed-objects, iterators and async iterators (e.g. `fetch()` returns `Response`. Its `body` field is `ReadableStream`). These GC-related things and async-related things cannot easily adapt to Wasm code using linear memory. It's hard to design a specification of turning Web IDL interfaces to Wasm interfaces.

There was [Web IDL Bindings Proposal](https://github.com/WebAssembly/interface-types/blob/1c46f9fe30143867545c9747fa8a94b72e5d9737/proposals/webidl-bindings/Explainer.md) but superseded by Component Model proposal.

Currently Wasm cannot be run in browser without JS code that bootstraps Wasm.

## Memory64 performance

The original version of Wasm only supports 32-bit address and up to 4GiB linear memory. 

In Wasm, a linear memory has a finite size. Accessing an address out of size need to trigger a [trap](https://webassembly.github.io/spec/core/intro/overview.html) that aborts execution. Normally, to implement that range checking, the runtime need to insert branches for each linear memory access (like `if (address >= memorySize) {trap();}`). 

But Wasm runtimes have an optimization: map the 4GB linear memory to a virtual memory space. The out-of-range pages are not allocated from OS, so accessing them cause error from OS. Wasm runtime can use signal handling to handle these error. No range checking branch needed.

That optimization doesn't work when supporting 64-bit address. There is no enough virtual address space to hold Wasm linear memory. So the branches of range checking still need to be inserted for every linear memory access. This costs performance.

See also: [Is Memory64 actually worth using?](https://spidermonkey.dev/blog/2025/01/15/is-memory64-actually-worth-using.html)

## Debugging Wasm running in Chrome

Firstly, the `.wasm` file need to have [DWARF](https://dwarfstd.org/) debug information in custom section.

There is a [C/C++ DevTools Support (DWARF) plugin](https://chromewebstore.google.com/detail/cc++-devtools-support-dwa/pdcpmagijalfljmkmjngeonclgbbannb)  ([Source code](https://github.com/ChromeDevTools/devtools-frontend/tree/main/extensions/cxx_debugging)). That plugin is designed to work with C/C++. When using it on Rust, breakpoints and inspecting integer local variables work, but other functionalities (inspecting string, inspecting global, evaluate expression, etc.) are not supported.

VSCode can debug Wasm running in Chrome, using [vscode-js-debug plugin](https://github.com/microsoft/vscode-js-debug). [Documentation](https://code.visualstudio.com/docs/nodejs/browser-debugging), [Documentation](https://code.visualstudio.com/docs/nodejs/nodejs-debugging#_debugging-webassembly). It allows inspecting integer local variable. But the local variable view doesn't show string content. Can only see string content by inspecting linear memory. The debug console expression evaluation doesn't allow call functions.

(It also requires VSCode [WebAssembly DWARF Debugging](https://marketplace.visualstudio.com/items?itemName=ms-vscode.wasm-dwarf-debugging) extension. Currently (2025 Sept) that extension doesn't exist in Cursor.)

Chromium [debugging API](https://chromedevtools.github.io/devtools-protocol/tot/Debugger/).

## Hot reload that keeps execution state

It's easy to load new Wasm code in browser, without keeping Wasm execution state. 

During development, hot reloading Wasm code while keeping execution state can improve iteration speed. Without hot reload, the developer changed the code need to re-enter the same application state to test changes.

Java and C# supports hot reloading (Java only supports changing function body or inlined constants), with the help of their runtime. Just being able to reload function body is already useful. [See also](https://loglog.games/blog/leaving-rust-gamedev/#hot-reloading-is-more-important-for-iteration-speed-than-people-give-it-credit-for)

In native applications, hot reload often requires advanced machine code instumentation. Because it need to be done while application is running. Keeping the execution state and adapting it to new code is hard.

Web code runs in event loop. After one cycle of event loop, there is no Wasm or JS code running. Now the only execution state become linear memory, globals and tables, etc. The stack and code execution position doesn't exist and don't need to be kept. It's theoretically possible to do hot swap during that, by creating a new Wasm module using new code, creating a new Wasm instance using new module, with the old linear memory, globals, tables etc. Related details:

- The initialization code that resets some global state need to be skipped after hotswap.
- The layout of data section changes. Data sections are loaded into linear memory during startup. The original pointers (including strings) pointing into data section need to still work. One solution is to place new data section in new places, which wastes some memory. As that hot swapping is intended to be only used in development, wasting some memory is fine. The address of global constants need to be relocated (can reuse existing dynamic linking functionality).
- If the memory layout in linear memory changes, it won't work, unless there is no such data in linear memory. This also includes Rust futures. Async functions cannot be easily be hot reloaded.
- All references to Wasm-exported functions in JS need to be replaced with new ones, otherwise it will still execute old Wasm instance.
- It doesn't work with [JS Promise integration](https://github.com/WebAssembly/js-promise-integration), as the Wasm runtime manages the stack for that.
- The Wasm globals and tables need to be exported, otherwise JS cannot access them.
- The function table need to be new, but the function in each index need to correspond to the function in same index in the old table, otherwise function indexes on linear memory will break.

That hot realod is not yet implemented.

## Appendix

### Spectre vulnerability explanation

Background:

- CPU has a cache for accelerating memory access. Some parts of memory are put into cache. Accessing these memory can be done by accessing cache, which is faster. 
- The cache size is limited. Accessing new memory can evict existing data in cache, and put newly accessed data into cache.
- Whether a content of memory is in cache can be tested by memory access latency.
- CPU does speculative execution and branch prediction. CPU tries to execute as many as possible instructions in parallel. When CPU sees a branch (e.g. `if`), it tries to predict the branch and speculatively execute code in branch. 
- If CPU later find branch prediction to be wrong, the effects of speculative execution (e.g. written registers, written memory) will be rolled back. However, memory access leaves side effect on cache, and that side effect won't be cancelled by rollback. 
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
  - One specific memory region in `probeTable` will be loaded into cache. Accessing that region is faster.
  - CPU found that branch prediction is wrong and rolls back, but doesn't rollback side effect on cache.
- The attacker measures memory read latency in `probeTable`. Which place access faster correspond to the value of secret.
- To accurately measure memory access latency, `performance.now()` is not accurate enough. It need to use a multi-threaded counter timer: One thread (web worker) keeps increasing a shared counter in a loop. The attacking thread reads that counter to get "time". The cross-thread counter sharing requires `SharedArrayBuffer`. Although it cannot measure time in standard units (e.g. nanosecond), it's accurate enough to distinguish latency difference between fast cache access and slow RAM access.

The same thing can also be done via equivalent Wasm code using `SharedArrayBuffer`.

Related: Another vulnerability related to cache side channel: [GoFetch](https://gofetch.fail/). It exploits Apple processors' cache prefetching functionality.

