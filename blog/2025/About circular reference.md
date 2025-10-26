---
date: 2025-10-25
tags:
  - Programming
  - Math
unlisted: "true"
---

# About circular reference

<!-- truncate -->

## Deadlock

Deadlock can be understood via **resource allocation graph**.

It has two kinds of nodes:

- Threads. (Sometimes called processes). Often drawn as circle.
- Locks. Often drawn as square.

Its edges represent **dependency**. A point to B means A depends on B. Specifically it has two kinds of edges:

- Assignment edge. A lock points to a thread. It denotes that the thread already holds the lock. The lock's release depends on the thread's progress.
- Request edge. A thread points to a lock. It denotes that the thread try to hold the lock. The thread's progress depends on acquiring the lock.

When that graph forms a cycle, deadlock occurs.

For non-reentrant locks, deadlock can happen with only one lock and one thread.

## Lock-free deadlock

Deadlock can also happen when there is no explicit lock. I call it **lock-free deadlock**. (That naming is inspired by "[serverless servers](https://vercel.com/blog/serverless-servers-node-js-with-in-function-concurrency)", "constant variables" and "[unnamed namespaces](https://en.cppreference.com/w/cpp/language/namespace.html#Unnamed_namespaces)".)

A simple Golang program showing lock-free deadlock:

```go
func threadA(aToB chan string, bToA chan string, wg *sync.WaitGroup) {
	defer wg.Done()

	fmt.Println("A: Starting and attempting to send message...")

	aToB <- "Hello from A"
	fmt.Println("A: Successfully sent message.")

	msg := <-bToA
	fmt.Printf("A: Received message: '%s'\n", msg)
}

func threadB(aToB chan string, bToA chan string, wg *sync.WaitGroup) {
	defer wg.Done()

	fmt.Println("B: Starting and attempting to send message...")

	bToA <- "Hello from B"
	fmt.Println("B: Successfully sent message.")

	msg := <-aToB
	fmt.Printf("B: Received message: '%s'\n", msg)
}

func main() {
	var wg sync.WaitGroup

	aToB := make(chan string)
	bToA := make(chan string)

	wg.Add(2)

	go threadA(aToB, bToA, &wg)
	go threadB(aToB, bToA, &wg)

	wg.Wait()

	fmt.Println("Finished")
}
```

Output

```
A: Starting and attempting to send message...
B: Starting and attempting to send message...
fatal error: all goroutines are asleep - deadlock!
```

(The WaitGroup is to make the main thread wait for two goroutines. Without it, the main thread can exit normally despite goroutine waiting.)

In Golang, the channels are by-default not buffered. Sending data into channel will block until the data is received. Change the channel to buffered channel `make(chan string, 1)` can solve that deadlock.

That lock-free deadlock can also be drawn as resource allocation graph. Channel is also a kind of node. `threadA` depends on `aToB` consuming value, `aToB` consuming value depends on `threadB` progress, `threadB` depends on `bToA` consuming value, `bToA` consuming value depends on `threadA` progress.

## Reference counting circular leak

Reference counting is based on the assumption that: if there is no reference to one object, that object can be freed. It's based on local reference to object, not global reachability.

However, when there is a cycle, each object in cycle will be kept referced. But if that cycle is isolated, inaccessible to the program (not referencable from global variables and local variables), the object should be collected, but locally it's still referenced.

Tracing GC can collect them because tracing GC scans the whole object graph globally.

But reference counting works locally. If the cycle has length limit, such as limiting to 2-object cycles, checking cycle locally can still work. But in real applications the reference cycle can be arbitrarily large, so checking cycle locally won't reliably work. Only by global traversing can these cycles be reliably handled.

## Lazy evaluation infinite container

Haskell is a pure functional language where there is no mutable state. Haskell also has lazy evaluation.

Without mutable state and lazy evaluation, reference cycle cannot be created. Because new values can only contain the existing values when creating it.

Haskell has lazy evaluation so it allows circular reference. Example:

```haskell
ones :: [Integer]
ones = 1 : ones
```

It will be an infinte list of `1`s.

It can also be seen as a tree structure expand infinitely, with no circular reference.

Similarily, this

```haskell
from :: Integer -> [Integer]
from n = n : from (n + 1)
```

creates an infinite list of increasing integers from `n`.

