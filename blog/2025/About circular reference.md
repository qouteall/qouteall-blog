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

It has two kinds of edges:

- A lock points to a thread. Assignment edge. It denotes that the thread already holds the lock. The lock's release depends on the thread's progress.
- A thread points to a lock. Request edge. It denotes that the thread try to hold the lock. The thread's progress depends on acquiring the lock.

All two kinds of edges represent "depending on" relation.

When that graph forms a cycle, deadlock occurs.

For non-reentrant locks, deadlock can happen with only one lock and one thread.

## Lock-free deadlock

Deadlock can also happen when there is no explicit lock. I call it **lock-free deadlock**. (That naming is inspired by "[serverless servers](https://vercel.com/blog/serverless-servers-node-js-with-in-function-concurrency)", "constant variables" and "[unnamed namespaces](https://www.learncpp.com/cpp-tutorial/unnamed-and-inline-namespaces/)".)

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

## Reference counting circular leak



## Lazy evaluation infinite container


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

## Non-Turing-Complete programming languages



## Y combinator


Although Y combinator's own expression doesn't require self-reference, the type of Y combinator requires self-reference to express.

## Circular reference in math



### Circular proof

[Circular proof](https://en.wikipedia.org/wiki/Circular_reasoning): if A then B, if B then A. Circular proof is wrong. It can prove neither A nor B.

### Russel's paradox

The set that indirectly includes itself cause [Russel's paradox](https://en.wikipedia.org/wiki/Russell%27s_paradox).

Let R be the set of all sets that are not members of themselves. R contains R deduces R should not contain R, and vice versa. Set theory carefully avoids cirular reference.

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

