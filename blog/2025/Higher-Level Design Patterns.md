
- Computation-data duality.
- Mutation-data duality.
- Move computation between stages.
- View.
- Invariant production, grow, and maintenance.

## Computation-data duality

Code (machine code, bytecode, programming language code, etc.) can represent computation. Code already is data.

Turn action into data. Replace raw execution code with an interpreter.

By turning action into code, the raw action code is split into two parts: the one generating action and the one executing action. (It's also related to moving computation stages)

Some benefits:

- Inspection: Explicit execution state is easier to inspect and display (than stackframe and code execution position).
- Serialization: Explicit execution state can be serialized and deserialized, thus be stored to database and sent across network.
- Suspension: Explicit execution state allows temporarily suspending execution and resume it later. Suspending thread is harder (to avoid suspending the whole process, you need to run it in a separate thread) and less efficient (current OS is not optimized for one million threads). (Turning execution state as data. It's related to mutation-data duality)
- Modification: Explicit execution state can be modified. It makes cancellation and rollback easier. While modifying running code control flow is harder.
- Forking: Allows forking control flow, which can simplify some logic (not common in practice).

The idea is to use data to represent computation (includes logic and strategy). 

- Higher-order functions. A function value is a piece of code with data that can be called. Functional programming encourage building complex logic by composing simple building blocks.
- Control-flow-state-machine duality. A suspendable control flow (async, generator) can be turned into a state machine. (Restate does serialization of control flow)

## Mutation-data duality

Mutation can be represented as data. Data can be interpreted as mutation.

- Instead of just doing in-place mutation, we can enqueue a command to do mutation later. The command is then processed to do actual mutation.
- In a transactional database, modifying things in a transaction adds mutation records instead of just modify in-place. The mutation is persisted when transaction commits. With snapshot isolation, mutation in one transaction is invisible to another except for locking, but mutation in current transaction is visible to own.
- In a client-side GUI application, modifying a thing requires sending a request to the server. That request is data. The server's response is also data, which may confirm or deny the modification. The client can display modified data before server responds to reduce latency, but need to rollback temporary changes when the server denies the modification (in multiplayer cases, one player's modification can invalidate another player's modification).
- In Raft, all modification are in the log, and the latest state is derived from the log.

Instead of doing in-place modification, we can:
- Defer mutation. Put updates in some queue and mutate in a deferred way.
- Keep both the new state and the old states (copy-on-write, read-copy-update).
- Express the latest state as a view of old state + mutations.
- Two alternate buffers. Example: when simulating Conway's game of life, you need two buffers. Just having one buffer and doing in-place modification will give wrong result, as each simulation step requires the previous state, and in-place modification replaces some previous states with new states. Sometimes a computation requires old state, and sometimes it requires new state, depending on exact requirements.
- Layered filesystem (in Docker).

The benefits:
- Easier to debug and track mutations, because mutations are explicit data, not implicit history.
- Avoid data race issues under parallelism (copy-on-write, reading old snapshot, make mutation command processing sequential).
- Make rollback and undo easier. Rollback is common in transactional databases (transaction can rollback), editing software (need to support undo) and multiplayer client (one player's edit can be invalidated by another player or other server-driven events, but the client edit need to display immediatey).
- Make forking and tree-search easier. Sometimes we need to inspect result of different mutations applied to the same data, creating "forks" or "parallel universes". The modifications can form a tree (or a graph), and we can search on it. Expressing the new state as a view of old state + mutations can reduce copying cost.
- Allow merging mutations in distributed systems (like Git).

In some places, we have a new state and need to compute the mutation (diff). Examples: Git, React. In Git and React, there is one commonality that the state itself is outside of their control and calculating the diff is how they "sync" the mutation to other systems (sync change to DOM in React, sync changes to other machines in Git).

## Move computation between stages

Partial computation: only compute some parts of the data, and keep the structure of the whole computation:

- In lazy evaluation, the unobserved data is not computed.
- In multi-stage programming, some data are fixed while some data are unknown. The fixed data can be used for optimization. It can be seen as runtime constant value folding. JIT can be seen as treating bytecode as runtime constant and fold them in interpreter code.
- Replacing a value with a function that produces value or AST helps handling the currently-unknown data.
- In proof language (e.g. Idris, Lean), having a hole and inspecting the type of hole can help proving.
- 

## View



## Invariant production, grow, and maintenance

