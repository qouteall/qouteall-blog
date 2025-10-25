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



## Lock-free deadlock




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

