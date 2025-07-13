---
date: 2025-07-13
---

<!-- truncate -->

## Code reuse mechanisms

![](Polymorphism_for_Code_Reuse.drawio.svg)



| Code reuse mechanism                                                  | A                    | P and Q                                                       | R                                        | P' and Q'                                                                                 | Use Arg                                                                  |
| --------------------------------------------------------------------- | -------------------- | ------------------------------------------------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| Extract function                                                      | Execution code block | Values                                                        | Function                                 | Values (converted to argument type)                                                       | Pass argument                                                            |
| Extract higher-order function (closure, lambda expression)            | Execution code block | Execution code blocks (can take outer arguments)              | Function, taking function as argument    | Closure function (can capture values)                                                     | Call function argument                                                   |
| OOP inheritance, Interface, dynamic trait (subtype polymorphism)      | Execution code block | Execution code blocks (can take outer arguments)              | Code that use supertype object reference | Objects of different subtypes (can carry different fields), overriding polymorphic method | Call polymorphic method                                                  |
| Function overloading, static trait, typeclasses (ad-hoc polymorphism) | Execution code block | Execution code blocks (possibly dealing with different types) | Function                                 | Type or typeclass (usually inferred)                                                      | Call overloaded function / call trait function / call typeclass function |
| Generic type (parametric polymorphism)                                | Type definition      | Types                                                         | Generic type                             | Type parameters                                                                           | Use type parameter                                                       |
| Generic function (parametric polymorphism)                            | Execution code block | Types                                                         | Generic function                         | Type parameters (usually inferred)                                                        | Use type parameter                                                       |
| Type erasure                                                          | Code block           | Values in different types, or works with different types      | Function (can be constructor)            | Value of top type, with type information at runtime                                       | Pass value, check type, cast type, reflection, etc.                      |
| Duck typing (row polymorphism)                                        | Execution code block | Object field accesses, method calls                           | Field access or method call by name      | Different values with common fields or methods                                            | Use common fields/methods by name                                        |
| Macro                                                                 | Code fragment        | Code fragments                                                | Macro                                    | Code fragments                                                                            | Use macro argument                                                       |


## Regularize and de-regularize

Extracting function is regularization, while inlining function is de-regularization. Extracting function turns duplicated code into a shared function, and inlining turns shared function into duplicated code.

Extracting function can utilize regularity and simplifies code. When the requirement changes and the regularity doesn't hold, inlining is the way of de-regularize function call and allow easier code changing. If the requirement changes but you don't inline the unsuitable function, you may add boolean flags to that function, introducing accidental complexity.

| Regularize                                                                                   | De-regularize                                                                 |
| -------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| Extract function                                                                             | Inline function                                                               |
| Extract generic parameter                                                                    | Inline generics / type erasure                                                |
| Encapsulate                                                                                  | Remove encapsulation                                                          |
| Extract higher-order function                                                                | Inline dynamic dispatch                                                       |
| Extract polymorphic method call                                                              | Inline dynamic dispatch                                                       |
| Use cross-platform frameworks                                                                | Develop separately for different platforms                                    |
| Adding flexibility                                                                           | Removing flexibility                                                          |
| Generalize                                                                                   | Specialize                                                                    |
| Abstracted "clever" code                                                                     | Duplicated "dumb" code                                                        |
| Easier to implement requirements that follow regularity.                                     | Harder to implement requirements that follow regularity. (duplicated changes) |
| Harder to implement requirements that breaks regularity. (add complex special-case handling) | Easier to implement requirements that breaks regularity.                      |

Why we specialize instead of generalize sometimes:

- Generalization introduces new concepts and **adds cognitive load**. Sometimes, not adding these is better, depending on how useful the abstraction is.
- The requirement can break the assumption or regularity that the generalization is based on. **New exceptions break generalization**.