Although the conceptual list is infinitely large, due to lazy evaluation, only the needed places need to be computed and stored into memory. They can be used as long as computation don't use the whole list.

## Halting problem

[Halting problem](https://en.wikipedia.org/wiki/Halting_problem) is proved impossible to solve, by using circular reference.

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

(Note that halting problem cares about whether program halts in finite time, but don't care about how long it takes. A program that need to run 1000 years to complete still halts.)

## Non-Turing-Complete programming languages

SQL is not Turing-complete when not using recursive common table extension (`with recursive ...`) and other procedural extensions (e.g. `while`).

The proof languages, like Lean and Idris, are not Turning-complete. Because a valid proof require the corresponding program to halt. They have special mechanisms to ensure that program eventually halts. 

The proof languages describe both program and proof, according to [Curry–Howard correspondence](https://en.wikipedia.org/wiki/Curry%E2%80%93Howard_correspondence).

- The propositions, like `1 = 1`, `1 + 1 = 2`, correspond to types. If you can obtain a value of that type, you can prove it (what's inside the value is not important for proof, only the type is important).
- The function types, like `(x: Integer) -> (x + 0) = x` represent a proposition `x + 0 = x`. Pass argument `1` to that function, you get `(1 + 0) = 1`
- If you can write a program of that type, without using any side effect , and the program halts, then that theorem corresponding to type is proved.
- A proof can rely on another proof. For example `((x + 1) + 1) + y = 1 + ((x + 1) + y)` relies on `(x + 1) + y = 1 + (x + y)`, recursively "call function" until the basic case `(0 + 1) + y = 1 + (0 + 1)`
- To prove theorem corresponding to type `X`, run a program that return `X`. When the program normally halts and output a value of type `X` then it's proved. 
- If the program don't halt, then value of type `X` can never be obtained.
- That program should not use any side effect (mutation, external IO, random, etc.).
- In reality, you just need to ensure the program halts and don't need to really execute the program.

## Circular reference in math

### Circular proof

[Circular proof](https://en.wikipedia.org/wiki/Circular_reasoning): if A then B, if B then A. Circular proof is wrong. It can prove neither A nor B.

### Russel's paradox

The set that indirectly includes itself cause [Russel's paradox](https://en.wikipedia.org/wiki/Russell%27s_paradox).

Let R be the set of all sets that are not members of themselves. R contains R deduces R should not contain R, and vice versa. Set theory carefully avoids cirular reference.

## Y combinator

Although Y combinator's own expression doesn't require self-reference, the type of Y combinator requires self-reference to express.


### Gödel's incomplete theorem

Firstly encode symbols, statements and proofs into data. The statements that contain free variables (e.g. x is a free variable in "x is an even number") can also be encoded (it can represent "functions" and even "higher-order functions").

Specifically, Gödel encodes symbols, statements and proofs into integer, called Gödel number. There exists many ways of encoding symbols/statements/proofs as data, and which exact way is not important. For simplicity, I will treat them all as data, and ignore the conversion between data and symbol/statements/proofs.

`is_proof(theory, proof)` allows determining whether a proof successfully proves a theory. 
  
Then `provable(theory)` is defined as whether there exists a `proof` that satisfies `is_proof(theory, proof)`.

Unprovable is inverse of provable: `unprovable(theory) = !provable(theory)`

Let `H(x) = unprovable(x(x))`. Then let `G = H(H) = unprovable(H(H)) = unprovable(G)` [^godel_substitution], which creates a self-referencial statement: `G`  means `G` is not provable. If `G` is true, then `G` is not provable, then `G` is false, which is a paradox.

The `x(x)` is symbol substitution. replacing the free variable `x` with `x`, while avoid making two different variables same name by renaming when necessary. 

It's also similar to Y combinator: `Y = f -> (x -> f(x(x))) (x -> f(x(x)))`. In that case `f = unprovable`, `H = x -> f(x(x))`, `Y(f) = H(H)`, `Y(f)` is a fixed point of `f`: `f(Y(f)) = Y(f)`. `G = Y(f)`, `f(G) = G`

---

Related:

[Weakening Cycles So That Turing Can Halt](https://pling.jondgoodwin.com/post/weakening-cycles/)

[A Universal Approach to Self-Referential Paradoxes, Incompleteness and Fixed Points](https://arxiv.org/pdf/math/0305282)

