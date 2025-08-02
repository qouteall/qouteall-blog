---
date: 2025-07-06
tags:
  - Programming
---

# Pitfalls in API usability

<!-- truncate -->

Here API means the generalized concept of "API":

- Instruction set is the "API" of CPU. Machine code invokes the "API" of CPUs.
- Source code invokes the "API" of programming languages.
- Functions and types are API.
- Networking protocols (IP, TCP, UDP, HTTP, etc.) are the "API" of the internet. Restful APIs.
- Data formats and configuration formats are also "API".
- All the contracts and protocols between different parts of software/hardware are in the broader sense of "API".

With that broader sense of API, **all programming revolves around using "APIs"** (and creating "APIs"). 

API usability is important to developer productivity.

## Pitfalls in API usability

- Missing documentation details about exact format of input/output data or missing examples. The document writer, under **curse of knowledge**, may assume the user know, but most users don't know.
- Doesn't provide example usages. Examples are valuable because **a working example cannot omit details**. Without detailed documentation, developers usually test the API manually to figure out details. **Tweaking (tinkering) a working example makes learning more proactive and efficient**.
- Is very hard to do manual testing. No simple REPL. Cannot easily setup virtual environments. Cannot easily take and load snapshots. Cannot call from simple commands. Cannot easily undo mistakes made in testing. Cannot easily use curl to test a Restful API.
- Lacking debugging and visualization tools. Doesn't allow easily check internal state or intermediary data. Doesn't allow easy profiling. 
  An example is using efficient binary data format instead of text format, but lack tools to inspect and edit the binary data (one main advantage of text-based data format is that it's easy to inspect and edit without special tools for the format).
- Behavior is unintuitive, causing developers to easily misunderstand the behavior. This can also happen if the behavior deviates from the similar APIs of mainstream tools, when most developers are familiar with mainstream tools. One example is yaml require a space after colon (different to JSON). Another example is CSS layouting.
- Missing documentation telling the **traps** (wrong ways of using the API).
- When the API is used wrongly, silently do nothing (**fail-silent**) or do unexpected things (undefined behavior), without giving error. An example is memory management in memory-unsafe languages (already improved by tools such as valgrind). Another example is that a wrong spelling field name in a JSON config file makes the config ineffective, without giving error message because JSON parsers usually ignore unknown fields.
- **No safety net** preventing wrong usage of API. The common example is memory management in memory-unsafe languages (C/C++). Another common example is data race.
- **Abstraction leakage**. You only know how to correctly use it if you understand the implementation detail. The abstraction fails to hide complexity.
- The API changed between versions incompatibly. The online materials may be outdated and LLMs are trained with outdated material.
- Doesn't explicitly tell that some configuration is unused or not effective. (Example: for two sets of configurations, where one overrides another, changing the overridden one has no effect.)
- Error messages is silently put into another place (can only check using a special command or a special function call, or in a special log file). Beginners usually don't how where to see the error message.
- Error message is vague and doesn't tell which thing is wrong. Example: only provide an error code that correspond to many kinds of errors. 
  Sometimes it's caused by not retaining enough runtime metadata. It cannot output useful error message because the relevant information is missing at runtime.
- Doesn't tell error early. Only tell error if some functionality is used. This may make some configuration bugs become unnoticed until some condition is met. 
- Doesn't tell error in the correct stage of computing. A wrong configuration of stage 1 may not give error in stage 1, but gives error in stage 2 when stage 2 processes invalid data from stage 1, which make the error message more obsecure because the context in stage 1 is lost.
- The tool does too many "magic" under the hood. The API seems simple but is actually complex. The "magic" sometimes make things more convenient, but sometimes cause unwanted behavior.
  - Try to use heuristics to "fix error". This makes the true error hidden and not fixed (make the app eventually accumulate many errors unnoticed). The heuristics cannot fully fix the error and malfunction in some edge cases.
  - Another example is layouting in CSS. Most layout-related attributes in CSS are very versatile. Each attribute usually have many side effects. CSS aims to make layout work with very few CSS attributes, but result in a complex system that's hard-to-understand.
- A convenience feature causes security vulnerability. (e.g. some JSON libraries store class name in JSON to support polymorphic objects, but trusting class name from user is insecure.)
- Too many downstream errors hiding the root error. 
  An example is log spam in log file, where only the first error is meaningful and all subsequent spam errors are side-effects of the first error. 
  In C++ if you use some STL container wrongly there may be a spam of compiler error that's in STL code, hiding the root error.
- The API becomes complex to accomodate special custom usages, making common simple usage harder and more complex.
- The API is too simple to accomodate special custom usage. Doing special custom usage requires complex and error-prone hacking (relying on internal implementation instead of public API).
- Provides two sets of APIs (such as one set of old version API and one set of new version API, or one set of simple but slow API and one set of complex but fast API). But two sets of APIs have complex interactions under the hood, using both of them causes weird behaviors.
- Lacking of isolation and orthogonality. Changing one thing affects another thing that's seemingly unrelated. An example is layout in CSS.
- Having strict constraint that makes prototyping hard. In Rust changing data structure may involve huge refactoring (adding or removing lifetime parameters in every usage, replacing a reference with Arc, etc. [See also](https://loglog.games/blog/leaving-rust-gamedev/)). These constraints can help correctness and make reviewing PR easier, but they hinder prototyping and iteration. It's a tradeoff.
- Default API usage make it easy to be used inefficiently. Example: directly passing regular expression string in argument cause it to parse regular expression on every call (can be mitigated by underlying caching).
- Sacrifice usability for seemingly correctness. An example is Windows's file system, where you cannot move or delete a file that's being used. This seemingly helps correctness, but it make software upgrade harder. In Windows, softwre upgrading is error-prone to other software reading its files. Can only safely upgrade via rebooting. Also forgien key helps correctness but make backup loading and schema migration harder.
- The API was designed without caring about performance, and cannot optimize in the future without breaking compatibility. 
- The API overfly focus on security, making doing simple things harder.
- Feedback loop is long. Example: after changing the code, the developer have to wait for slow CI/CD to see the effect in website. The long feedback loop makes working inefficient, consumes more patience and make the developer retain less temporal memory. A good example is hot-reloading, where feedback loop is very short. 
- An LLM hallucinates about an important nuanced assumption, causing developer to misunderstand the API's nuanced assumption, then waste a lot of time debugging without questioning the assumption.
- Order-dependent setup and fragility to mis-ordering. Getting one order wrong cause it to break. This is especially hard to deal with in concurrent or distributed systems, where the order is influenced by random factors, causing unreproducable random errors. 
- Duplicated configuration. When a configuration is duplicated 3 times, changing it requries changing all of the 3 places.
- Overly flexible config file. A config file is a plain text file that does not support rich features provided by a normal programming language, such as variables, conditions and repetition. **Trying to make the config file more flexible and expressive eventually turn it into a DSL that's hard to use** (existing debugging and logging tools cannot be used on it, existing libraries cannot be used on it, and it usually lacks IDE support).
- Have to maintain consistency between the data managed by library and the data managed by your code. Each one can update the other one (no single source of truth). If the two pices of data are not kept in sync, weird issues will happen.

