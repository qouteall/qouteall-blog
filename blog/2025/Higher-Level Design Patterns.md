
# Higher-Level Design Pattern

<!-- truncate -->

- Computation-data duality.
- Mutation-data duality.
- Move computation between stages.
- Generalized View.
- Invariant production, grow, and maintenance.

## Computation-data duality

It's often beneficial to turn an action into data, then replace the raw action code into an interpreter.

By turning action into code, the raw action code is split into two parts: the one generating action and the one executing action. (It's also related to moving computation stages)

Some benefits:

- Higher-order functions. It allows [reusing a piece of code along with captured data](./About%20Code%20Reuse,%20Polymorphism%20and%20Abstraction#code-reuse-mechanisms). 
- Inspection: Explicit execution state is easier to inspect and display (than stackframe and code execution position).
- Serialization: Explicit execution state can be serialized and deserialized, thus be stored to database and sent across network.
- Suspension: Explicit execution state allows temporarily suspending execution and resume it later. Suspending thread is harder (to avoid suspending the whole process, you need to run it in a separate thread) and less efficient (current OS is not optimized for one million threads). (Turning execution state as data. It's related to mutation-data duality)
- Modification: Explicit execution state can be modified. It makes cancellation and rollback easier. (Modifying execution stack and execution state is harder, and it's not supported by many mainstream languages.)
- Forking: Allows forking control flow, which can be useful in some kinds of simulations.


### Algebraic effect and execution state

**Algebraic effect**: An effect handler executes some code in a scope. Some code is executed under an effect handler. When it performs an effect, the control flow jumps to the effect handler, and the execution state (delimited continuation) (up to the effect handler's scope) is also saved. The effect handler can then resume using the execution state. [A more detailed introduction to Algebraic effects](https://overreacted.io/algebraic-effects-for-the-rest-of-us/)

The applications of algebraic effect idea:

- Async/await
- Generator
- React `Suspense`
- Serializing saved execution state so that it can be saved to disk or sent via network. Example: Restate.

One core idea is to **save the execution state, allowing resuming execution later**. 

## Mutation-data duality

Mutation can be represented as data. Data can be interpreted as mutation.

- Instead of just doing in-place mutation, we can enqueue a command to do mutation later. The command is then processed to do actual mutation.
- In a transactional database, modifying things in a transaction adds mutation records instead of just modify in-place. The mutation is persisted when transaction commits. With snapshot isolation, mutation in one transaction is invisible to another except for locking, but mutation in current transaction is visible to own.
- In a client-side GUI application, modifying a thing requires sending a request to the server. That request is data. The server's response is also data, which may confirm or deny the modification. The client can display modified data before server responds to reduce latency, but need to rollback temporary changes when the server denies the modification (in multiplayer cases, one player's modification can invalidate another player's modification).
- Derive latest state from a log. Examples: Database WAL, Raft, Lambda architecture.

Instead of doing in-place modification, we can:
- Defer mutation. Put updates in some queue and mutate in a deferred way.
- Keep both the new state and the old states (copy-on-write, read-copy-update).
- Express the latest state as a view of old state + mutations.
- Two alternate buffers. Example: when simulating Conway's game of life, you need two buffers. Just having one buffer and doing in-place modification will give wrong result, as each simulation step requires the previous state, and in-place modification replaces some previous states with new states. Sometimes a computation requires old state, and sometimes it requires new state, depending on exact requirements.
- Layered filesystem (in Docker).

The benefits:
- Easier to inspect and debug mutations, because mutations are explicit data, not implicit execution history.
- Avoid data race issues under parallelism (copy-on-write, reading old snapshot, make mutation command processing sequential).
- Make rollback and undo easier. Rollback is common in transactional databases (transaction can rollback), editing software (need to support undo) and multiplayer client (one player's edit can be invalidated by another player or other server-driven events, but the client edit need to display immediatey).
- Allow merging mutations in distributed systems (like Git).

In some places, we have a new state and need to compute the mutation (diff). Examples: Git, React. In Git and React, there is one commonality that the state itself is outside of their control and calculating the diff is how they "sync" the mutation to other systems (sync change to DOM in React, sync changes to other machines in Git).

Bitemporal modelling: Store two pieces of records. One records the data and time updated to database. Another records the data and time that reflect the reality. (Sometimes the reality changes but database doesn't edit immediately. Sometimes database contains wrong informaiton that's corrected later.) 

## Move computation between stages

Partial computation: only compute some parts of the data, and keep the structure of the whole computation.

- In lazy evaluation, the unobserved data is not computed.
- Deferred mutation. Relates to mutation-data duality.
- Replace immediately executed code with data (expression tree, DSL, etc.) that will be executed (interpreted) later. Relates to computation-data duality.
- In multi-stage programming, some data are fixed while some data are unknown. The fixed data can be used for optimization. It can be seen as runtime constant value folding. JIT can be seen as treating bytecode as runtime constant and fold them in interpreter code.
- Replacing a value with a function that produces value or AST helps handling the currently-unknown data.
- Using a future (promise) object to represent a pending computation.
- In proof language (e.g. Idris, Lean), having a hole and inspecting the type of hole can help proving.
- Compiletime-runtime duality: a computation (or a check) can be in compile-time or runtime. It can be even before compile-time (code generation). It can be during running (e.g. JIT compilation). It can also be after first application run (e.g. profile-guided optimization).

Batched compuation vs immediate compuation:

- Immediately free memory vs GC
- ...

## Generalized View

Views in SQL databases are "fake" tables that are derived from other tables. 

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
- **Type contains viewing from binary data to information**.

[^bits_view]: Also: In hard disk, magnetic field is viewed as bits. In CD, the pits and lands are viewed as bits. In SSD, the electron's position in floating gate is viewed as bits. In fiber optics, light pulses are viewed as bits. In quantum computer, the quantum state (like spin of electron) can be viewed as bits. ......

More generally:

- The mapping between binary data and information is view. Information is bits+context. The context is how the bits are mapped between information.
- Abstraction involves viewing different things as the same things.

### Dynamically-typed languages also have "types"

Dynamically-typed languages also have "types". The "type" here is **the mapping between in-memory data and information**, and the **constraint of in-memory data**.

Even in dynamic languages, the data still has "shape" at runtime. The program only works with specific "shapes" of data. It cannot work with arbitrary data.

Mainstream languages often have relatively simpler and less expressive type systems. Some "shape" of data are complex and cannot be easily expressed in mainstram languages' type system (without type erasure). Dynamic languages are better in these cases: they avoid the shackle of unexpressive type system, and can get rid of syntax inconvenience related to type erasure (using type erasure in static languages require inconvenient things like type conversion).

## Invariant production, grow, and maintenance

Most algorithms use the idea of producing invariant, growing invariant and maintaining invariant:

- Initially, create invariant at the smallest scale (in the simplest case).
- Then incrementally make small invariant be larger, until the invariant become big enough to finish the task.
- For a data structure that has an invariant, every mutaiton to it need to maintain invariant.

Examples:

- Merge sort. Create sorted sub-sequence in smallest scale (e.g. two elements). Then merge two sorted sub-sequences into a bigger one, and continue. The invariant of sorted-ness grows up to the whole sequence.
- Quick sort. Select a pivot. Then partition the sequence into a part that's smaller than pivot and a part that's larger than pivot (and a part that equals pivot) [^quick_sort_equal_pivot]. By partitioning, it creates invariant $\text{LeftPartElements} < \text{Pivot} < \text{RightPartElements}$. By recursively creating such invariants until to the smallest scale (individual elements), the whole sequence is sorted.
- Binary search tree. It creates invariant $\text{LeftSubtreeElements} < \text{ParentNode} < \text{RightSubtreeElements}$. When there is only one node, the invariant is produced at the smallest scale. Every insertion then follows that invariant and then grows and maintains that invariant.


[^quick_sort_equal_pivot]: About the elements that equals the pivot, how exactly to treat them is implementation-specific. Some put them into the first partition. Some put then into another third partition.