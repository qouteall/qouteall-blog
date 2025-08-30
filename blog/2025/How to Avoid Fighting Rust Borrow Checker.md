---
date: 2025-08-30
tags:
  - Programming
---

# How to Avoid Fighting Rust Borrow Checker

<!-- truncate -->

The 3 important facts in Rust:

- **Tree-shaped ownership**. In Rust's ownership system, one object can own many children or no chlld, but must be owned by **exactly one parent**. Ownership relations form a tree. [^about_sharing]
- **Mutable borrow exclusiveness**. If there exists one mutable borrow for an object, then no other borrow to that object can exist. Mutable borrow is exclusive.
- **Borrow is contagious**. If you borrow a child, you indirectly borrow the parent (and parent's parent, and so on) **if crossing function boundary**. Just borrowing one wheel of a car makes you borrow the whole car. Combined with previous point, it can cause troubles for mutable data.

[^about_sharing]: Reference counting (`Rc`, `Arc`) allows sharing. Here I mean "native" Rust ownership relation form a tree.

Note that Rust borrow checker can reject some incorrect programs but also reject some correct programs.

## Considering reference shape

Firstly consider the reference [^about_reference] shape of your in-memory data.

[^about_reference]: Note that here "reference" here means reference in general OOP context (where there is no distinction between ownership and non-owning reference, think about reference in Java/C#/JS/Python). This is different to the Rust reference. I will use "borrow" for Rust reference in this article.

![](rust_reference_shape.drawio.png)

- If the reference is tree-shaped, then it's simple and natural in Rust. 
- If the reference shape has **sharing**, things become a little complicated.
    - Sharing means there are two or more references to the same object.
    - If shared object is immutable:
        - If the sharing is scoped (only temporarily shared), then you can use immutable borrow. You may need lifetime annotation.
        - If the sharing is not scoped (may share for a long time, not bounded within a scope), you need to use reference counting (`Rc` in singlethreaded case, `Arc` in possibly-multithreaded case)
    - If shared object is mutable, then it's in **borrow-check-unfriendly case**. Solutions elaborated below.
    - Contagious borrow can cause unwanted sharing (elaborated below).
- If the reference shape has **cycle**, then it's also in **borrow-check-unfriendly case**. Solutions elaborated below.

The most fighting with borrow checker happens in the **borrow-check-unfriendly cases**.

## Summarize solutions

The solutions in borrow-checker-unfriendly cases (will elaborate below):

- Data-oriented design. Avoid unnecessary getter and setter. Split borrow.
- Avoid just-for-convenience circular reference.
- Use ID/handle to replace borrow.
- Avoid mutation or defer mutation.
- `Arc<QCell<T>>`, `Arc<RwLock<T>>`
- Use unsafe and raw pointer.

## Contagious borrow issue

The previously mentioned two important facts:

- **Mutable borrow exclusiveness**. If you mutable borrow it, others cannot borrow it.
- **Borrow is contagious**. Just borrowing one wheel of a car makes you borrow the whole car, if the borrow crosses function boundary. (unless using split borrow that works within one scope)

A simple example: [^about_code_example]

```rust
pub struct Parent {  
    total_score: u32,  
    children: Vec<Child>  
}
pub struct Child {  
    score: u32  
}

impl Parent {  
    fn get_children(&self) -> &Vec<Child> {  
        &self.children  
    }  
  
    fn add_score(&mut self, score: u32) {  
        self.total_score += score;  
    }  
}

fn main() {  
    let mut parent = Parent{total_score: 0, children: vec![Child{score: 2}]};  
  
    for child in parent.get_children() {  
        parent.add_score(child.score);  
    }
}
```

[^about_code_example]: This simplified code example is just for illustrating contagious borrow issue. The total score doesn't need to be a mutable field. It's analogous a complex state that will exist in real applications. 

Compile error:

```
25 |     for child in parent.get_children() {
   |                  ---------------------
   |                  |
   |                  immutable borrow occurs here
   |                  immutable borrow later used here
26 |         parent.add_score(child.score);
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
```

This code is totally memory-safe: the `.add_score()` only touch the `total_score` field, and `.get_children()` only touch the `children` field. They work on separate data, they still clashes, because of **contagious borrow**:

- In `fn get_children(&self) -> &Vec<Child> { &self.children }`, although the method body just borrows `children` field, the return value indirectly borrows the whole `self`.
- `parent.get_children()` immutably borrows `parent`. It borrows the whole `parent`, not just a field in `parent`. The borrow checker works **locally** (just checked function signature) and doesn't check the body of `get_children`.
- The for loop indirectly borrows the result of `parent.get_children()` which indirectly borrows `parent`.
- In `fn add_score(&mut self, score: u32) { self.total_score += score; }`, the function body only mutably borrowed `total_score` field, but the argument `&mut self` borrows the whole `Parent`, not just one field.

The core problem is that you just want to borrow one field, but forced to borrow the whole object. And due to mutable borrow exclusiveness, this doesn't compile. (It often works fine in immutable case, because immutable borrows can co-exist for same object.)

What if I just inline `get_children` and `add_score`? Then it compiles fine:

```rust
pub struct Parent {  
    total_score: u32,  
    children: Vec<Child>  
}
pub struct Child {  
    score: u32  
}
fn main() {  
    let mut parent = Parent{total_score: 0, children: vec![Child{score: 2}]};  
  
    for child in &parent.children {  
        let score = child.score;  
        parent.total_score += score;  
    }  
}
```

Why that compiles? Because it does a **split borrow**: the compiler sees borrowing of individual fields in `main()` function, and don't do contagious borrow.

The deeper cause is that:

- **Borrow checker works locally**: when seeing a function call, it **only checks function signature**, instead of checking code inside the function. (Its benefit is to make borrow checking faster and simpler. Doing whole-program analysis is hard and slow, and doesn't work with things like dynamic linking.)
- **Information is lost in function signature**: the borrowing information becomes coarse-grained and is simplified in function signature. The type system does not allow expressing borrowing only one field, and can only express borrowing the whole object. [There are propsed solutions](https://smallcultfollowing.com/babysteps/blog/2025/02/25/view-types-redux/).

The solutions:

- **Avoid OOP-style getter and setter** (just make fields public), unless necessary.
  - **Data-oriented** design (DOD). Just directly work on data and make fields public.
  - If you design a library and want encapsulation, it's recommended to use ID/handle to replace borrow (elaborated below).
  - The getter that returns cloned/copied value is fine. If data is immutable, getter is also usually fine.
- Use ID/handle to replace borrow.
- Other workarounds like `Arc<QCell<>>` `Arc<RwLock<>>`, etc.
- **Avoid mutation** or **defer mutation**:

## Avoid mutation or defer mutation

The previous problem occurs partially due to mutable borrow exclusiveness. If all borrows are immutable, then contagious borrow is usually not a problem.

The common way of avoiding mutation is **mutate-by-recreate**: All data is immutable. When you want to mutate something, you create a new version of it. Just like in pure functional language (e.g. Haskell).

Unfortunately, **mutate-by-recreate is also contagious**: if you recreated a new version of a child, you need to also recreate a new version of parent that holds the new child, and parent's parent, and so on. In functional languages there are abstractions like [lens](https://hackage.haskell.org/package/lens) to make this kind of cascade-recreate more convenient.

Mutate-by-recreate can be useful for cases like:

- Safely sharing data in multithreading (read-copy-update(RCU), copy-on-write (COW))
- Take snapshot and rollback efficiently

Mutate-by-recreate can be optimized by sharing unchanged sub-structures. See also: [Persistent data structure](https://en.wikipedia.org/wiki/Persistent_data_structure), [Rope](https://en.wikipedia.org/wiki/Rope_(data_structure)).

Another solution is to **treat mutation as data**. To mutate something, **append a mutation command into command queue**. Then execute the mutation commands at once. (Note that command should not indirectly borrow base data.)

- In the process of creating new commands, it only do immutable borrow to base data, and only one mutable borrow to the command queue. 
- When executing the commands, it only do one mutable borrow to base data at a time.

What if I need the latest state before executing the commands in queue? Then inspect both the command queue and base data to get latest state ([LSM tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree) does similar things). This seems inconvenient, but can be often avoided by turning it into multiple stages, each stage only sees result of previous stage.

Treating mutation as data also has other benefits:

- The mutation can be serialized, and sent via network or saved to disk.
- The mutation can be inspected for debugging and logging.
- You can post-process the command list, such as sorting, filtering. If the data is sharded, the mutation command can dispatch to specific shard.
- In distributed system, there is a log (command list) that's synchronized between nodes using a consensus protocol (like Raft). The log is source-of-truth: the mutable state is completely derived from the log (and previous state checkpoints).
- The idea of turning operations into data is also adopted by [io_uring](https://en.wikipedia.org/wiki/Io_uring) and modern graphics APIs (Vulkan, Metal, WebGPU).

The previous code rewritten using deferred mutation:

```rust
pub struct Parent {  
    total_score: u32,  
    children: Vec<Child>  
}  
pub struct Child {  
    score: u32  
}  
pub enum Command {  
    AddTotalScore(u32),  
    // can add more kinds of commands  
}  
  
impl Parent {  
    fn get_children(&self) -> &Vec<Child> {  
        &self.children  
    }  
  
    fn add_score(&mut self, score: u32) {  
        self.total_score += score;  
    }  
}  
  
fn main() {  
    let mut parent = Parent{total_score: 0, children: vec![Child{score: 2}]};  
    let mut commands: Vec<Command> = Vec::new();  
  
    for child in parent.get_children() {  
        commands.push(Command::AddTotalScore(child.score));  
    }  
  
    for command in commands {  
        match command {  
            Command::AddTotalScore(num) => {  
                parent.add_score(num);  
            }  
        };  
    }  
}
```

## About circular reference

### Circular reference in mathematics

Some may argue that "Circular reference is a bad thing. Look how much trouble do circular references create in mathematics":

<details>

- [Circular proof](https://en.wikipedia.org/wiki/Circular_reasoning): if A then B, if B then A. Circular proof is wrong. It can prove neither A nor B.
- The set that indirectly includes itself cause [Russel's paradox](https://en.wikipedia.org/wiki/Russell%27s_paradox): Let R be the set of all sets that are not members of themselves. R contains R deduces R should not contain R, and vice versa. Set theory carefully avoids cirular reference.
- [Halting problem](https://en.wikipedia.org/wiki/Halting_problem) is proved impossible to solve, by using circular reference:
  
  Assume there exists a function `halts(program, input)`, which takes in a `program` and `input` data, and outputs a boolean telling whether `program(input)` will eventually halt.
  
  Then construct a paradox program `paradox`: 

```pseudocode
fn paradox(program: Program) {
    if halts(program, program) {
        while true {} // dead loop
    } else {
        return; // halts
    }
}
```

  Then `halts(paradox, paradox)` will cause a paradox. If it returns true, then `paradox(paradox)` halts, but in `paradox`'s definition it should deadloop.
  
  [Rice's theorem](https://en.wikipedia.org/wiki/Rice%27s_theorem) is an extension to Halting problem: All non-trivial semantic properties of programs are undecidable (includes whether it eventually halts).

- [Gödel's incomplete theorem](https://en.wikipedia.org/wiki/G%C3%B6del%27s_incompleteness_theorems). 
  - Firstly encode symbols, statements and proofs into data [^godel_integer]. The statements that contain free variables (e.g. x is a free variable in "x is an even number") can also be encoded (it can represent "functions" and even "higher-order functions").
  - `is_proof(theory, proof)` allows determining whether a proof successfully proves a theory. 
  - Then `provable(theory)` is defined as whether there exists a `proof` that satisfies `is_proof(theory, proof)`.
  - Unprovable is inverse of provable: `unprovable(theory) = !provable(theory)`
  - Let `H(x) = unprovable(x(x))`. Then let `G = H(H) = unprovable(H(H)) = unprovable(G)` [^godel_substitution], which creates a self-referencial statement: `G`  means `G` is not provable. If `G` is true, then `G` is not provable, then `G` is false, which is a paradox.

[^godel_integer]: Specifically, Gödel encodes symbols, statements and proofs into integer, called Gödel number. There exists many ways of encoding symbols/statements/proofs as data, and which exact way is not important. For simplicity, I will treat them all as data, and ignore the conversion between data and symbol/statements/proofs.

[^godel_substitution]: Here `x(x)` is symbol substitution. replacing the free variable `x` with `x`, while avoid making two different variables same name by renaming when necessary. It's also similar to Y combinator: `Y = f -> (x -> f(x(x))) (x -> f(x(x)))`. In that case `f = unprovable`, `H = x -> f(x(x))`, `Y(f) = H(H)`, `Y(f)` is a fixed point of `f`: `f(Y(f)) = Y(f)`. `G = Y(f)`, `f(G) = G`

There is something in common between Halting problem, Russel's paradox and Gödel's incomplete theorem: they all self-reference and "negate" itself, causing paradox.

</details>

### Circular reference in programming

Circular reference being bad in mathematics does NOT mean they are also bad in programming. The circular reference in math theories are different to circular reference in data. There are many valid cases of circular references in programming (e.g. there are doublely-linked list running in Linux kernel that works fine).

But circular reference do add risks to memory management:

- In C/C++, circular reference need to be carefully handled to avoid use-after-free.
- In GC languages, circular reference has memory leak risk. If all children references parents, and parents references children, then referencing any child will keep the whole structure alive.

Here are some common use cases of circular reference:

- Case 1: The parent references a child. The child references its parent, just for convenience. (Referencing to parent is not necessary, parent can be passed by argument)
- Case 2: The parent registers a callback to child. When something happened on child, the callback is called, and parent do something. It that case, parent references child, child references callback, callback references parent.
- Case 3: In a tree structure, the child references parent allows getting the path from one node to root node. Without it, you cannot get the path from just one node reference, and need to store variable-length path information.
- Case 4: The data is inherently a graph structure that can contain cycles.
- Case 5: The data requires self-reference.

### Avoid "just-for-convenience" circular reference in Rust

In the case 1 above: The child references parent, just for convenience. In OOP code. If you just have a reference to a child object, and you want to use some data in parent, child referencing parent would be convenient. Without it, the parent also need to be passed as argument. 

That convenience in OOP languages will lead to troubles in Rust. It's recommended to pass extra arguments instead of having circular reference.

Note that due to previously mentioned contagious borrow issue, you cannot mutably borrow child and parent at the same time (except using interior mutability). The workaround is to 1. do a split borrow on parent and pass the individual components of parent (pass more arguments and be more verbose than in other languages) 2. use interior mutability (e.g. `RefCell`, `Mutex`, `QCell`).

### The callback circular reference

**Observer pattern** is commonly used in GUI and other dynamic reactive systems. If parent want to be notified when some event happens on child, the parent register callback to child, and child calls callback when event happens.

However, the callback function object often have to reference the parent (because it need to use parent's data). Then it creates circular reference: **parent references child, child references callback, callback references parent**, as mentioned previously in case 2.

Solutions:

- Use reference counting and interior mutability. The classical ones: `Rc<RefCell<T>>` (singlethreaded), `Arc<RwLock<T>>` (multithreaded). I also recommend using [`QCell`](https://docs.rs/qcell/latest/qcell/) (elaborated below): `Rc<QCell<T>>`, `Arc<QCell<T>>`.
  
  The back-reference (callback to parent, child to parent) should use `Weak` to avoid memory leak.
- Use event bus to replace callbacks. Similar to the previous deferred mutation, we turn event into data. Each component listen to specific "event channel" or "event topic". When something happens, put the event into event bus, then event bus notifies components.
- Use ID/handle to replace borrow (elaborated later).

### The circular reference that's inherent in data structure

In the previously mentioned case 3 and case 4, circular reference is needed in data structure.

Solutions:

- Use reference counting and interior mutability (previously mentioned). This is **recommended when there are many different types of components and you want to add new types easily** (like in GUI).
- Use ID/handle to replace borrow (elaborated later). This is **recommended when you want more compact memory layout, and you rarely need to add new types into data** (suitable for data-intensive cases, can obtain better performance due to cache-friendliness).

### Self-reference

Self-reference means a struct contains an interior pointer to another part of data that it owns.

Zero-cost self reference requires `Pin` and `unsafe`. Normal Rust mutable borrow allow moving the value out (by `mem::replace`, or `mem::swap`, etc.). `Pin` disallows that, as self-reference pointer can be invalidated by moving. They are complex and hard to use. Workarounds includes separating child and use reference counting so it's no longer self-reference. 

## Use handle/ID to replace borrow

Data-oriented design:

- Try to pack data into contagious array, (instead of objects laid out sparsely managed by allocator).
- Use handle (e.g. array index) or ID to replace reference.
- **Decouple object ID with memory address**. An ID can be serialized (save to disk and sent via network) but a pointer cannot [^pointer_serialize].
- The different fields of the same object doesn't necessarily need to be together in memory. The one field of many objects can be put together (parallel array).
- Manage memory based on **arenas**.

[^pointer_serialize]: Actually, pointer is integer and can serialize, but you cannot use the same address in another process (or after process restart), because there may be other data in the same address.

One kind of arena is [slotmap](https://docs.rs/slotmap/latest/slotmap/):

- Slotmap is basically an array of elements, but each element has a version integer. 
- Each handle (key) has an index and a version. 
- When accessing the slotmap, it firstly does a bound check, then checks version. 
- After removing element, the version increments. The previous handle cannot get the new element at the same index, because of version mismatch.
- The handles (keys) are just `Copy`-able data that's not restricted by borrow checker.
- Although memory safe, it still has the equivalent of "use-after-free": using a handle of an already-removed object cannot get element from the slotmap [^slotmap_uniqueness]. Each get element operation may fail.
- Note that slotmap is not efficient when there are many unused empty space between elements. Slotmap offers two other variants for sparce case.

[^slotmap_uniqueness]: Each slotmap ensures key uniqueness, but if you mix keys of different slotmaps, the different keys of different slotmap may duplicate. Using the wrong key may successfully get an element but logically wrong.

Other map structure, like `HashMap` or `TreeMap` can also be arenas. 

If no element can be removed from arena, then a `Vec` can also be an areana.

One important fact: when we use ID/handle to replace borrow, the borrow checker no longer ensure that the ID/handle will point to a living object.

### Entity component system

Entity component system (ECS) is a way of organizing data that's different to OOP. In OOP, an object's fields are laid together in memory. But in ECS, each object is separated into components. The same kind of components for different entities are managed together (often laid together in memory). It can improve cache-friendliness. (Note that performance is affected by many factors and depends on exact case.)

ECS also favors composition over inheritance. Inheritance tend to bind code with specific types that cannot easily compose.

(For example, in an OOP game, `Player` extends `Entity`, `Enemy` extends `Entity`. There is a special enemy `Ghost` that ignores collision and also extends `Enemy`. But one day if you want to add a new player skill that temporarily ignores collision like `Ghost`, you cannot make `Player` extend `Ghost` and have to duplicate code. In ECS that can be solved by just combining special collision component.)

### Generalized reference and two reference semantics

The concept of **generalized reference**:

- The reference in GC languages is generalized reference.
- Pointer is generalized reference.
- Borrowing in Rust is generalized reference.
- Ownership in Rust is also considered as generalized reference.
- Smart pointer (`Rc`, `Arc`, `Weak`, `Box` in Rust, `shared_ptr`, `weak_ptr`, `unique_ptr` in C++, etc.) are generalized reference.
- **ID**s are generalized reference. (It includes all kinds of IDs, including **handles**, UUID, string id (URL, file path, username, etc.), integer id, primary key, and all kinds of identification information).

The generalized reference is separated into two kinds: strong and weak:

- Strong generalized reference: The system **ensures it always points to a living object**. 
  
  It contains: normal references in GC languages (when not null), Rust borrow and ownership, strong reference counting (`Rc`, `Arc`, `shared_ptr` when not null), and ID in database with foreign key constraint.
- Weak generalized reference: The system **does NOT ensure it points to a living object**.
  
  It contains: ID (no foreign key constraint), handles, weak reference in GC languages, weak reference counting (`Weak`, `weak_ptr`).

The major differences:

- For weak generalized references, **every data access may fail, and requires error handling**. (just panic is also a kind of error handling)
- For strong generalized reference, the **lifetime of referenced object is tightly coupled with the existence of reference**:
  - In Rust, the coupling comes from borrow checker. The borrow is limited by lifetime and other constraints.
  - In GC langauges, the coupling comes from GC. The existence of a strong reference keeps the object alive. Note that in GC languages there are **live-but-unusable objects** (e.g. Java `FileInputStream` is unusable after closing).
  - In reference counting, the coupling of course comes from runtime reference counting.
- For weak generalized reference, the **lifetime of object is decoupled from referces to it**.

If you want to design an abstraction that **decouples** object lifetime and how these objects are referenced, it's recommended to either:

- Use weak generalized reference, such as ID and handle. The object can be freed without having to consider how its IDs are held.
- Use strong generalized reference, but **add a new usability state that's decoupled with object lifetime**. This is common in GC languages. Examples:
  - In JS, if you send an `ArrayBuffer` to another web worker, the `ArrayBuffer` object can still be referenced and kept alive, but the binary content is no longer accessible from that `ArrayBuffer` object.
  - In Java, the IO-related objects (e.g. `FileInputStream`) can no longer be used after closing, even these objects are still referenced and still alive.


## Mutable borrow exclusiveness

As previously mentioned, Rust has **mutable borrow exclusiveness**:

- A mutable borrow to one object cannot co-exist with any other borrow to the same object. Two mutable borrows cannot co-exist. One mutable and one immutable also cannot co-exist.
- Multiple immutable borrows for one object can co-exist.

That is also called "mutation xor sharing", as mutation and sharing cannot co-exist.

In multi-threading case, this is natural: multiple threads read the same immutable data is fine. As long as one thread mutates the data, other thread cannot safely read or write it without other synchronization (atomics, locks, etc.).

But in single-threaded case, this restriction is not natural at all. No mainstream language (other than Rust) has this restriction.

> _Mutation xor sharing_ is, in some sense, neither necessary nor sufficient. It’s not _necessary_ because there are many programs (like every program written in Java) that share data like crazy and yet still work fine. It’s also not _sufficient_ in that there are many problems that demand some amount of sharing – which is why Rust has “backdoors” like `Arc<Mutex<T>>`, `AtomicU32`, and—the ultimate backdoor of them all—`unsafe`.
> 
> \- [The borrow checker within](https://smallcultfollowing.com/babysteps/blog/2024/06/02/the-borrow-checker-within/)

### Interior pointer

Rust has **interior pointer**. Interior pointer are the pointers that point into some data inside another object. A **mutation can invalidate the memory layout that interior pointer points to**. So that mutable borrow exclusiveness is still important for memory safety in single thread:

For example, you can take pointer of an element in `Vec`. If the `Vec` grows, it may allocate new memory and copy existing data to new memory, thus the interior pointer to it can become invalid (break the memory layout that interior pointer points to). Mutable borrow exclusiveness can prevent this issue from happening:

```rust
fn main() {  
    let mut vec: Vec<u32> = vec!(1, 2, 3);  
    let interior_pointer: &u32 = &vec[0];  
    vec.push(4);  
    print!("{}", *interior_pointer);  
}
```

Compile error:

```
3 |     let interior_pointer: &u32 = &vec[0];
  |                                   --- immutable borrow occurs here
4 |     vec.push(4);
  |     ^^^^^^^^^^^ mutable borrow occurs here
5 |     print!("{}", *interior_pointer);
  |                  ----------------- immutable borrow later used here
```

Another example is about `enum`: interior pointer pointing inside `enum` can also be invalidated, because different enum variants has different memory layout [^memory_layout]. In one layout the first 8 bytes is integer, in another layout the first 8 bytes may be a pointer. Treating an integer as a pointer is definitely not memory-safe.

[^memory_layout]: The memory layout here means **how we map between information and in-memory binary data**. It's not just the placement of fields of a struct.

```rust
enum DifferentMemoryLayout {  
    A(u64, u64),  
    B(String)  
}  
  
fn main() {  
    let mut v: DifferentMemoryLayout = DifferentMemoryLayout::A(1, 2);  
    let interior_pointer: &u64 = match v {  
        DifferentMemoryLayout::A(ref a, ref b) => {a}  
        DifferentMemoryLayout::B(_) => { panic!() }  
    };  
    v = DifferentMemoryLayout::B("hello".to_string());  
    println!("{}", *interior_pointer);  
}
```

Compile error:

```
9  |         DifferentMemoryLayout::A(ref a, ref b) => {a}
   |                                  ----- `v` is borrowed here
...
12 |     v = DifferentMemoryLayout::B("hello".to_string());
   |     ^ `v` is assigned to here but it was already borrowed
13 |     println!("{}", *interior_pointer);
   |                    ----------------- borrow later used here
```

Note that sometimes mutating can keep validity of interior pointer. For example, changing an element in `Vec<u32>` doesn't invalidate interior pointer to elements, because there is no memory layout change. But Rust by default prevents all mutation when interior pointer exists (unless using interior mutability).

### Interior pointer in other languages

Golang also supports interior pointer, but doesn't have such restriction. For example, interior pointer into slice:

```go
package main

import "fmt"

func main() {
	slice := []int{1, 2, 3}
	interiorPointer := &slice[0]
	slice = append(slice, 4)
	fmt.Printf("%v\n", *interiorPointer)
	fmt.Printf("old interior pointer: %p  new interior pointer: %p\n", interiorPointer, &slice[0])
}
```

Output

```
1
old interior pointer: 0xc0000ac000  new interior pointer: 0xc0000ae000
```

Because after re-allocating the slice, the old slice still exists in memory (not immediately freed). If there is an interior pointer into the old slice, the old slice won't be freed by GC. The interior pointer will always be memory-safe (but may point to stale data).

Golang also doesn't have sum type, so there is no equivalent to enum memory layout change in the previous Rust example.

Also, Golang's doesn't allow taking interior pointer to map entry value, but Rust allows. Rust's interior pointer is more powerful than Golang's.

In Java, there is no interior pointer. So no memory safety issue caused by interior pointer.

But in Java there is one thing logically similar to interior pointer: `Iterator`. Mutating a container can cause iterator invalidation:

```java
public class Main {  
    public static void main(String[] args) {  
        List<Integer> list = new ArrayList<>();  
        list.add(1);  
  
        Iterator<Integer> iterator = list.iterator();  
        while (iterator.hasNext()) {  
            Integer value = iterator.next();  
            if (value < 3) {  
                list.remove(0);  
            }  
        }  
    }  
}
```

That will get `java.util.ConcurrentModificationException`. Java's `ArrayList` has an internal version counter that's incremented every time it changes. The iterator code checks concurrent modification using version counter. 

Even without the version check, it will still be memory-safe because array access is range-checked.

Note that **iteration invalidation is logic error**, no matter whether it's memory-safe or not.

In Java, you can remove element via the iterator, then the iterator will update together with container, and no longer invalidate. Or use `removeIf` that avoids managing iterator.

Mutable borrow exclusiveness is still important in single-threaded case, because of interior pointer. **But if we don't use any interior pointer, then mutable borrow exclusiveness is not necessary for memory safety in single-thread case**.

That's why mainstream languages has no mutable borrow exclusiveness, and still works fine in single-threaded case. Java, JS and Python has no interior pointer. Golang and C# have interior pointer, they have GC and restrict interior pointer, so memory safe is still kept without mutable borrow exclusiveness.

One benefit of interior pointer is to allow tight memory layout, without having to do extra heap allocation just to get a reference some inner data.

### Interior mutability summary

Mutable borrow exclusiveness is overly restrictive. It is not necessary for memory safety in single-threaded code. It's also . there is **interior mutability** that allows getting rid of that constraint.

Interior mutability allows you to mutate something from an immutable reference to it. (Because of that, immutable borrow doesn't necessarily mean the pointed data is actually immutable. This can cause some confusion.)

Ways of interior mutability:

- `Cell<T>`. It's suitable for simple copy-able types like integer. In the previous contagious borrow example, if the `total_score` is replaced with `Cell<u32>` then mutating it doesn't need mutable borrow of parent thus avoid the issue. `Cell<T>` only supports replacing the whole `T` at once, and doesn't support getting a mutable borrow.
- `RefCell<T>`, suitable for data structure that does incremental mutation, in single-threaded cases. It has internal counters tracking how many immutable borrow and mutable borrow currently exist. If it detects violation of mutable borrow exclusiveness, `.borrow()` or `.borrow_mut()` will panic.It can cause crash if there is nested borrow that involves mutation. 
- `Mutex<T>` `RwLock<T>`, for locking in multi-threaded case. Its functionality is similar to `RefCell`. Note that unnecessary locking can cost performance, and has risk of deadlock. It's not recommended to overuse `Arc<Mutex<T>>` just because it can satisfy the borrow checker.
- [`QCell<T>`](https://docs.rs/qcell/latest/qcell/). Elaborated below.


They are usually used inside reference counting (`Arc<...>`, `Rc<...>`).

### `RefCell` is not panacea

In the previous contagious borrow case, wrapping parent in `RefCell<>` can make the code compile. However it doesn't fix the issue. It just turns compile error into runtime panic:

```rust
use std::cell::RefCell;  
pub struct Parent { total_score: u32,  children: Vec<Child> }  
pub struct Child { score: u32 }  
impl Parent {  
    fn get_children(&self) -> &Vec<Child> {  
        &self.children  
    }  
    fn add_score(&mut self, score: u32) {  
        self.total_score += score;  
    }  
}  
fn main() {  
    let parent: RefCell<Parent> = RefCell::new(Parent{total_score: 0, children: vec![Child{score: 2}]});  
    for child in parent.borrow().get_children() {  
        parent.borrow_mut().add_score(child.score);  
    }  
}
```

It will panic with `RefCell already borrowed` error.

In Rust, just having a mutable borrow `&mut T`, **Rust assumes that you can use it at any time**. **But holding the reference is different to using reference**. It's entirely possible that I have two `&mut T` for same object, but I only use one at a time. This is the use case that `RefCell` solves.

`RefCell` still follows mutable borrow exclusiveness rule. In previous contagious borrow example, the `Parent` is borrowed one immutablely and one mutable, thus `RefCell` will still panic at runtime.

Another problem: It's hard to return a reference borrowed from `RefCell`.

As the previous example can be fixed by `Cell`, without `RefCell`, here is another contagious borrow exampleL

```rust
use std::collections::HashMap;  
pub struct Parent {  
    entries: HashMap<String, Entry>,  
    passed_names: Vec<String>  
}  
pub struct Entry {  score: u32  }  
impl Parent {  
    fn get_entries(&self) -> &HashMap<String, Entry> {  
        &self.entries  
    }  
    fn add_name(&mut self, name: &str) {  
        self.passed_names.push(name.to_string());  
    }  
}  
fn main() {  
    let mut map: HashMap<String, Entry> = HashMap::new();  
    map.insert("a".to_string(), Entry { score: 100 });  
    let mut parent = Parent {  
        entries: map, passed_names: vec![]  
    };  
    for (name, entry) in parent.get_entries() {  
        if (entry.score > 20) {  
            parent.add_name(name);  
        }  
    }  
}
```

Compile error

```
22 |     for (name, entry) in parent.get_entries() {
   |                          --------------------
   |                          |
   |                          immutable borrow occurs here
   |                          immutable borrow later used here
23 |         if (entry.score > 20) {
24 |             parent.add_name(name);
   |             ^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
```

Then let's try to fix it using `RefCell`. As previously mentioned, wrapping `Parent` in `RefCell` doesn't work. We need to wrap two fields of parent into `RefCell`:

```rust
pub struct Parent {  
    entries: RefCell<HashMap<String, Entry>>,  
    passed_names: RefCell<Vec<String>>  
}  
pub struct Entry {  score: u32  }  
impl Parent {  
    fn get_entries(&self) -> &HashMap<String, Entry> {  
        self.entries.borrow()  
    }  
    fn add_name(&self, name: &str) {  
        self.passed_names.borrow_mut().push(name.to_string());  
    }  
}
```

But returning a reference in `RefCell` is not that simple:

```
error[E0308]: mismatched types
  --> src\main.rs:10:9
   |
9  |     fn get_entries(&self) -> &HashMap<String, Entry> {
   |                              ----------------------- expected `&HashMap<String, Entry>` because of return type
10 |         self.entries.borrow()
   |         ^^^^^^^^^^^^^^^^^^^^^ expected `&HashMap<String, Entry>`, found `Ref<'_, HashMap<String, Entry>>`
   |
   = note: expected reference `&HashMap<_, _>`
                 found struct `Ref<'_, HashMap<_, _>>`
help: consider borrowing here
   |
10 |         &self.entries.borrow()
   |         +
```

Because the reference borrowed from `RefCell` is not normal reference, it's actually `Ref`. `Ref` implements `Deref` so it can be used similar to a normal borrow. But it's different to a normal borrow.

The "help: consider borrowing here" suggestion won't solve the compiler error. If you follow its instruction you will get new compile error "returns a reference to data owned by the current function". It's important that **compiler's suggestion may be misleading**. Don't focus too much on compiler's suggestion.

Returning `Ref` works:

```rust
impl Parent {  
    fn get_entries(&self) -> Ref<HashMap<String, Entry>> {  
        self.entries.borrow()  
    }
}  
```

But returning `Ref` defeats the purpose of getter abstraction. `Ref` is tightly coupled with `RefCell`. For example, if one day the value returned by `get_entries` become data that's computed on demand:

```rust
impl Parent {  
    fn get_entries(&self) -> Ref<HashMap<String, Entry>> {  
        let mut map: HashMap<String, Entry> = HashMap::new();  
        map.insert("a".to_string(), Entry { score: 100 });  
        RefCell::new(map).borrow()  
    }  
}
```

Then

```
12 |         RefCell::new(map).borrow()
   |         -----------------^^^^^^^^^
   |         |
   |         returns a value referencing data owned by the current function
   |         temporary value created here
```

A more "adaptive" approach is to save referenced data as `Rc<RefCell<...>>`. But it's NOT recommended to overuse `Rc<RefCell<>>`, because:

- It has (relatively small) performance cost. `Rc` Reference counting and `RefCell` borrow counting costs peformance. (Performance is affected by many factors. It depends on exact case.)
- `Rc` has memory leak risk (need to use `Weak` to cut loop)
- `RefCell` has runtime panic risk. A previous example already shows. [See also](https://loglog.games/blog/leaving-rust-gamedev/#dynamic-borrow-checking-causes-unexpected-crashes-after-refactorings)
- Its syntax ergonomic is not good. The code will have a lot of "noise" like `.borrow().borrow_mut()` etc.

Similarily, `Arc` is the multi-threaded version of `Rc`. `Mutex` `RwLock` are the multi-threaded version of `RefCell`. It's also not recommended to overuse things like `Arc<Mutex<T>>`:

- `Arc`'s performance cost is larger than `Rc` because it involves atomic operation. Elaborated below.
- `Mutex` `RwLock`'s performance cost is larger than `RefCell` because locking also involves atomic operation, can reduce parallelism and can cause context switch.
- `Arc` also has memory leak risk (need to use `Weak` to cut loop)
- `Mutex` and `RwLock` have deadlock risk.
- Its syntax ergonomic is not good either.

[^cache_contention]: It's actually more complicated and depends on which hardware. 

### `Arc` is not always fast

`Arc` use atomic operations to change its reference count.

However, when many threads frequently change the same atomically counter, performance can degrade. The more threads touching it, the slower it is.

Modern CPUs use cache coherency protocol (e.g. [MOESI](https://en.wikipedia.org/wiki/MOESI_protocol)). **Atomic operations often require the CPU core to hold "exclusive ownership" to cache line** (this may vary between different hardware). Many threads frequently doing so cause cache contention, similar to locking, but on hardware.

[Example 1](https://web.archive.org/web/20250708051211/https://www.conviva.com/platform/the-concurrency-trap-how-an-atomic-counter-stalled-a-pipeline/), [Example 2](https://pkolaczk.github.io/server-slower-than-a-laptop/)

Solutions:

- Avoid sharing the same reference count. Copying data is sometimes better.
- [arc_swap](https://docs.rs/arc-swap/latest/arc_swap/). It uses [hazard pointer](https://en.wikipedia.org/wiki/Hazard_pointer) and other mechanics to increase performance.
- [trc](https://docs.rs/trc/1.2.4/trc/) and [hybrid_rc](https://docs.rs/hybrid-rc/latest/hybrid_rc/). They use per-thread not-atomic counter, and another global counter for how many threads use it. This can make changing shared refernce count be less frequent, getting higher performance.
- [aarc](https://docs.rs/aarc/latest/aarc/) and [crossbeam_epoch](https://docs.rs/crossbeam-epoch/latest/crossbeam_epoch/). Use epoch-based memory reclamation.

 These deferred memory reclamation techniques (hazard pointer, epoch-based) are also used in lock-free data structures. If one thread can read an element while another thread removes and frees the same element in parallel, it will not be memory-safe (this issue doesn't exist in GC languages).

### `QCell`

[`QCell<T>`](https://docs.rs/qcell/latest/qcell/) has an internal ID. `QCellOwner` is also an ID. You can only use `QCell` via an `QCellOwner` that has matched ID. 

The borrowing to `QCellOwner` "centralizes" the borrowing of many `QCell`s associated with it, ensureing mutable borrow exclusiveness. Using it require passing borrow of `QCellOwner` in argument everywhere it's used.

QCell will fail to borrow if the owner ID doesn't mismatch. Different to `RefCell`, if owner ID matches, it won't panic just because nested borrow.

Its runtime cost is low: just check whether cell's id matches owner's id.

One advantage of `QCell` is that the duplicated borrow will be compile-time error instead of runtime panic, which helps catch error earlier. If I change the previous `RefCell` panic example into `QCell`:

```rust
pub struct Parent { total_score: u32, children: Vec<Child> }
pub struct Child { score: u32 }
impl Parent {
    fn get_children(&self) -> &Vec<Child> { &self.children }
    fn add_score(&mut self, score: u32) { self.total_score += score; }
}
fn main() {
    let owner: QCellOwner = QCellOwner::new();
    let parent: QCell<Parent> = QCell::new(&owner, Parent{total_score: 0, children: vec![Child{score: 2}]});
    for child in parent.ro(&owner).get_children() {
        parent.rw(&mut owner).add_score(child.score);
    }
}
```

Compile error:

```
17 |     for child in parent.ro(&owner).get_children() {
   |                  --------------------------------
   |                  |         |
   |                  |         immutable borrow occurs here
   |                  immutable borrow later used here
18 |         parent.rw(&mut owner).add_score(child.score);
   |                   ^^^^^^^^^^ mutable borrow occurs here

```

[GPUI](https://zed.dev/blog/gpui-ownership)'s `Model<T>` is similar to `Rc<QCell<T>>`, where GPUI's `AppContext` correspond to `QCellOwner`.

It can also work in multithreading, by having `RwLock<QCellOwner>`. This can allow one lock to protect many pieces of data in different places [^lock_granularity].

[^lock_granularity]: Sometimes, having fine-grained lock is slower because of more lock/unlock operations. But sometimes having fine-grained lock is faster because it allows higher parallelism. Sometimes fine-grained lock can cause deadlock but coarse-grained lock won't deadlock. It depends on exact case.

[Ghost cell](https://docs.rs/ghost-cell/latest/ghost_cell/) and [LCell](https://docs.rs/qcell/latest/qcell/struct.LCell.html) are similar to QCell, but use closure lifetime as owner id. They are zero-cost, and more restrictive (use closure lifetime as owner id).

### Rust lock is not re-entrant

Re-entrant lock means one thread can lock one lock, then lock it again, then unlock twice, without deadlocking. Rust lock is not re-entrant. (Rust lock is also responsible for keeping mutable borrow exclusiveness. Allowing re-entrant can produce two `&mut` for same object.)

For example, in Java, the two-layer locking doesn't deadlock:

```java
public class Main {  
    public static void main(String[] args) {  
        Object lock = new Object();  
        synchronized (lock) {  
            synchronized (lock) {  
                System.out.println("within two layers of locking");  
            }  
        }  
        System.out.println("finish");  
    }  
}
```

But in Rust the equivalent will deadlock:

```rust
fn main() {  
    let mutex: Mutex<u64> = Mutex::new(0);  
    {  
        let mut g1: MutexGuard<u64> = mutex.lock().unwrap();  
        {  
            println!("going to do second-layer lock");  
            let mut g2 = mutex.lock().unwrap();  
            println!("within two layers of locking");  
        }  
    }  
    println!("finish");  
}
```

It prints `going to do second-layer lock` then deadlocks.

In Rust, it's important to **be clear about which scope holds lock**. You cannot just casually do locking like in Java/C# (this method touch that shared data so add `synchronized` [^casual_locking]). Golang lock is also not re-entrant.

[^casual_locking]: Although re-entrant lock is more tolerant to this kind of casual locking, not being clear about locking means risk of deadlock. Casual locking is not recommended even in Java.

Another important thing is that Rust only unlocks at the end of scope by default. `mutex.lock().unwrap()` gives a `MutexGuard<T>`. `MutexGuard` implements `Drop`, so it will drop at the end of scope. It's different to the local variables whose type doesn't implement `Drop`, they are dropped after their last use (unless borrowed). This is called NLL (non-lexical lifetime).

## Just clone the data

For example, if borrow checker has trouble with a string borrowing, you can just clone the string. It's usually fine as long as it's not performance bottleneck.

Note that there are two different kinds of data:

- The identity of object is important. Cloning it is treated as adding a new entity into the system.
- The identity of object is not important. Only object content matters. Cloning doesn't affect semantics. Cloning is fine.

## Using unsafe

By using unsafe you can freely manipulate pointers and are not restricted by borrow checker. But writing unsafe Rust is harder than just writing C, because you need to **carefully avoid breaking the constraints that safe Rust code relies on**. A bug in unsafe code can cause issue in safe code.

Writing unsafe Rust correctly is hard. Here are some traps in unsafe:

- Don't violate mutable borrow exclusiveness. 
  - A `&mut` cannot overlap with any other borrow that overlaps.
  - The overlap here also includes interior pointer. A `&mut` to an object cannot co-exist with any other borrow into any part of that object.
  - Violating that rule cause undefined behavior and can cause wrong optimization. Rust adds `noalias` attribute for mutable borrows into LLVM IR. LLVM will heavily optimize based on `noalias` (e.g. merging and reorder reads/writes). [See also](https://doc.rust-lang.org/nomicon/aliasing.html)
  - The above rule doesn't apply to raw pointer `*mut T`.
  - It's very easy to accidentally violate that rule when using borrows in unsafe. It's recommended to always use raw pointer and avoid using borrow (including slice borrow) in unsafe code. [Related1](https://chadaustin.me/2024/10/intrusive-linked-list-in-rust/), [Related2](https://web.archive.org/web/20230307172822/https://zackoverflow.dev/writing/unsafe-rust-vs-zig/)
- Pointer provenance.
  - Two pointers created from two provenance is considered to never alias. If their address equals, it's undefined behavior.
  - Converting an integer to pointer gets a pointer with no provenance, which is undefined behavior, unless the integer was converted from a pointer.
  - Adding a pointer with an integer doesn't change provenance.
- Using uninitialized memory is undefined behavior.
- `a = b` will drop the original object in place of `a`. If `a` is uninitialized, then it will drop an unitialized object, which is undefined behavior. Use `addr_of_mut!(...).write(...)` [Related](https://lucumr.pocoo.org/2022/1/30/unsafe-rust/)
- Handle panic unwinding.
- ...

Modern compilers tries to optimize as much as possible. **To optimize as much as possible, the compiler makes assumptions as much as possible. Breaking any of these assumption can lead to wrong optimization.** That's why it's so complex. [See also](https://queue.acm.org/detail.cfm?id=3212479)

Unfortunately Rust's syntax ergonomics on raw pointer is currently not good:

- If `p` is a raw pointer, you cannot write `p->field` (like in C/C++), and can only write `(*p).field`
- Raw pointer cannot be method receiver (self).
- There is no "raw pointer to slice". You need to manually `.add()` pointer and dereference. Bound checking is also manual.

## Contagious borrowing between branches

Current borrow checker does coarse-grained analysis on branch. One branch's borrowing is **contagious** to another branch.

Currently, this won't compile ([see also](https://blog.rust-lang.org/inside-rust/2023/10/06/polonius-update/)):

```rust
fn get_default<'r, K: Hash + Eq + Copy, V: Default>(
    map: &'r mut HashMap<K, V>,
    key: K,
) -> &'r mut V {
    match map.get_mut(&key) { // -------------+ 'r
        Some(value) => value,              // |
        None => {                          // |
            map.insert(key, V::default()); // |
            //  ^~~~~~ ERROR               // |
            map.get_mut(&key).unwrap()     // |
        }                                  // |
    }                                      // |
}   
```

Becaue the first branch `Some(value) => ...`'s output value indirectly mutably borrows `map`, the second branch has to also indirectly mutably borrow `map`, which conflicts with another mutable borrow in scope.

This will be fixed by [Polonius](https://rust-lang.github.io/polonius/current_status.html) borrow checker. Currently (2025 Aug) it's available in nightly Rust and can be enabled by an option.

## `Send` and `Sync`

Rust favors tree-shaped ownership. Each object is owned by exactly one place. If you send something to another thread, only one thread can access it, so it's thread-safe. No risk of data race.

Sending an immutable borrow to another thread is also fine as long as the shared data is actually immutable.

But there are exceptions. One exception is interior mutability, like `Cell` and `RefCell`. Because interior mutability, the data pointed by immutable borrow `&T` may no longer actually be immutable. So Rust prevents sharing `&Cell<T>` and `&RefCell<T>` by making `Cell<T>` and `RefCell<T>` not `Sync`. If `X` is `Sync` then `&X` is `Send`. If `X` is not `Sync` then `&X` is not `Send`. This prevents `Cell` and `RefCell` from being shared across threads.

`Rc`'s reference counter is not atomic, so `Rc` is neither `Send` or `Sync`.

But `&Mutex<T>` can be shared because lock protects them. Also immutable reference to atomic cells like `&AtomicU32` can be shared because of atomicity. `Mutex<T>` and `AtomicU32` are `Sync` so `&Mutex<T>` and `&AtomicU32` are `Send`.

There are things that are `Sync` but not `Send`, like `MutexGuard`. If something is already locked, sharing its reference to other threads temporarily is fine. But moving a `MutexGuard` to another thread is not fine because locking is tied to thread.

## When using async runtime

Tokio is a popular async runtime. 

In Tokio, submitting a task require the future to be `Send` and `'static`.

```rust
pub fn spawn<F>(future: F) -> JoinHandle<F::Output>  
where  
    F: Future + Send + 'static,  
    F::Output: Send + 'static,
```

It requires the future need to be `Send` and `'static`:

- `'static` means that the future is standalone and doesn't borrow other things. If the future need to share data with outside, pass `Arc<T>` into (not `&Arc<T>`). (Note that the "static" here is different to "static" in C/C++/Java/C#. In Rust, borrowing global variable is `'static` but standalone values are also `'static`.)
- `Send` means that the future can be sent across threads. Tokio use work-stealing, which means that one thread's task can be stolen by other threads that currently have no work.

Another important trap is that, the normal sleep `std::thread::sleep` and normal locking `std::sync::Mutex` should not be used when using async runtime, because they block using OS functionality without telling async runtime, so they will block the async runtime's scheduling thread. In Tokio, use `tokio::sync::Mutex` and `tokio::time::sleep`.

## Side effect of extracting and inlining variable

In C and GC languages:

- If a variable is used only once, you can inline that variable. This will only change execution order (except in short-circuit [^short_circuit]).
- Extracting a variable will only change execution order (except when variable is used twice or in short-circuit).

[^short_circuit]: `a() || b()` will not execute `b()` if `a()` returns true. `a() && b()` will not execute `b()` if `a()` returns false.

But in Rust it's different. 

### Lifetime of temporary value

- A temporary value drops immediately after evaluating, except when there is a borrow to it, its lifetime extends by the borrow. It's called temporary lifetime extension. 
  - There are implicit ways of creating borrow. `DeRef` can implicitly borrow, `a.b()` can implicitly borrow `a`
  - `match`, `if let` or `while let` can also borrow which extend the lifetime
  - Sometimes temporary lifetime extension doesn't work, such as `let guard = Mutex::new(0).lock().unwrap();`
- A value that's put into a local variable:
  - If its type implements `Drop`, then it will drop at the end of scope (one example is `MutexGuard`).
  - If its type doesn't implement `Drop`, then it will drop after its last use. This is called NLL (non-lexical lifetime).

Simplify: 

- Putting a temporary value to local variable usually make it live longer.
- Inlining a local variable usually make it live shorter.

### Reborrow

Normally mutable borrow `&mut T` can only be moved and cannot be copied. 

But **reborrow** is a feature that sometimes allow you to use a mutable borrow multiple times. Reborrow is very common in real-world Rust code. [Reborrow is not explicitly documented](https://github.com/rust-lang/reference/issues/788). [See also](https://haibane-tenshi.github.io/rust-reborrowing/)

Example:

```rust
fn mutate(i: &mut u32) -> &mut u32 {  
    *i += 1;  
    i  
}  
fn mutate_twice(i: &mut u32) -> &mut u32 {  
    mutate(i);  
    mutate(i)  
}
```

That works. Rust will implicitly treat `mutate(i)` as `mutate(&mut *i)` so that `i` is not moved into and become usable again.

But extracting the second `i` into a local variable early make it not compile:

```rust
fn mutate_twice(i: &mut u32) -> &mut u32 {  
    let j: &mut u32 = i;  
    mutate(i);  
    mutate(j)  
}
```

```
7  | fn mutate_twice(i: &mut u32) -> &mut u32 {
   |                    - let's call the lifetime of this reference `'1`
8  |     let j: &mut u32 = i;
   |                       - first mutable borrow occurs here
9  |     mutate(i);
   |            ^ second mutable borrow occurs here
10 |     mutate(j)
   |     --------- returning this value requires that `*i` is borrowed for `'1`
```

### Move cloned data into closure

`tokio::spawn` require future to be standalone and doesn't borrow other things (`'static`). 

Implicitly passing it is passing `&Arc<T>` makes the future borrow other things and be not standalone.

```rust
#[tokio::main]  
async fn main() {  
    let data: Arc<u64> = Arc::new(1);  
  
    let task1_handle = tokio::spawn(async move {  
        println!("From task: Data: {}", *data);  
    });  
  
    println!("From main thread: Data: {}", *data);  
}
```

Compile error

```
6    |     let data: Arc<u64> = Arc::new(1);
     |         ---- move occurs because `data` has type `Arc<u64>`, which does not implement the `Copy` trait
7    |
8    |     let task1_handle = tokio::spawn(async move {
     |                                     ---------- value moved here
9    |         println!("From task: Data: {}", *data);
     |                                          ---- variable moved due to use in coroutine
...
12   |     println!("From main thread: Data: {}", *data);
     |                                             ^^^^ value borrowed here after move
```

Manually clone the `Arc<T>` and put it into local variable works. It will make the cloned version to move into future:

```rust
#[tokio::main]  
async fn main() {  
    let data: Arc<u64> = Arc::new(1);  
  
    let data2 = data.clone(); // this is necessary
    let task1_handle = tokio::spawn(async move {  
        println!("From task: Data: {}", *data2);  
    });  
  
    println!("From main thread: Data: {}", *data);  
}
```

Note that inlining `data2` local variable make it not compile:

```rust
#[tokio::main]  
async fn main() {  
    let data: Arc<u64> = Arc::new(1);  
    let task1_handle = tokio::spawn(async move {  
        println!("From task: Data: {}", *data.clone());  
    });  
    println!("From main thread: Data: {}", *data);  
}
```

```
5    |     let data: Arc<u64> = Arc::new(1);
     |         ---- move occurs because `data` has type `Arc<u64>`, which does not implement the `Copy` trait
6    |     let task1_handle = tokio::spawn(async move {
     |                                     ---------- value moved here
7    |         println!("From task: Data: {}", *data.clone());
     |                                          ---- variable moved due to use in coroutine
8    |     });
9    |     println!("From main thread: Data: {}", *data);
     |                                             ^^^^ value borrowed here after move
```

[There is a proposal on improving syntax ergonomic of it.](https://rust-lang.github.io/rust-project-goals/2024h2/ergonomic-rc.html)

## Summarize the contagious things

- Borrowing that cross function boundary is contagious. Just borrowing a wheel of car can indirectly borrow the whole car.
- Mutate-by-recreate is contagious. Recreating child require also recreating parent that holds the new child, and parent's parent, and so on.
- Lifetime annotation is contagious. If some type has a lifetime parameter `'a`, then every type that holds it and every function signature that use it must also have the lifetime parameter `'a` (some may be auto-inferred, but not all). Refactoring that adds/remove lifetime parameter can be a huge work.
- In current borrow checker, one branch's borrowing is contagious to the whole branching scope.
- `async` is contagious. `async` function can call normal function. Normal function cannot easily call `async` function (but it's possible to call by blocking).
- Being not `Sync`/`Send` is contagious. A struct that indirectly owns a non-`Sync` data is not `Sync`. A struct that indirectly owns a non-`Send` data is not `Send`.
- Error passing is contagious. If panic is not acceptable, then all functions that indirectly call a fallible function must return `Result`.


