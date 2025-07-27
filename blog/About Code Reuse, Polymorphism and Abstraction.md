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

| Regularization                                                                               | De-regularization                                                             |
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



Why we sometimes specialize instead of generalizing:

- Generalization introduces new concepts and **adds cognitive load**. Sometimes, not adding these is better, depending on how useful the abstraction is.
- A new requirement can break the assumption or regularity that the generalization is based on. **New exceptions break generalization**.

About leaky abstraction: Abstraction aim to hide details and make things simpler. But some abstractions are **leaky**: to use it correctly you need to understand the details that it tries to hide. The more leaky an abstraction is, the less useful it is.

If a new requirement follows the regularity that the abstraction uses, then the abstraction is good and makes things simpler.

But when the new requirement change breaks the regularity, then abstraction hinders the developer. The developer will be left with two choices:

- De-regularize the abstraction and do the change accordingly. (And create new abstractions that follow the new regularity. This is refactoring.)
- Add special case handlings within the current abstraction. The exceptions can **make the previously unrelated things related again** (break orthogonality), **increasing (accidental) complexity**. It will often involve new boolean flags that control internal behavior, weird data relaying, new state sharing, new concurrency handling, etc.

**Every abstraction makes some things easier AND make other things harder. It's a tradeoff.**

> every game engine has things they make easier and things they make harder. working exclusively with one tool for a long time makes your brain stop even considering designs that fall outside the scope of that tool. it can make it feel like the tool doesnt have limits
> 
> \- Tyler Glaiel, [Link](https://x.com/TylerGlaiel/status/1880340558767702377)

## "Simple" requirements that are hard to implement

Sometimes a seemingly simple requirement is actually hard to implement, because the new requirement breaks existing abstraction.

Examples:

### Change of data modelling

- You use user name as id of user. But a new requirement comes: the user must be able to change the user name. 
  (Using name as id is usually a bad design because it's incompatible with name changing.)
- In a game, if an entity dies, you delete that entity. But a new requirement comes: a dead entity can be resurrected by a new magic.
  To implement that, you cannot delete the dead entity and you need to add dead entity into data modelling. For example, add a boolean flag of whether it's living, and check that flag in every logic of living entity.
- Your app supports one language. And the event log is recorded using simple strings. But a new requirement comes: make the app support multiple languages. The user can switch language at any time and see the event log in their language.
  To implement that, you cannot store the text as string and should store the text as translatable template.


### Change to dataflow and source-of-truth

- You built a singleplayer game. All game logic runs locally. All game data are in memoery and you manually load/save from file. But a new requirement comes: make it multiplayer.
  In singleplayer game, the in-memory data can be source-of-truth, but in multiplayer the server is source-of-truth. Every non-client operation now requires packet sending and receiving. What's more, to reduce visible latency, the client side game must guess future game state and correct the guess from server packets (add rollback mechanism).
- You built a todo list app. All data are loaded from server. All edits also go through server. But a new requirement comes: make the app work offline and sync when it connects with internet.

### Adding edge cases into simple logic

- You have a permission system where different functionalities form a tree: if one user have the permission of parent node then the user has the permissions of the child nodes. But a new requirement comes: one functionality moves category, so that node's parent changes. You must also keep the existing users' permissions the same as before.
- Some functionality require some permission. You use user token to authenticate. But a new requirement comes: allow non-logged-in users access a part of the functionality.
- A new requirement comes: add bot as a new type of user, and the bot user has different authentication logic than normal user.

### Adding a lot of flexibility into the system

- You have a fixed workflow. A new requirement comes: allow the user to configure and customize the workflow.
  (Developing specially for each enterprise customer is actually easier than creating a configurable flexible "rules engine". The custom "rules engine" will be more complex and harder to debug than just code. [The Configuration Complexity Clock](https://mikehadlow.blogspot.com/2012/05/configuration-complexity-clock.html))
- Allow formatting like bold and color in username.
- Adding a plugin system.

### Working on full data to working on partially known data

- You built a data visualization UI. Originally, it firstly loads all data from server then render. But when the data size become huge, loading becomes slow and you need break the data into parts, dynamically load parts and visualize loaded parts.
- A game has loading screen when switching scene. A new requirement comes: make the loading seamless and remove the loading screen.
- You load all data from database and then compute things using programming language. One day the data become so big that cannot be held in memory or it exceeds database query limit. You need to either 
  - Load partial data into memory, compute separately and then merge the result, or
  - rewrite logic into SQL and let database compute it 

### Migrating to incompatible APIs

Migrating the libraries/framework/OS/database/game engine/other components whose API is not compatible with the old. From a normal user's perspective, the migration does not add any new feature, takes a long time and can introduce new bugs.

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