About leaky abstraction: Abstraction aim to hide details and make things simpler. But some abstractions are **leaky**: to use it correctly you need to understand the details that it tries to hide. The more leaky an abstraction is, the less useful it is.

If a new requirement follows the regularity that the abstraction uses, then the abstraction is good and makes things simpler. 

But when the new requirement change breaks the regularity, then abstraction hinders the developer. The developer will be left with two choices:

- De-regularize the abstraction and do the change accordingly. (And create new abstractions that follow the new regularity. This is refactoring.)
- Add special case handlings within the current abstraction. This will make things more complex. The exceptions will break orthogonality, making the previously unrelated things related again. It will often involve new boolean flags that control internal behavior, weird data relaying, new state sharing, new concurrency handling, etc.

**Every abstraction makes some things easier AND make other things harder. It's a tradeoff.**

> every game engine has things they make easier and things they make harder. working exclusively with one tool for a long time makes your brain stop even considering designs that fall outside the scope of that tool. it can make it feel like the tool doesnt have limits
> 
> \- Tyler Glaiel, [Link](https://x.com/TylerGlaiel/status/1880340558767702377)


## Simple interface = hardcoded defaults = less customizability

Real world is complex. Doing things require making decision on a lot of details.

If some tool has a simple interface, it must have hardcoded a lot of detail decisions inside. If the interface exposes these detail decisions, the interface won't be simple.

This even applies if you treat LLM as an abstraction layer. When you write a vague prompt and LLM generates a whole application/feature for you, the generated code contains many optionated detail decisions that's made by LLM, not you. (Of course you can prompt the LLM to customize a detail later. the point is that LLM make decisions for you to fill the unspecified things in the prompt).

## Orthogonality: separate things that can combine easily

The software usually need to do multi-case handling: do different computation to different kinds of information.

In non trivial cases, the software need to do different computation to **different combinations of information**. Making the handling of them more **orthogonal** will make that simpler. 

In the context of programming, **orthogonality** can be think as **two different things can be combined without interfering each other**. **Orthogonality also mean unrelatedness** Non-orthogonal means when two things combine, they interfere with each other. **Orthogonality also means logical isolaton and freedom to combine, allowing combinatory explosion to be easily handled.**

If different parts of decision-making cannot be orthogonally destructured, then the software will become complex.

Splitting a complex operation into multiple steps makes it more orthogonal. Doing the work of multiple different steps in one step adds complexity and reduces orthogonality. 

The reality is usually less perfect than theories. Often two things are mostly orthogonal but has some non-orthogonal edge cases. If the edge cases are not complex, and are few, then the program can be designed around the fact that the two things are mostly orthogonal, and add the special case handling to two modules. However, if there are many special cases, or some special cases are very complex to handle, then the two modules are very non-orthogonal and should be re-designed.

### Reducing fake orthogonality

Sometimes the interface allow passing two orthogonal options, but it actually does not support some combinations of options. This is fake orthogonality (seems orthogonal in interface but actually doesn't).

Sum types are useful for avoiding the invalid combinations of data, reducing fake orthogonality. They can help correctness by stopping the invalid combinations of data from being created.

Another case is that the software provides orthogonality in interface, and actually supports all combinations of options (including many useless option combinations), but the implementation is non-orthogonal, then the implementaiton will face **combinatory explosion**. Limiting the supported combinations in interface is better.

> If you consider it as a library, you can use Windows linker functionality X in combination with Unix linker functionality Y, but there was no precedent for what the linker should behave in such a case. Even worse, in many situations, it was not obvious what would be the “right” behavior. We spent a lot of time discussing to define the semantics that would make sense for all possible feature combinations, and we carefully wrote complex code to support all targets simultaneously. However, in hindsight, this was probably not a good way to spend time because no one really wanted to use such hypothetical feature combinations. lld v1 probably didn't have any real users.
>
> \- [My story on “worse is better”](https://www.sigbus.info/worse-is-better)

