
# Higher-Level Design Patterns

<!-- truncate -->

Higher-level software design patterns:

- Computation-data duality.
- Mutation-data duality.
- Partial computation and multi-stage computation.
- Generalized View.
- Invariant production, grow, and maintenance.

These patterns and ideas are often deeply connected and used together.

## Computation-data duality

It involves two different aspects: 

- Turn computation (logic and action) into data.
- Turn execution state into explicit data.

The benefit of turning computation (logic and action) into data:

- Closure (lambda expression, function value). A function along with captured data. It allows [reusing a piece of code along with captured data](./About%20Code%20Reuse,%20Polymorphism%20and%20Abstraction#code-reuse-mechanisms). It can help abstraction: separate the generation of computation (create function values) and execution of computation (executing function). (It's related to partial computation and multi-stage computation)
- Composition. The computation that's turned to data can be more easily composed. Functional programming encourages having simple building blocks and compose them into complex logic.
- Flexibility. The computation that's turned to data can be changed and rebuilt dynamically.

The benefit of turning execution state into explicit data:

- Inspection: Explicit execution state is easier to inspect and display  (the machine code can be optimized, and it's platform-depenent, so machine code execution position and runtime stack are harder to inspect and manipulate than explicit data)
- Serialization: Explicit execution state can be serialized and deserialized, thus be stored to database and sent across network. (Example: Restate)
- Suspension: Explicit execution state allows temporarily suspending execution and resume it later. Suspending thread is harder (to avoid suspending the whole process, you need to run it in a separate thread) and less efficient (current OS is not optimized for one million threads). (Turning execution state as data. It's related to mutation-data duality)
- Modification: Explicit execution state can be modified. It makes cancellation and rollback easier. (Modifying execution stack and execution state is harder, and it's not supported by many mainstream languages.)
- Forking: Allows forking control flow, which can be useful in some kinds of simulations.

The distinction between computation and execution state is blurry. A closure can capture data. An execution state can be seen as a **continuation**, which is also a computation.

### Algebraic effect and continuation

**Algebraic effect**: An effect handler executes some code in a scope. Some code is executed under an effect handler. When it performs an effect, the control flow jumps to the effect handler, and the execution state (**delimited continuation**) up to the effect handler's scope is also saved. The effect handler can then resume using the execution state. [A simple introduction to Algebraic effects](https://overreacted.io/algebraic-effects-for-the-rest-of-us/)

**Delimited continuation** is the execution state turned into data. It's delimited because the execution state only include the stackframes within effect handling scope.

The continuation (without "delimited") contains the whole execution state of the whole program (assume program is single-threaded). Delimited continuation is "local". Continuation is "global". The "local" one is more fine-grained and useful.

**Continuation passing style** (CPS) is a way of representing programs. In CPS, each function accepts a continuation. Returning becomes calling the continuation. Calling continuation is to continue execution. The output of continuation is the "final output of whole program" (if IO or mutable state involved, the "final output of whole program" can be empty).

### Don't always go too far on DSL

See also: [Configuration complexity clock](https://mikehadlow.blogspot.com/2012/05/configuration-complexity-clock.html)

When you try to handle many different business requests, one solution is to create a flexible rules engine. Configuring the rules engine can handle all of the requests.

However then a tradeoff befome salient:

- If the rules engine is high in abstraction level, doing a lot of predefined things under-the-hood, then: it will be unadaptive when a new requirement clashes with predefined behavior. [Simple interface = hardcoded defaults = less customizability](./About%20Code%20Reuse,%20Polymorphism%20and%20Abstraction#simple-interface--hardcoded-defaults--less-customizability).
- If the rules engine is low in abstraction level, then doing things will require more configuration. It's not more convenient than just coding. It becomes a new DSL. The new DSL is often worse than mainstream languages because:
  - The new DSL often has poor debug support.
  - The new DSL often has no IDE support.
  - No existing libraries ecosystem. Need to reinvent wheel.
  - The new DSL is often less battle-tested and more buggy.

DSL are useful when it's high in abstraction level, and new requirements mostly follow the abstration.

## Mutation-data duality

Mutation can be represented as data. Data can be interpreted as mutation.

- Instead of just doing in-place mutation, we can enqueue a command (or event) to do mutation later. The command is then processed to do actual mutation. (It's also moving computatin between stages)
- **Event sourcing**. Derive latest state from a events (log, mutations). Express the latest state as a view of old state + mutations. The idea is adopted by database WAL, data replication, Lambda architecture, etc.
- Layered filesystem (in Docker). Mutating or adding file is creating a new layer. The unchanged previous layers can be cached and reused.

The benefits:

- Easier to inspect, audit and debug mutations, because mutations are explicit data, not implicit execution history. Easier to audit and replay.
- Can replay mutations and rollback easily.
- Can replicate (sync) data change without sending full data.

About rollback:

- Transactional databases allow rollback a uncommited transaction. (Implementation details vary between databases. PostgreSQL write is append-only on disk except vacuum. MySQL InnoDB does in-place mutation on disk but writes undo log and redo log.)
- Editing software often need to support undo. It's often implemted by storing previous step's data, while sharing unchanged substructure to optimize.
- Multiplayer game client that does server-state-prediction (to reduce visible latency) need to rollback when prediction is invalidted by server's message.
- CPU does branch prediction and speculative execution. If branch prediction fails or there is other failure, it internally rollback. ([Spectre vulnerability](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)) and [Meltdown vulnerability](https://en.wikipedia.org/wiki/Meltdown_(security_vulnerability)) are caused by rollback not cancelling side effects in cache that can be measured in access speed).

In some places, we specify a new state and need to compute the mutation (diff). Examples:

- Git. Compute diff based on file snapshots. The diff can then be manipulated (e.g. merging, rebasing, cherry-pick).
- React. Compute diff from virtual data structure and apply to actual DOM. Sync change from virtual data structure to actual DOM.
- Kubernetes. You configure what pods/volumes/... should exist. Kubernetes observes the diff between reality and configuration, then do actions (e.g. launch new pod, destroy pod) to cover the diff. 

Related: avoiding system calls and replacing calls with data can improve performance:

- io_uring: after initial setup, IO operations no longer involve system call. Just writing to memory can dispatch IO tasks.
- Graphics API: Old OpenGL use system calls to change state and dispatch draw call. New Graphics APIs like Vulkan, Metal and WebGPU all use command buffer. Operations are turned to data in command buffer.

Bitemporal modelling: Store two pieces of records. One records the data and time updated to database. Another records the data and time that reflect the reality. (Sometimes the reality changes but database doesn't edit immediately. Sometimes database contains wrong informaiton that's corrected later.) 

Related: Mutate-by-recreate. Keep data immutable and only change one root reference. It can avoid data race issues under parallelism (copy-on-write, read-copy-update, reading old snapshot during mutation).


## Partial computation and multi-stage computation

Partial computation: only compute some parts of the data, and keep the structure of the whole computation.

- In lazy evaluation, the unobserved data is not computed.
- Deferred mutation. Relates to mutation-data duality.
- Replace immediately executed code with data (expression tree, DSL, etc.) that will be executed (interpreted) later. Relates to computation-data duality.
- In multi-stage programming, some data are fixed while some data are unknown. The fixed data can be used for optimization. It can be seen as runtime constant value folding. JIT can be seen as treating bytecode as runtime constant and fold them in interpreter code.
- Replacing a value with a function or expression tree helps handling the currently-unknown data.
- Using a future (promise) object to represent a pending computation.
- In Idris, having a hole and inspecting the type of hole can help proving.

Deferred (async) compuation vs immediate compuation:

- Immediately free memory is immediate computation. GC is deferred computation.
- Stream processing is immediate computation. Batch processing is deferred computation.
- Pytorch's most matrix operations are async. GPU computes in background. The tensor object's content may be yet unknown (and CPU will wait for GPU when you try to read its content).
- PostgreSQL and SQLite require deferred "vacuum" that rearranges storage space.

### Program lifecycle

A computation, an optimization, or a safety check can be done in:

- Pre-compile stage. (Code generation, IDE linting, etc.)
- Compile stage. (Compile-time computation, macros, dependent type theorem proving, etc.)
- Runtime stage. (Runtime check, JIT compilation, etc.)
- After first run. (Offline profile-guided optimization, etc.)

Most computations that are done at compile time can be done at runtime (with extra performance cost). But if you want to avoid the performance cost by doing it in compile time, it becomes harder. 

Rust and C++ has **Statics-Dynamics Biformity** ([see also](https://hirrolot.github.io/posts/why-static-languages-suffer-from-complexity#)): most runtime computation methods cannot be easily used in compile-time. Using compile-time mechanisms often require data to be encoded in types, which then require type gymnastics.

The ways that solve (or partially solve) the biformity between compile-time and runtime computation:

- Zig compile-time computation and reflection. [See also](https://ziglang.org/documentation/master/#comptime)
- Dependently-typed languages. (e.g. Idris, Lean)
- Scala multi-stage programming. [See also](https://docs.scala-lang.org/scala3/reference/metaprogramming/staging.html). It's at runtime, not compile-time. But its purpose is similar to macro and code generation. Dynamically compose code at runtime and then get JITed.

## Generalized View

Views in SQL databases are "fake" tables that represents the result of a query. The view's data is derived from other tables (or views). 

The **generalized** concept of view: View takes one information model and present it as another information model, and allow operating as another information model.

The generalized view can be understood as:

- Encapsulating information. Hiding you the true underlying information and only expose derived information.
- "Faking" information.

Examples of the generalized view concept:

- Bits are views of voltage in circuits. [^bits_view]
- Integers are views of bits
- Characters are views of integers. Strings are views of characters.
- Other complex data structures are views to binary data. (pointer can be seen as a view to pointed data)
- A functions is a view of a mapping.
- Index (and lookup acceleration structure) are also views to underlying data.
- Cache is view to underlying data/computation.
- Lazy evaluation provides a view to the computation result.
- Virtual memory is a view to physical memory.
- File system is a view to data on disk. The not-on-disk data can also be viewed as files (Unix everything-is-file philosophy).
- Symbolic link in file systems is a view to another point in file system.
- Database provides generalized views of in-disk/in-memory data.
- Linux namespaces, hypervisors, sandboxes, etc. provides view of aspects of the system.
- Proxy, NAT, firewall, virtualized networking etc. provides manipulated view of network.
- Transaction isolation in databases provide views of data (e.g. snapshot isolation).
- Replicated data and redundant data are views to the original data.
- Multi-tier storage system. From small-fast ones to large-slow ones: register, cache, memory, disk, cloud storage.
- Previously mentioned computation-data duality and mutation-data duality can be also seen as viewing.

[^bits_view]: Also: In hard disk, magnetic field is viewed as bits. In CD, the pits and lands are viewed as bits. In SSD, the electron's position in floating gate is viewed as bits. In fiber optics, light pulses are viewed as bits. In quantum computer, the quantum state (like spin of electron) can be viewed as bits. ......

More generally:

- The mapping between binary data and information is view. **Information is bits+context**. The context is how the bits are mapped between information. **Type contains viewing from binary data to information**.
- **Abstraction involves viewing different things as the same things**.


### Dynamically-typed languages also have "types"

Dynamically-typed languages also have "types". The "type" here is **the mapping between in-memory data and information**.

Even in dynamic languages, the **data still has "shape" at runtime**. **The program only works with specific "shapes" of data**. 

For example, in Python, if a function accepts an array of string, but you pass it one string, then it treats each character as a string, which is wrong.

Mainstream languages often have relatively simpler and less expressive type systems. Some "shape" of data are complex and cannot be easily expressed in mainstram languages' type system (without type erasure).

Dynamic languages' benefits:

- Avoid the shackle of an unexpressive type sysytem.
- Avoid syntax inconvenience related to type erasure (type erasure in typed languages require inconvenient things like type conversion).
- Can quickly iterate by changing one part of program, before the changes work with other parts of program (in static typed languages, you need to resolve all compile errors in the code that you don't use now). This is double-edged sword. The broken code that was not tested tend to get missed.
- Save some time typing types and type definitions.

But the statically-typed languages and IDEs are improving. The more expressive type systems reduce friction of typing. Types can help catching mistakes, help understanding code and help IDE functionalities. Type inference and IDE completion saves time of typing types.

### Computation-storage tradeoff

A view can be backed by either storage or computation (or a combination of storage and computation). 

Modern highly-parallel computation are often bottlenecked by IO and synchronization. Adding new computation hardware units is easy. Making the information to flow efficiently between these hardware units is hard.

When memory IO becomes bottleneck, re-computing rather than storing can be beneficial.

## Invariant production, grow, and maintenance

Most algorithms use the idea of producing invariant, growing invariant and maintaining invariant:

- Produce invariant. Create invariant at the smallest scale, in the simplest case.
- Grow invariant. Combine or expand small invariants to make them larger. This often utilizes **transitive rule**. Do it until the invariant become big enough to finish the task.
- Maintain invariant. Every mutaiton to a data structure need to maintain its invariant.

About transitive rule: if X and Y both follow invariant, then result of "merging" X and Y also follows invariant. "Transitive" is why the invariant can grow without re-checking the whole data. When the invariant is forced in language level, it can be "[contagious](./How%20to%20Avoid%20Fighting%20Rust%20Borrow%20Checker#summarize-the-contagious-things)".

### Invariant in algorithms

- Merge sort. Create sorted sub-sequence in smallest scale (e.g. two elements). Then merge two sorted sub-sequences into a bigger one, and continue. The invariant of sorted-ness grows up to the whole sequence.
- Quick sort. Select a pivot. Then partition the sequence into a part that's smaller than pivot and a part that's larger than pivot (and a part that equals pivot) [^quick_sort_equal_pivot]. By partitioning, it creates invariant $\text{LeftPartElements} < \text{Pivot} < \text{RightPartElements}$. By recursively creating such invariants until to the smallest scale (individual elements), the whole sequence is sorted.
- Binary search tree. It creates invariant $\text{LeftSubtreeElements} \leq \text{ParentNode} \leq \text{RightSubtreeElements}$. When there is only one node, the invariant is produced at the smallest scale. Every insertion then follows that invariant and then grows and maintains that invariant.
- Dijkstra algorithm. The visited nodes are the nodes whose shortest path from source node are known. By using the nodes that we know shortest path, it "expands" on graph, knowing new node's shortest path from source. The algorithm iteratively add new nodes into the invariant, until it expands to destination node.
- Dynamic programming. The problem is separated into sub-problems. There is no cycle dependency between sub-problems. One problem's result can be quickly calculated from sub-problem's results (e.g. max, min). 
- ......


[^quick_sort_equal_pivot]: About the elements that equals the pivot, how exactly to treat them is implementation-specific. Some put them into the first partition. Some put then into another third partition.

### Invariant in application

For example, invariants in business logic:

- User name cannot duplicate.
- Bank account balance should never be negative. No over-spend.
- Product inventory count should never be negative. No over-sell.
- One room in hotel cannot be booked two times with time overlap.
- The product should be shipped before the order is paid.
- Any new comment message should correspond to a notification. No lost notification or duplicated notification.
- The user can't view or change information that's out of their permission.
- User cannot use a functionality if subscription ends.
- ......

Invariants in data:

- The reduntant data, derived data and acceleration data structure (index, cache) should stay consistent with base data (source-of-truth).
- The client side data should be consistent with server side data.
- Memory safety invariants. Pointer should point to valid data. Should not use-after-free. Only free once. etc.
- Thread safety invariants. This non-thread-safe data structure should not be shared between threads. That data structure must be accessed under lock. etc.
- The modification of some action should be cancelled after a subsequent action fails. (Ad-hoc transaction, application-managed rollback)

### Maintaining invariant

The timing of maintaining invariant: 

- Immediate invariant maintenance
- Delayed invariant maintenance (tolerant stale data. cache, batched processing)

The responsibility of maintaining invariant:

- The database/framework/OS/language etc. is responsible of maintaining invariant. For example, database maintains the validity of index and materialized view. If they don't have bugs, the invariant won't be violated.
- The application code is responsible for maintaining the invariant. This is more **error-prone**.

In the second case (application code maintains invariant), to make it less error prone, we can **encapsulate the data and the invariant-maintaining code**, and ensuring that **any usage of encapsulated API won't violate the invariant**. If some usages of API can break the invariant and developer can only know it by considering implementation, then it's a leaky abstraction.

For example, one common invariant to maintain is **consistency between derived data and base data (source-of-truth)**. There are many solutions:

- Make the derived data always **compute-on-demand**. No longer need to manually maintain invariant. But it may cost performance.
  - Caching can make compute-on-demand be faster.
- Store the derived data as mutable state and manually keep consistency with source-of-truth. This is the most error-prone solution. All **modifications to base data should "notify" the derived data** to update accordingly. Sometimes notify is to call a function. Sometimes notify involves networking.
  - A more complex case: the derived data need to modified in a way that reflect to base data. This **violates single source-of-truth**. It's even more error-prone.
  - Even more complex: the client side need to reduce visible latency by predicting server side data, and wrong prediction need to be corrected by server side data. It not only violates single source-of-truth but also often require rollback mechanism.
- Relying on other tools (database/framework/OS/language etc.) to maintain the invariant, as previously mentioned.

In real-world legacy code, **invariants are often not documented**. They are implicit in code. A developer not knowing an invariant can easily break it.

Type systems also help maintaining invariant. But a simple type system can only maintain simple invariants. Complex invariants require complex types to maintain. If it becomes too complex, type may be longer than execution code, and type errors become harder to resolve. It's a tradeoff.

## Corresponding GoF design patterns

[GoF design patterns](https://en.wikipedia.org/wiki/Design_Patterns)

- Computation-data duality:
  - Factory pattern. Turn object creation code into factory object.
  - Prototype pattern. Turn object creation code into copying prototype.
  - Chain of Responsibility pattern. Multiple processor objects process command objects in a pipeline.
  - Command pattern. Turn command (action) into object.
  - Interpreter pattern. Interpret data as computation.
  - Iterator pattern / generator. Turn iteration code into state machine.
  - Strategy pattern. Turn strategy into object.
  - Observer pattern. Turn event handling code into an observer.
- Mutation-data duality:
  - Command pattern. It also involves computation-data duality. The command can both represent mutation, action and computation.
- Partial computation and multi-stage computation:
  - Builder pattern. Turn the process of creating object into multiple stages. Each stage builds a part.
  - Chain of Responsibility pattern. The processing is separated into a pipeline. It's also in computation-data duality.
- Generalized view:
  - Adapter pattern. View one interface as another interface.
  - Bridge pattern. View the underlying different implentations as one interface.
  - Composite pattern. View multiple objects as one object.
  - Decorator pattern. Wrap an object, changing its behavior and view it as the same object.
  - Facade pattern. View multiple complex interfaces as one simple interface.
  - Proxy pattern. Proxy object provides a view into other things.
  - Template method pattern. View different implementations as the same interface methods.
  - Flyweight pattern. Save memory by sharing common data. View shared data as owned data.
- Invariant production, grow and maintenance:
  - There is no GoF pattern that tightly corresponds to it.


Other GoF design patterns briefly explained:

- Visitor pattern. Can be replaced by pattern matching.
- State pattern. Make state a polymorphic object.
- Memento pattern. Backup the state to allow rollback. Although it's related to mutable state, it doesn't involve turning mutation into data, so it's not mutation-data duality.
- Singleton pattern. It's similar to global variable, but can be late-initialized, can be ploymorphic, etc.
- Mediator pattern. One abstraction to centrally manage other abstractions.

