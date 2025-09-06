
# Higher-Level Design Pattern

<!-- truncate -->

- Computation-data duality.
- Mutation-data duality.
- Partial computation and multi-stage computation.
- Generalized View.
- Invariant production, grow, and maintenance.

## Computation-data duality

It's often beneficial to turn an action into data, then replace the raw action code into an interpreter.

By turning action into data, the raw action code is split into two parts: the one generating action and the one executing action. (It's also related to moving computation stages)

It involves two different aspects: 

- Turn computation (logic and action) into data.
- Turn execution state into explicit data.

The benefit of turning logic and action into data:

- Closure (lambda expression). A function along with captured data. It allows [reusing a piece of code along with captured data](./About%20Code%20Reuse,%20Polymorphism%20and%20Abstraction#code-reuse-mechanisms). 
- Composition. The logic that's turned to data can be more easily composed. Functional programming encourages having simple building blocks and compose them into complex logic.

The benefit of turning execution state into explicit data:

- Inspection: Explicit execution state is easier to inspect and display (than stackframe and code execution position).
- Serialization: Explicit execution state can be serialized and deserialized, thus be stored to database and sent across network.
- Suspension: Explicit execution state allows temporarily suspending execution and resume it later. Suspending thread is harder (to avoid suspending the whole process, you need to run it in a separate thread) and less efficient (current OS is not optimized for one million threads). (Turning execution state as data. It's related to mutation-data duality)
- Modification: Explicit execution state can be modified. It makes cancellation and rollback easier. (Modifying execution stack and execution state is harder, and it's not supported by many mainstream languages.)
- Forking: Allows forking control flow, which can be useful in some kinds of simulations.

The distinction between computation and execution state is blurry. A closure can capture data. An execution state can be seen as a **continuation**, which is also a computation.

### Algebraic effect and execution state

**Algebraic effect**: An effect handler executes some code in a scope. Some code is executed under an effect handler. When it performs an effect, the control flow jumps to the effect handler, and the execution state (**delimited continuation**) up to the effect handler's scope is also saved. The effect handler can then resume using the execution state. [A simple introduction to Algebraic effects](https://overreacted.io/algebraic-effects-for-the-rest-of-us/)

The applications of algebraic effect idea:

- Async/await
- Generator
- React `Suspense`
- Serializing saved execution state so that it can be saved to disk or sent via network. Related: Restate.
- Control flow forking. 

### Don't always go too far on DSL

See also: [Configuration complexity clock](https://mikehadlow.blogspot.com/2012/05/configuration-complexity-clock.html)

When you try to handle many different business requests, one solution is to create a flexible rules engine. Configuring the rules engine can handle all of the requests.

However then a tradeoff befome salient:

- If the rules engine is high in abstraction level, doing a lot of predefined things under-the-hood, then: it will be unadaptive when a new requirement clashes with predefined behavior.
- If the rules engine is low in abstraction level, then doing things will require more configuration. It's not more convenient than just coding. It becomes a new DSL. The new DSL is often worse than mainstream languages because:
  - The new DSL often has poor debug support.
  - The new DSL often has no IDE support.
  - No existing libraries ecosystem. Need to reinvent wheel.
  - The new DSL is often less battle-tested and more buggy.

DSL are useful when it's high in abstraction level, and new requirements mostly follow the abstration.

## Mutation-data duality

Mutation can be represented as data. Data can be interpreted as mutation.

- Instead of just doing in-place mutation, we can enqueue a command (or event) to do mutation later. The command is then processed to do actual mutation. (It's also moving computatin between stages)
- **Event sourcing**. Derive latest state from a log. Express the latest state as a view of old state + mutations. The idea is adopted by database WAL, data replication, Lambda architecture, etc.
- Layered filesystem (in Docker). Mutating or adding file is creating a new layer. The unchanged previous layers can be cached and reused.

The benefits:

- Easier to inspect, audit and debug mutations, because mutations are explicit data, not implicit execution history. Easier to audit and replay.
- Can replay mutations and rollback easily.
- Can replicate (sync) data change without sending full data.

Abot rollback:

- Transactional databases allow rollback a uncommited transaction. (Implementation details vary between databases. PostgreSQL is append-only. MySQL InnoDB does in-place mutation on disk but writes undo log and redo log.)
- Editing software often need to support undo. It's often implemted by storing previous step's data, while sharing unchanged substructure to optimize.
- Multiplayer game client that does server-state-prediction (to reduce visible latency) need to rollback when prediction is invalidted by server's message.
- CPU does branch prediction and speculative execution. If branch prediction fails, it internally rollback ([Spectre vulnerability](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)) is caused by rollback not cancelling side effects in cache that can be measured in access speed).

In some places, we specify a new state and need to compute the mutation (diff). Examples:

- Git. Compute diff based on file snapshots. The diff can then be manipulated (e.g. merging, rebasing, cherry-pick).
- React. Compute diff from virtual data structure and apply to actual DOM. Sync change from virtual data structure to actual DOM.
- Kubernetes. You specify what nodes/services it has. Kubernetes found the diff between reality and configuration, then do actions (e.g. launch new service, close service) to cover the diff. 

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



Deferred compuation vs immediate compuation:

- Immediately free memory vs GC.
- Stream processing vs batch processing.
- Pytorch's most matrix operations are async. GPU computes in background. The tensor object's content may be yet unknown (and CPU will wait for GPU when you try to read its content).
- Some databases (PostgreSQL, SQLite, etc.) require deferred "vacuum" that rearranges storage space.

### Program lifecycle

A computation, an optimization, or a safety check can be done in:

- Pre-compile stage. (Code generation, IDE linting, etc.)
- Compile stage. (Compile-time computation, macros, dependent type theorem proving, etc.)
- Runtime stage. (runtime check, JIT compilation, etc.)
- After first run. (Offline profile-guided optimization, etc.)

Most computations that are done at compile time can be done at runtime (with extra performance cost). But if you want to avoid the performance cost by doing it in compile time, you often need to do complex type gymnastics (In Rust and C++, most runtime computation methods cannot be directly used in compile-time. Compile-time computation often require encoding data structure in types and a lot of type gymnastics. But the harmony between compile-time and runtime can be accomplished in Zig and dependently-typed languages. Related: [Statics-dynamics Biformity](https://hirrolot.github.io/posts/why-static-languages-suffer-from-complexity#)

## Generalized View

Views in SQL databases are "fake" tables that represents the result of a query. The view's data is derived from other tables (or views). 

The **generalized** concept of view: View takes one information model and present it as another information model, and allow operating as another information model.

The generalized view can be understood as:

- Encapsulating information. Hiding you the true underlying information and only expose derived information.
- Faking and lying information.

Examples of the generalized view concept:

- Bits are views of voltage in circuits. [^bits_view]
- Integers are views of bits
- Characters are views of integers. Strings are views of characters.
- Other complex data structures are views to binary data.
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

[^bits_view]: Also: In hard disk, magnetic field is viewed as bits. In CD, the pits and lands are viewed as bits. In SSD, the electron's position in floating gate is viewed as bits. In fiber optics, light pulses are viewed as bits. In quantum computer, the quantum state (like spin of electron) can be viewed as bits. ......

More generally:

- The mapping between binary data and information is view. Information is bits+context. The context is how the bits are mapped between information. **Type contains viewing from binary data to information**.
- Abstraction involves viewing different things as the same things.


### Dynamically-typed languages also have "types"

Dynamically-typed languages also have "types". The "type" here is **the mapping between in-memory data and information**.

Even in dynamic languages, the **data still has "shape" at runtime**. **The program only works with specific "shapes" of data**. It cannot work with arbitrary data. The code that computes number cannot work when you give it a file object. The code that requires a field cannot work when that field is missing.

Mainstream languages often have relatively simpler and less expressive type systems. Some "shape" of data are complex and cannot be easily expressed in mainstram languages' type system (without type erasure).

Dynamic languages are better in avoiding the shackle of unexpressive type system, and getting rid of syntax inconvenience related to type erasure (type erasure in typed languages require inconvenient things like type conversion).

### Computation-storage tradeoff

A view can be backed by either storage or computation (or a combination of storage and computation). 

Modern highly-parallel computation are often bottlenecked by IO and synchronization. Adding new computation hardware units is easy. Making the information to flow efficiently between these hardware units is hard.

## Invariant production, grow, and maintenance

Most algorithms use the idea of producing invariant, growing invariant and maintaining invariant:

- Initially, create invariant at the smallest scale (in the simplest case).
- Then incrementally make small invariant be larger, until the invariant become big enough to finish the task.
- For a data structure that has an invariant, every mutaiton to it need to maintain invariant.

Examples:

- Merge sort. Create sorted sub-sequence in smallest scale (e.g. two elements). Then merge two sorted sub-sequences into a bigger one, and continue. The invariant of sorted-ness grows up to the whole sequence.
- Quick sort. Select a pivot. Then partition the sequence into a part that's smaller than pivot and a part that's larger than pivot (and a part that equals pivot) [^quick_sort_equal_pivot]. By partitioning, it creates invariant $\text{LeftPartElements} < \text{Pivot} < \text{RightPartElements}$. By recursively creating such invariants until to the smallest scale (individual elements), the whole sequence is sorted.
- Binary search tree. It creates invariant $\text{LeftSubtreeElements} < \text{ParentNode} < \text{RightSubtreeElements}$. When there is only one node, the invariant is produced at the smallest scale. Every insertion then follows that invariant and then grows and maintains that invariant.
- Dijkstra algorithm. The visited nodes are the nodes whose shortest path from source are known. By using the nodes that we know shortest path, it "expands" on graph, knowing new node's shortest path from source. The algorithm iteratively add new nodes into the invariant, until it spans the whole graph.


[^quick_sort_equal_pivot]: About the elements that equals the pivot, how exactly to treat them is implementation-specific. Some put them into the first partition. Some put then into another third partition.

Two ways of maintaining invariant: immediate perfect invariant maintenance, delayed imperfect invariant maintenance.

Most bugs occur because an invariant is broken. Maintaining some complex invariants require all the code over different places to "collaborate" together. If one piece code was changed by a developer not knowing the invariant, it breaks.

One purpose of encapsulation is to reduce the chance the invariant be broken.

Type systems also help maintaining invariant. But a simple type system can only maintain simple invariants. Complex invariants require complex types to maintain. If it becomes too complex, type may be longer than execution code.

### Manually maintaining invariant is hard

If there is a source-of-truth mutable state, and another data that's derived from source-of-truth.

There are many ways of modelling it:

- Make the derived data always compute-on-demand.
- Store the derived data as mutable state and manually keep consistency with source-of-truth (manulally maintainining invariant).
- Use other tools to maintain invariant. E.g. use database index and constraints. Use reactive frameworks.

The second one is error-prone and is a common source of bugs.

## Corresponding GoF design patterns

[GoF design patterns](https://en.wikipedia.org/wiki/Design_Patterns)

- Computation-data duality:
  - Factory pattern. Turn object creation code into factory object.
  - Prototype pattern. Turn object creation code into copying prototype.
  - Chain of Responsibility pattern. Multiple processor objects process command objects in a pipeline.
  - Command pattern. Turn command (action) into object.
  - Interpreter pattern. Interpret data as computation.
  - Iterator pattern / generator. Turn iteration code into state machine.
  - Strategy pattern. Turn strategy into object (function value).
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
  - Facade pattern. View complex interfaces as simple interfaces.
  - Flyweight pattern. View owning into sharing.
  - Proxy pattern. Proxy object provides a view into other things.
  - Facade pattern. Provide a simpler abstraction.
  - Template method pattern. View different implementations as the same interface methods.
  - Flyweight pattern. Save memory by sharing common data. View shared data as owned data.
- Invariant production, grow and maintenance:
  - There is no GoF pattern that tightly corresponds to it.


Other GoF design patterns briefly explained:

- Visitor pattern. Can be replaced by pattern matching.
- State pattern. Make state a polymorphic object.
- Memento pattern. Backup the state to allow rollback. Although it's realted to mutable state, it doesn't involve turning mutation into data, so it's not mutation-data duality.
- Singleton pattern. It's similar to global variable, but can be late-initialzied, can be ploymorphic, etc.
- Mediator pattern. One abstraction to centrally manage other abstractions.

