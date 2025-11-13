---
date: 2025-10-25
tags:
  - Programming
  - Math
unlisted: true
---

# Deadlock, Circular Reference and Halting

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

## Priority inversion

In some real-time (or near-real-time) systems, important threads have higher priority than other thread. The thread scheduler tries to run higher-priority threads first. 

Priority inversion problem can make high-priority threads keep stuck, effectively similar to deadlock (although it's not strictly deadlock).

The common priority inversion problem involves 3 threads, with low/medium/high priorities respectively:

- The low-priority thread holds a lock.
- A high-priority thread tries to acquire lock. It cannot and wait for low-priority thread to release lock.
- Another medium-priority thread keeps running. When medium-priority thread runs, it occupies CPU cores so that low-priority thread cannot run. The high-priority thread's running now indirectly depend on medium-priority thread. If medium-priority thread keeps running, high-priority thread will never run.

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

In Golang, channels are not buffered by default, then producer waits for consumer. If you put a thing into a channel that no one will consume, it will wait forever. But changing the channel to buffered channel `make(chan string, 1)` will make the producer to not wait for consumer as long as buffer is not full, so deadlock is solved.

That lock-free deadlock can also be drawn as resource allocation graph. Channel is also a kind of node. `threadA` depends on `aToB` consuming value, `aToB` consuming value depends on `threadB` progress, `threadB` depends on `bToA` consuming value, `bToA` consuming value depends on `threadA` progress.

Note that Golang channel buffer must have a constant size limit. There are packages for unbounded channel ([chanx](https://github.com/smallnest/chanx)). However the memory may be not large enough if big bursts occurs. Use disk-backed event queue (e.g. Kafka) to get larger buffer capacity. 

Note that sometimes you need back pressure (stop producer when buffer is full) to improve stability (e.g. avoid using up memory) at the cost of blocking requests.

### Buffered channels can still deadlock

The previous deadlock can be solved by making channel buffered. However, buffering doesn't solve all lock-free deadlocks. For example:

```go
go func() {
    value := <- ch1
    ch2 <- "some result"
} ()

go func() {
    value := <- ch2
    ch1 <- "some result"
} ()
```

## Channel+Lock deadlock

Example:

```go
go func(m *sync.Mutex, c chan string) {
    m.Lock()
    defer m.Unlock()
    value := <-c
} (m, c)

go func(m *sync.Mutex, c chan string) {
    m.Lock()
    defer m.Unlock()
    c <- "some result"
} (m, c)
```

## Trap of select

### In Golang

For example, do some work with timeout, using channel and select:

```go
func doWorkWithTimeout(timeout time.Duration) (string, error) {
	ch := make(chan string)
	go func() {
		result := doWork()
		ch <- result // this blocks
        fmt.Println("doWork done")
	}()
	select {
	case result := <- ch:
		return result, nil
	case <- time.After(timeout):
		return "", errors.New("timeout")
	}
}
```

`select` will finish if either case gives a result. If it timeouts, `select` will finish by second case and never consume from `ch`. So the `ch <- result` will hang forever, causing **goroutine leak**. This can be fixed by making `ch` buffered.

Many memory leaks in Golang are caused by goroutine leak. Goroutine leak will also cause its task to never finish which can cause other bugs. If a leaked goroutine holds a lock, it will cause deadlock. If something waits for a leaked goroutine it will also deadlock.

### In async Rust

Async Rust has uncooporative cancellation.

Cooporative cancellation is that the code manually checks whether it should cancel and then exit by its own. Uncoorporative cancellation is that you forcefully stop some code from running.

Golang doesn't support uncooporative cancellation. Java doesn't allow you to kill a thread (Java has interrupt which is cooperative). OS APIs can kill a thread but it causes other problems (e.g. won't release a lock, won't cleanup a resource) so it shouldn't be used for cancellation.

In Rust, futures are not background tasks. Futures only progress when polled. There are two kinds of async cancellation in Rust:

- The future won't be polled, but it's not yet dropped. (This is prone to deadlock.)
- The future is dropped. It obviously won't be polled.

In either case, the async function won't continue running. It's called cancellation. Note that the already-done IO operations won't be cancelled (sent packets won't be magically withdrawn). The cancellation here just stops the async function from continuing.

`tokio::select` the value of non-selected cases will be dropped. `tokio::select` very different to Golang select. In Golang select, the non-selected cases won't consume data from channel. 

In `tokio::select`, for each case, you can:

- Pass an owned future. If that case is not selected, the future will be dropped. This cause uncorporative cancellation.
- Pass a borrow of future. If that case is not selected, `select` will proceed without further polling that future.
- Pass a `JoinHandle`.

[Tokio document 1](https://tokio.rs/tokio/tutorial/select#cancellation), [Tokio document2](https://docs.rs/tokio/latest/tokio/macro.select.html#cancellation-safety)

See also: [Making Async Rust Reliable - Tyler Mandry](https://tmandry.gitlab.io/blog/posts/making-async-reliable/)  [400 - Dealing with cancel safety in async Rust / RFD / Oxide](https://rfd.shared.oxide.computer/rfd/400)   [609 - Futurelock / RFD / Oxide](https://rfd.shared.oxide.computer/rfd/0609)  [FuturesUnordered and the order of futures](https://without.boats/blog/futures-unordered/) [Barbara gets burned by select - wg-async](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_gets_burned_by_select.html)

## Livelock

TODO

## Circular reference counting leak

Reference counting cause memory leak if there exists a cycle of strong references.

Reference counting is based on the assumption that: if there is no reference to one object, that object can be freed. It's based on local reference to object, not global reachability.

However, when there is a cycle, each object in cycle will be kept referced. But if that cycle is isolated, inaccessible to the program (not referencable from global variables and local variables), the object should be collected, but locally it's still referenced.

Tracing GC can collect them because tracing GC scans the whole object graph globally.

**But reference counting works locally**. If the cycle has length limit, such as limiting to 2-object cycles, checking cycle locally can still work. But in real applications the reference cycle can be arbitrarily large, so checking cycle locally won't reliably work. **Tracing GC works globally** so it can handle arbitrarily-large cycles.

The common solution is to use weak reference counting to cut cycle.

## Rust dislikes circular reference

TODO

See also [Linear Types One-Pager](https://blog.yoshuawuyts.com/linear-types-one-pager/)

## Lazy evaluation circular reference

### Infinite container

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

### Reverse state monad

In normal state monad, the new state is computed on old state. But in reverse state monad, the state flows backwards. Old state can be computed on new state. You can change the old state that's used in previous computation. This "magic" relies on lazy evaluation.

Definition of reverse state monad.

```haskell
newtype RState s a = RState { runRState :: s -> (a, s) }

instance Monad (RState s) where
  ...
  RState sf >>= f = RState $ \s ->
    let (a, past)   = sf future
        (b, future) = runRState (f a) s
    in (b, past)
```

Note that it has circular dependency: `a` depends on `future`, `future` depends on `a`. It can work due to lazy evalution (it doesn't always work).

Example usage:

```haskell
{-# LANGUAGE GeneralizedNewtypeDeriving #-}

import Control.Monad (ap)

newtype RState s a = RState { runRState :: s -> (a, s) }

instance Functor (RState s) where
  fmap f st = RState $ \s ->
    let (a, s') = runRState st s
    in (f a, s')

instance Applicative (RState s) where
  pure x = RState $ \s -> (x, s)
  (<*>) = ap

instance Monad (RState s) where
  return = pure
  RState sf >>= f = RState $ \s ->
    let (a, past)   = sf future
        (b, future) = runRState (f a) s
    in (b, past)

get :: RState s s
get = RState $ \s -> (s, s)

put :: s -> RState s ()
put s = RState $ \_ -> ((), s)

-- it modifies old state based on new state
modify :: (s -> s) -> RState s ()
modify f = RState $ \s -> ((), f s)

example :: RState Int String
example = do
  x <- get
  modify (* 2)
  y <- get
  return $ "Before: " ++ show x ++ ", After: " ++ show y

main :: IO ()
main = do
  let result = runRState example 2333
  putStrLn $ show result
```

It will output

```
("Before: 4666, After: 2333",4666)
```

Note that reverse state monad is still in a normal Haskell program. It cannot magically "make time flow backwards". It also cannot magically solve equations to compute old state based on new state. If new state relies on old state it will just deadloop.

### Memory leak caused by lazy evaluation

For example, if you have a large list of integers and you compute sum of it. If the sum value is not used, the list will be kept in memory for future evaluation. The list can only be freed after the sum is evaluated. If the sum result will never be evaluated, it memory leaks.

The two cases:

- When the input of computation is smaller than output of computation (e.g. generate a big list), lazy evaluation can temporarily save memory.
- When the input of computation is larger than output of computation (e.g. sum a big list), lazy evaluation can waste or leak memory.

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

For a Turing machine, if the states are nodes, then each iteration of running is an edge, jumping from old state to new state. It forms a graph. Not halting is having a cycle in that graph, and that cycle is reachable from beginning state.

### Nothing can be analyzed?

Halting problem and Rice's theorem says that we cannot reliably analyze arbitrary Turing-complete programs. 

But it doesn't mean nothing can be analyzed. It just means we cannot analyze arbitrary program. There are many analyzable programs. If the program is simple enough, it obviously can be analyzed. If a program use proper encapsulation, analyze can be simplified by only focusing on one part of program. 

If we apply some constraints, to make it not Turing complete but still expressive, then halting can be ensured, while being still useful enough, like in Lean.

Rust has a lot of constraints to limit sets of programs to an analyzable subset, so it can analyze about memory safety and thread safety.

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

Strictly speaking, Turing complete requires infinitely large memory, so all practical computers and languages don't satisfy strict Turing complete. Apart from memory constraint, blockchain applications often consume fee (gas) in each step of execution, but Turing complete requires unlimited execution steps, so they also not strictly Turing complete.

## Ethernet loop

In the raw form of Ethernet, there is no communication between switches, so one switch cannot know the whole network topology. 

How raw form of Ethernet do routing:

- When it receive a packet from one interface, it knows that the source MAC address correspond to that interface. It's stored into MAC address table. This is self-learning.
- When it doesn't know which interface a MAC address correspond to, it broadcasts packet to all other interfaces, except for the interface that the packet comes from. 

It works fine when there is no cycle in network topology. But when there is a cycle, the broadcast will come back to the same switch but from another interface. It not only messes up the self-learning of MAC address table, but can also cause the switch to broadcast the same packet again, and again, causing boradcast storm. 

This is solved in spanning tree protocol, where switches share topology information with each other, then break the loop.

## Circular reference in math

### Circular proof

[Circular proof](https://en.wikipedia.org/wiki/Circular_reasoning): if A then B, if B then A. Circular proof is wrong. It can prove neither A nor B.

### Russel's paradox

The set that indirectly includes itself cause [Russel's paradox](https://en.wikipedia.org/wiki/Russell%27s_paradox).

Let R be the set of all sets that are not members of themselves. R contains R deduces R should not contain R, and vice versa. Set theory carefully avoids cirular reference.

### Y combinator

Raw lambda calculus only does "string substitution" [^string_substitution] and doesn't allow a function to directly reference itself. But it can be workarounded by Y combinator:


$$
Y = \lambda f . (\lambda x . f (x \ x)) (\lambda x . f (x \ x))
$$

[^string_substitution]: Lambda calculus substitutes in a way that avoids naming conflict. It's not simple string substitution.

Written in TypeScript:

```typescript
type Func<Input, Output> = (input: Input) => Output;

type SelfAcceptingFunc<Input, Output> = (s: SelfAcceptingFunc<Input, Output>) => Func<Input, Output>;

function Y<Input, Output>(
    f: (s: Func<Input, Output>) => Func<Input, Output>
): Func<Input, Output> {
    // temp = λ x . f (x x)
    let temp: SelfAcceptingFunc<Input, Output> = 
        (x: SelfAcceptingFunc<Input, Output>) => f (input => x(x)(input));
        // Note: cannot write f(x(x)), it will deadloop
    return temp(temp);
}

const factorial = Y((f: (a: number) => number) => (n) => n > 1 ? n * f(n - 1) : 1);

console.log(factorial(4));
```

Note that the type of Y combinator requires self-reference, although Y combinator's expression itself don't require self-reference.


### Gödel's incomplete theorem

Firstly encode symbols, statements and proofs into data. The statements that contain free variables (e.g. x is a free variable in "x is an even number") can also be encoded (it can represent "functions" and even "higher-order functions").

Specifically, Gödel encodes symbols, statements and proofs into integer, called Gödel number. There exists many ways of encoding symbols/statements/proofs as data, and which exact way is not important. For simplicity, I will treat them all as data, and ignore the conversion between data and symbol/statements/proofs.

`is_proof(theory, proof)` allows determining whether a proof successfully proves a theory. 
  
Then `provable(theory)` is defined as whether there exists a `proof` that satisfies `is_proof(theory, proof)`.

Unprovable is inverse of provable: `unprovable(theory) = !provable(theory)`

Let `H(x) = unprovable(x(x))`. Then let `G = H(H) = unprovable(H(H)) = unprovable(G)` (it uses the same form as Y combinator), which creates a self-referencial statement: `G`  means `G` is not provable. If `G` is true, then `G` is not provable, then `G` is false, which is a paradox.

The `x(x)` is symbol substitution. replacing the free variable `x` with `x`, while avoid making two different variables same name by renaming when necessary. 



## Error of error of error...

> An error rate can be measured. The measurement, in turn, will have an error rate. The measurement of the error rate will have an error rate. The measurement of the error rate will have an error rate. 
> 
> We can use the same argument by replacing "measurement" by "estimation" (say estimating the future value of an economic variable, the rainfall in Brazil, or the risk of a nuclear accident). 
> 
> What is called a regress argument by philosophers can be used to put some scrutiny on quantitative methods or risk and probability. The mere existence of such regress argument will lead to two different regimes, both leading to the necessity to raise the values of small probabilities, and one of them to the necessity to use power law distributions.
> 
> \- N. N. Taleb, [Link](https://www.fooledbyrandomness.com/notebook.htm)


## Reference

[Understanding Real-World Concurrency Bugs in Go](https://songlh.github.io/paper/go-study.pdf)

[Weakening Cycles So That Turing Can Halt](https://pling.jondgoodwin.com/post/weakening-cycles/)

[A Universal Approach to Self-Referential Paradoxes, Incompleteness and Fixed Points](https://arxiv.org/pdf/math/0305282)

