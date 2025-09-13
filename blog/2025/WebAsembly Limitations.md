
Background: [WebAssembly](https://webassembly.org/) is an execution model and a code format:

- It's designed with performance concern. It can achieve higher performance than JS [^wasm_js_perf].
- It's designed with safety concern. Its execution is sandboxed. It can be run in browser. 
- It's close to native assembly (e.g. X86, ARM) but abstracts in a cross-platform way, so that many C/C++/Rust/etc. applications can be compiled to Wasm (but with limitations).
- Although its name has "Web", it's is not just for Web. It can be used outside of browser.
- Although its name has "Assembly", it has features (e.g. [GC](https://github.com/WebAssembly/gc)) that are in a higher abstraction layer than native assembly.

[^wasm_js_perf]: WebAssembly is not always faster than JS, depending on how optimization efforts are put in, and browser's limitations. But WebAssembly has higher potential for computation performance than JS. JS has a lot of flexibility. Flexibility costs performance. JS runtime often use runtime statistics to find unused flexibility and optimize accordingly. But statistics cannot be really sure so JS runtime still have to "prepare" for flexibility. The runtime statistics and "prepare for flexibility" all costs performance, in a way that cannot be optimized without changing code format.

## Limitations

### Stack access

Normally program runs with a stack. The stack holds local variables, call arguments, function pointers and other things. The function pointer is for knowing where to jump when function returns.

In Wasm, the main stack is managed by Wasm runtime (e.g. V8 in Chromium, SpiderMonkey in Firefox). The main stack is not in lineary memory can cannot be read/written by linear memory operations.

It has benefits:

- It avoids security issues related to control flow hijacking. A native application's stack is in memory. An out-of-bound write can change the return function address, causing it to execute wrong code. Common mitigations such as [data execution prevention](https://en.wikipedia.org/wiki/Executable_space_protection) (DEP) and [stack smashing protection](https://en.wikipedia.org/wiki/Buffer_overflow_protection#Random_canaries) (SSP) are not needed by WebAssembly programs. [See also](https://webassembly.org/docs/security/)
- It allows the runtime to optimize stack layout without changing program behavior.

But it also have downsides:

- GC needs to scan the references (pointers) on stack. If the Wasm app use application-managed GC (not Wasm built-in GC) (for reasons explained below), then the on-stack references (pointer) need to be "spilled" to linear memory, which has costs (larger binary, slower execution). 
- Stack switching cannot be done. Golang use stack switching for goroutine scheduling. There is a [proposal](https://github.com/WebAssembly/stack-switching) for adding this functionality.
- Dynamic stack resizing cannot be done. Golang does dynamic stack resizing so that new goroutines can be initialized with small stacks, reducing memory usage.

Note that there are 3 different "stacks" that need to be clarified:

- The main execution stack, that holds local variable, call arguments, function pointers. It's not in linear memory. It's managed by Wasm runtime.
- The virtual operand stack. Wasm computation model is based on stack machine. The stack in stack machine can be seen as a way of passing temporary values. Some of them will be in register, some of them will be in the main execution stack.
- The shadow stack. It's in linear memory. Holds the local variables that need to be in linear memory. Managed by Wasm application, not Wasm runtime.

### Memory deallocation

The Wasm linear memory can be seen as a large array of bytes. Address in linear memory is the index into the array.

There is a `memory.grow` instruction tht grows a linear memory. However, there is no way to shrink the linear memory. There is no way to return allocated memory back to Wasm runtime then back to OS. Mobile platforms (iOS, Android, etc.) often kill background process that has large memory usage, so not returning memory can be an issue. [See also](https://github.com/WebAssembly/design/issues/1397)

### Wasm GC limitation



### Multi-threading


### Cannot directly call Web APIs


