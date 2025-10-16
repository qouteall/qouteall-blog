
# Some other WebAssembly Knowledge

Previous: [WebAssembly Limitations](./WebAsembly%20Limitations)

## Wasm runtime data

The data that Wasm program works on:

- Runtime-managed stack. It has local variables, function arguments, return code addresses, etc. It's managed by the runtime. It's not in linear memory.
- Linear memory. 
  - A linear memory is an array of bytes. Can be read/written by address (address can be seen as index in array). 
  - Its size must be multiple to page size. Page size is 64 KiB.
  - Wasm supports having multiple linear memories.
  - A linear memory's size can grow, in units of page size. But a linear memory's size cannot shrink.
  - A linear memory can be shared by multiple Wasm instances, see multi-threading section below.
- Table. Each table is a (growable) array that can hold:
  - Function references.
  - Extern value references. Extern value can be JS value or other things, depending on environment.
  - Exception references.
  - GC value references.
- Heap. Holds GC values.
- Globals. A global can hold a number (`i32`, `i64`, `f32`, `f64`), an `i128` or a reference (including function reference, GC value reference, extern value reference, etc.). The globals are not in linear memory.

## Wasm stack machine

WebAssembly use [stack machine](https://en.wikipedia.org/wiki/Stack_machine) execution model. It uses an implicit stack for passing values. It can push or pop the implicit stack. Wasm specification doesn't have registers. The implicit stack serves similar purpose as registers (but the machine code compiled from Wasm will use registers).

The stack layout (stack size, the type of element on i-th place on stack) must be deterministic before/after a branch or loop. You cannot have an `if` that one branch leaves an element on stack but another branch doesn't.

Wasm also have local variables, called locals. They are not in the implicit stack. They can be freely accessed within one function.

JVM bytecode also uses stack machine execution model.

## Wasm binary content

Wasm binary (that `.wasm` file holds) contains many sections.

The kinds of sections:


- Custom section
- Type section. It defines types. These types can later be referred to using integer index. Note that Wasm GC types use structural typing, not nominal typing (elaborated later). It allows the types to be mutually recursive (for Wasm GC). It also defines function types.
- Import section
- Function section
- Table section
- Memory section
- Global section
- Export section
- Start section
- Element section
- Code section
- Data section
- Data count section
- Tag section


## Index space

In Wasm binary, functions by default has no name. They are referenced using integer index in binary. Both imported functions and the functions defined in binary share the same indexing space.

Same applies to tags, globals, memories, tables.

Types cannot be imported or exported. Types are also referred to using index.

## Wasm execution

Two different concepts: Wasm module and Wasm instance:

- A wasm binary (content of `.wasm` file) can compiled to a `WebAssembly.Module` in JS (often using `WebAssembly.compileStreaming`). That compile includes validation (and possibly turning Wasm bytecode to machine code, depending on runtime implementation). The `WebAssembly.Module` itself is immutable and can be sent to other web workers.
- A `WebAssembly.Instance` is used for Wasm execution. `new WebAssembly.Instance(module, importObject)` can create a new instance by providing a module and imports. These runtime data includes: linear memories, tables, globals, etc.

[Wasm Modules specification](https://webassembly.github.io/spec/core/syntax/modules.html)

In browser, Wasm cannot bootstrap without JS.

Wasm can import and export these things:

- Functions.
- Linear memories. 
  - Wasm application can use multiple memories, but most applications just use one memory. 
  - You can create a `WebAssembly.Memory` in JS code and import it to Wasm. 
  - You can also make Wasm binary to declare a memory and export it.
- Globals. (Although called "global", they actually thread-local.)
- Tables.
- Tags. (used for exceptions)

Currently a Wasm binary cannot directly import another function of another Wasm binary, without using JS glue code. This will be solved by component model. A current workaround is to put function reference in table and use indirect call.

## Wasm linking

WebAssembly also has linking. The `.wasm` file is not directly compiled at once. Each sub-module (in C it correspond to `.c` source file) firstly compiles into Wasm object file. Then Wasm object files are linked to `.wasm` file.

In native C/C++, each `.c` file or `.cpp` file is compiled to an object file, then linked together. But it's different in Rust. Rust often use crate as the unit of compilation (sometimes one crate can split into multiple codegen units).

The Wasm object files use the same format as Wasm binary (`.wasm`). But individual object files are incomplete and not directly executable.

The linker does many things:

- In Wasm binary, the functions and globals are referred to using index, not name. Linking can change index of these things, so linker will update these indexes in Wasm binary. (The custom section `name` can hold the function names.)
- The data sections are merged. The memory address into data section also changes.
- The layout of function table changes. All indexes into function table need to be updated.
- TODO

About constants: there are integer constants, string constants and other kinds of constants.

- An integer constant can be put into immediate value in Wasm instruction (e.g. `i32.const`).
- An integer constant can also be put into a Wasm global.
- A contant can be put into data section. String constants are put into data section.

Wasm binary use [LEB128](https://en.wikipedia.org/wiki/LEB128) format for encoding integers.

When linking, these constants in instructions need to be changed (relocation):

- Function index of function call
- Linear memory address of constant data in data section
- Global index of global-related operations
- Table index of table-related operations
- ...

These constants are immediate values in instructions. There are other constants that are also immediate values in instructions. Without extra data, linker cannot know whether an immediate value in instruction is relocatable (linear memory address, function index, etc.) or just constant (some constants are put to immeidate value instead of data section or global). 

So, there is a special kind of custom section that holds relocation info. That relocation info contains offset into Wasm binary, pointing to function index/linear memory address/etc. in instruction. When linker tries to relocate, it reads relocation information and change corresponding value in instruction.

Although the functions within Wasm binary has no name, only index, the exports and imports have name, and they correspond to field name in JS object.

## Type system of GC objects

The types of GC objects are **structural type** (duck typing), instead of nominal typing. A struct type is defined by its field types.

In Wasm struct there is no field name. The fields are referred to using indexes.

Inside Wasm binary, there are type declarations, and types are referenced using index to a type declaration. It can be seen that type is nominal (index as name) within one Wasm binary.

Java (and C#, C, Rust, etc.) use nominal typing. Even if two classes have exactly same fields, the two classes are still different. Typescript and Golang interfaces use structural typing: as long as the data have these fields/methods, it satisfy the type.

Wasm doesn't need to import or export the types, thanks to structural typing.

But what if a type references it self? For example, a tree node's field's type will also be the tree node's type. If that struct type is defined by its field types, then that type expression will be infinitely long. So Wasm adds [recursive type annotation](https://webassembly.github.io/spec/core/syntax/types.html#recursive-types). To compare whether two recursive types are the same, Wasm defines [rolling and unrolling](https://webassembly.github.io/spec/core/valid/conventions.html#aux-unroll-deftype).

Wasm also allows prefix subtyping, which can express OOP inheritance. (But not multiple inheritance.)

About Java's `interface`: TODO
