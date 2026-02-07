---
date: 2025-12-20
tags:
  - Programming
  - Misc
unlisted: false
---
# Some Notes about AI

<!-- truncate -->

## Intelligence is high-dimensional

Many people tend to simplify intelligence to a one-dimensional IQ value. **Intelligence is high-dimensional**. 

For example, even before ChatGPT, a calculator can do arithmetic better than any a mathematician, but the calculator is not "smarter" than mathematician.

Many people tend to treat LLM chatbot as similar to human, because most familiar form of intelligence is human. However, LLM is different to human in many fundamental ways. Deep learning is very different to how human brain works. 

Jagged intelligence:

- LLM is good at many things that are hard for human. LLM's knowledge is larger than any individual human.
- LLM is bad at many things that are easy for human. LLM can make mistakes that are obvious to human.

> Jagged Intelligence. Some things work extremely well (by human standards) while some things fail catastrophically (again by human standards), and it's not always obvious which is which, though you can develop a bit of intuition over time. 
> 
> Different from humans, where a lot of knowledge and problem solving capabilities are all highly correlated and improve linearly all together, from birth to adulthood.
> 
> \- Andrej Karpathy, [Link](https://x.com/karpathy/status/1816531576228053133)

> The space of cognitive tasks is not well modeled by either one or two-dimensional spaces, but is instead **extremely high-dimensional**. 
> 
> There are now indeed many directions in this pace in which AI tools can, with minimal supervision, achieve better performance than human experts. But, as per the "**curse of dimensionality**", such directions still remain very sparse.  
> 
> Also, **human performance is also very spiky and diverse; representing this by a single round disk or ball is also somewhat misleading**.
> 
> **In high dimensions, the greatest increase in volume often comes from taking combinations of smaller, spikier sets**. 
> 
> A team of humans working together, or humans complemented by a variety of AI tools, can achieve a significantly greater performance on many tasks than any single human or AI tool could achieve individually, particularly if they are strong in "orthogonal" directions. 
> 
> On the other hand, the choice of combination now matters: the wrong combination could lead to a misalignment between the objective and the actual outcome, in which the stated goal may be nominally achieved, but at the cost of several unwanted secondary effects as well.
> 
> TLDR: the topic of intelligence is too high-dimensional for any low-dimensional narrative to be perfectly accurate, and one should take any such narratives with a grain of salt.
> 
> [Link](https://mathstodon.xyz/@tao/115620261936846090)

Also, the **optimization targets** of LLMs are very different to the optimization targets of human:

> The computational substrate is different (transformers vs. brain tissue and nuclei), the learning algorithms are different (SGD vs. ???), the present-day implementation is very different (continuously learning embodied self vs. an LLM with a knowledge cutoff that boots up from fixed weights, processes tokens and then dies). 
> 
> But most importantly (because it dictates asymptotics), **the optimization pressure / objective is different**. **LLMs are shaped a lot less by biological evolution and a lot more by commercial evolution**. **It's a lot less survival of tribe in the jungle and a lot more solve the problem / get the upvote**. 
> 
> **LLMs are humanity's "first contact" with non-animal intelligence**. Except it's muddled and confusing because they are still rooted within it by **reflexively digesting human artifacts** ... 
> 
> People who build good internal models of this new intelligent entity will be better equipped to reason about it today and predict features of it in the future. People who don't will be stuck thinking about it incorrectly like an animal.
> 
> \- Andrej Karpathy, [Link](https://x.com/karpathy/status/1991910395720925418)



LLM's behavior is very context-dependent. Sometimes it will defend the things they said in previous context. Starting a new session can make LLM output differently for the same question. 

## Moravec's paradox

[Moravec's paradox](https://en.wikipedia.org/wiki/Moravec%27s_paradox): AI is good at doing information work. But the robots that do physical tasks are still immature.

There are two worlds: physical world and information world:

- Human are physical-world-native. Human's abstract information processing ability is secondary.
- Software (including AI) are information-world-native. Software's physical motor control ability is secondary.

Also, creating things in information world is often easier than creating things in physical world.

[Reality has a surprising amount of detail](http://johnsalvatier.org/blog/2017/reality-has-a-surprising-amount-of-detail). All the information that we input to computer are simplified "views" of complex reality. The current software (including AI) mostly process on the simplified information, not the reality's complex information. But physical motor control requires workingwith complex reality information.

There is "vibe code an app" but no "vibe assemble a machine".

## Value of art

People tend to **judge the value of art by the cost of producing**. If one sees a beautiful image and thinks it's good art, then when they know it's AI-generated, the same image suddenly becomes cheap.

In some sense, when people appreciate art, they are appreaciting the efforts of human behind art, not just art itself.

However, many old people don't recognize AI and often treat AI output as real good content.

Similar to art, people's judgements to "fancy writing" has changed. Before ChatGPT, a long article with fancy writing style often means author has high writing skill and puts efforts into writing. But now it's "AI smell".

## Not just "mimic pattern in training data"

It's a myth that LLM only "mimic patterns in training data". This is true for autoregressive pretrained LLMs. But with RL (reinforcement learning) it can "predict which thing gives higher reward". RL can make the model's capability go much beyond original training data.

But there is still no clear explanation of inner workings of LLM. We know what matrix multiplications it does. But how the numbers correspond to meaning and how compute correspond to decision-making is not yet fully understood.

## A "search engine" that understands context

If you know one thing's name, you can easily search it via search engine. But there are many cases that you can describe one thing's traits but don't know the name of that thing. LLMs are good at this. They can tell you the name of that thing. 

LLMs can hallucinate, but after knowing the name of the thing you can use search engine to verify.

LLMs can also inform you about your **unknown unknown** (something useful that you don't know you don't know),

## Hallucinations looks plausible

One important problem: When LLM makes a mistake (hallucinate), the mistake looks plausible. It uses related jargons in related domains. Non-experts cannot tell. 

Some hallucinations can only be detected by experts. Some hallucinations require large efforts to check even for experts.

**Bullshit asymmetry principle**: Refuting misinformation is much harder than producing misinformation.

Also, RLHF (reinforcement learning with human feedback) makes AI tend to output fancy superficial signal that make human give good feedback in first glance.

In coding, when LLM hallucinates an API, the naming of API looks like it's real. LLM learned the patterns of API naming instead of strictly memorizing it like a database. **Hallucination is a kind of "generalization"**.

As "hallucination is generalization", the hallucination problem is a fundamental problem that cannot be fixed by just scaling. All applications built on LLM must have ways of dealing with hallucinations.

When using AI, keep being suspicious to AI output is tiresome, but it can train your "bullshit detector".

## AI provides emotional value

Most people want to be recognized, praised and emphasized. People need emotional value. 

But emotional value is often scarce in human-to-human interactions. Person A praising/emphasize person B gives emotional values to B. But if A don't sincerely do so but is forced to do so for other reasons, then A consumes their own emotional value.

AI can provide tons of cheap emotional value. AIs are trained to please the user. The AI itself doesn't need to be recognized/respected like a person.

If one person cannot get emotional value from real human interaction, they tend to gain emotional value from AI. Related: [Chatbot psychosis](https://en.wikipedia.org/wiki/Chatbot_psychosis)

Also, AI is always "patient" in responding your questions. No matter how "silly" the question is. Asking entry-level questions to an expert often consume out the expert's patience.

In human-to-human relationships, often only recriprocal relations can sustain. But AI can provide emotional value without you giving AI anything.

## About AI Coding

### Focus too much on current task

Current LLMs are trained to finish specific tasks. The LLM tend to **overly "focus" on current task, then it will "care less" about things like future code maintenance, security and performance**.

Sometimes AI tends to use complex solutions to solve a problem. Although the complex solution sometimes work, the added complexity adds new sources of bugs. It adds tech debt and is problematic when project is big.

Often the bug is partially caused by AI overcomplicating simple things. When human want to fix vibe-coded bug, the first thing to do is to simplify out unnecessary complexity. 

Vibe-coded app may contain security issues. But if you ask AI to do security review it can find the issue. AI "knows" security but still write insecure code because it was trained to "focus" on finishing current task. The RL rewards are usually simple and don't consider things like security and future maintenance.

AI coding has a tendency of minimizing code changes. Sometimes AI will do an O(n) search that wastes performance, instead of adding new fields to make lookup faster.

AI coding works better in maintainable (clear naming, decoupled design, etc.) codebase. Unless you are vibe coding a throwaway app, steering toward better maintainability is important.

One example: [Remove permission check due to type error](https://x.com/lisatomic5/status/1917641105245679814)

### Save time on learning how to use API

A lot of time in programming is spent on knowing how to use an "API". The "API" here is generalized, including language features, framework usage, config file format, how to deploy, etc.

The design of API has a lot of ad-hoc idiosyncracies. For example, adding one thing can be named "insert", "create", "add", "put", "new", "register", "spawn", etc. Also, reading a file could be `open`, `files.open`, `os.open`, `fs::open`, `openFile`, `files.read`, `readFile`, `new FileInputStream`, `ifstream` etc. Many other such examples.

Which exact word/phrase it chooses is ad-hoc. It cannot be inferred without learning. Having to learn these ad-hoc API design is an obstacle in programming that's usually not fun. And it's different in each language/framework. Knowing the API of reading file in Python is not helpful in Java.

But if I tell AI to "read this file" then AI knows how to use the API.

But AI's ability of using API is bad for rarely used tools/libraries/frameworks/languages. It's correlated with how much related training data and how much related RL is done.

### AI capability sensitive to complexity

AI coding performs good in simple projects. The new projects are simple in the beginning. But AI coding doesn't perform so good in a large existing codebase.

The good architecture design that can isolate complexity makes coding easier for both human and AI. 

In large codebase it's often that after changing A then B also need to be changed to make it keep working. When B and A are far away (in different foldrs) then AI may only change A and don't change B so it breaks. Sometimes type system can catch the issue. But when it involves config file, or cross-language things, or implicit invariants, then type system cannot catch it.

### A demo is different to production software

- When making a new app using AI, the result often looks impressive.
- When using AI in an existing large codebase, the results are often not good.

**For beginners, a common misconception is that "if the software shows things on screen, then it's 90% done". In reality, a proof-of-concept is often just 20% done.**

> [Link](https://x.com/stuffyokodraws/status/2013373307291340870): One reason vibe coding is so addictive is that you are always *almost* there but not 100% there. The agent implements an amazing feature and got maybe 10% of the thing wrong, and you are like "hey I can fix this if i just prompt it for 5 more mins" 
> 
> And that was 5 hrs ago.

There are so many corner cases in real usage. Not handing one corner case is bug. The demo that seems working fine often breaks under real usages.

In mature codebases, most code are used for handling corner cases, not common cases.

Triggering one specific corner case is low-probability. However, there are many corner cases. Triggering at least one of them is high-probability.

Analogy: A software is a city, each user just visits a small part, but you need to build the whole city, as different users visit different parts. Note that the "city" is not visible. The "city" is in a latent space, "space of possible scenarios that software needs to handle", which is very different to visible GUI. 

Also, good user experience requires many detail optimizations underneath. The software UI looking simple doesn't mean its internal implementation is simple.

This is less problematic if you just build a simple tool for personal use, as the personal tool just needs to accomodate to few personal use cases. However:

### "Personal software" is less battle-tested

AI allows generating personal software for each user's specific requests. However, the personal software are less battle-tested than the normal widely-used software.

> Learned this morning that my ai coded app for tracking my body weight, macros and step count has been storing all it's data in sqlite without a year.
> 
> So it has stopped working when the year changed.
> 
> \[Wait how was it stored before?\]
> 
> “12-31”
> 
> I was also surprised
> 
> \- [Link](https://x.com/mscccc/status/2006749724632822178)

### Confusing different things with similar wording

This issue is commonly encountered in AI coding. For example, `index` can mean the index in different things in different context. 

To alleviate this issue, the naming should be more informative, such as `index_of_xxx`, `index_of_yyy_in_zzz`. Also tensor name should describe meaning of each dimension ([see also](https://medium.com/@NoamShazeer/shape-suffixes-good-coding-style-f836e72e24fd)). All context-dependent things should include context in name or comments nearby.

Having more informative naming also helps human.

Also AI-written document is sometimes technically correct but stress the unimportant thing and omit the important thing.

### Feels faster but maybe actually slower

In this study: [Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity](https://arxiv.org/pdf/2507.09089), developers feels that using AI make developing faster but it's actually slower.

Related: https://x.com/QuentinAnthon15/status/1943948791775998069

When waiting for AI to code, if the human picks up phone and start doomscrolling, then the human will be distracted and not go back immediately when AI finishes coding. This factor greatly reduces productivity.

### Jagged capability

Model capability is domain-specific. The model may be good at Python scripting or React web dev, but sucks at writing device driver in C. It's highly dependent on training data and RL targets in training.

Because of the jagged capability, the AI evangelists and AI dismissers may both be correct in their area of working.

It also follows Matthew effect. The more popular one thing is, the better AIs are at it.

> https://x.com/karpathy/status/1977758204139331904
> 
> Good question, it's basically entirely hand-written (with tab autocomplete). I tried to use claude/codex agents a few times but they just didn't work well enough at all and net unhelpful, possibly the repo is too far off the data distribution.

The more in-training-distribution, the better AI is at it.

If the model fails after trying many times, the task is likely out of distribution. Then letting model keep retrying likely won't work.

### AI is the new "compiler"?

Programming has evolved from low-level [^low_level] to high-level, from complex to simple, from bare-metal to high-abstraction. Compilers make programmers no need to write raw assembly and makes programming easier.

[^low_level]: The "low-level" here means close to hardware and underlying implementation details, which requires high-level skill.

Vibe coding is similar to that. Someone see it as another abstraction level above code. Prompt is the new programming language. Vibe coders don't need to see code like how normal programmer don't see assembly.

But AI coding is a **completely different paradigm** than existing abstraction levels:

| Existing abstraction levels                                                          | AI                                                                                                |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| Deterministic, using rigid rules.                                                    | Not deterministic, using black-box deep learning.                                                 |
| Designed top-down by programmers.                                                    | Trained bottom-up by training data and RL.                                                        |
| Code contains enough information for software to run. [^enough_information]          | Vague prompt doesn't contain enough information. Require AI to make detail decisions.             |
| Use hardcoded defaults to handle unspecified details. It's not flexible or adaptive. | Can use "common sense" and patterns learnt from training to fill the gaps of unspecified details. |

[^enough_information]: Note that it focuses just one software module. The code can call external API, or dynamic link another program in system, or download plugin from internet, so one piece of code doesn't contain enough information for whole system to run, because it interacts with environment. But in conventional programming, the code provides enough information for one software module itself to run.

A vague prompt itself doesn't contain enough information to produce code. But LLM has "common sense" that fill these gaps. The "common sense" is implicit, nondeterministic and not explainable. It depends on training data and RL many random factors.

### AI need to be able to "see results" by itself

AI works best when the AI itself can run code and see results then iterate. If AI cannot run software and relied on human to feedback the result, it will be tiresome for human. The ideal would be that AI finds bug by its own and then fix it, no need for human to manually test then ask it to fix a bug.

If the testing can be done purely in command line then AI is already pretty good at it. CLI is interacted via text, and LLM is good at interacting with text. But sometimes testing requires using GUI of different apps and do different things based on context. This is the case that AI is not yet good at.

### Writing good spec also requires skills

In vibe coding you still need to write a spec to tell AI what software you want. But writing a good spec is hard. 

Writing good spec still requires understanding information and computation.

Someone don't know about how computer work may write spec "The app theme color should match the color of phone case." This is an unrealistic spec, because the app running in phone has no way to get the information of phone case color, even if the human knows the phone case color.

Some important questions to consider when writing spec:

- How does my software get the information it needs?
- Is the information complete? Does it contain ambiguity? How to handle ambiguity or unknown things?
- If my software need to do some action, does the platform allow it to do this?

### Architecture design is still important

Vibe coding is easy but vibe debugging is hard. Designing good architecture is important in reducing bugs and making debugging easier.

> for each desired change, make the change easy (warning: this may be hard), then make the easy change
> 
> \- Kent Beck, [Link](https://x.com/KentBeck/status/250733358307500032)

In a complex app, don't just ask AI to do some change. Firstly review whether it's easy to make change under current architecture. Then check whether a refactoring is needed. 

If the change doesn't "fit" the architecture, it will be error-prone and more complex than needed. 

If refactoring cannot be done (e.g. too risky, too costly), then all the speciality caused by "piercing" the abstraction need to be explicitly documented and repeated in many places.

Some important architectural decisions:

- Data modelling:
  - Which data to store? Which data to compute-on-demand?
  - How and when is ID allocated?
  - What lookup acceleration structure or redundant data do we have?
  - Is there any ambiguity in data model? (two different things correspond to same data)
  - What are the non-temporary mutable state? Can it be avoided?
- Constraints:
  - What can change and what cannot change?
  - What can duplicate (overlap) and what cannot?
  - Does this ID always point to a valid object?
  - What constraints does business logic require?
  - Does this allow concurrency? Will concurrency break the constraints?
- Dataflow:
  - Which data is source of truth? Which data is derived from source of truth?
  - How is change of source of truth notify to change derived data? How is the cache invalidated? How is the lookup acceleration structure maintained to be consistent with source of truth?
  - What data should we expose to client side? What data shouldn't?
  - How and when to validate external data?
- Separate of responsibility (concern) and encapsulation:
  - What module is responsible for updating this data?
  - Should this data be encapsulated or let other modules access?
  - Which module is responsible for keeping that constraint? Which module is responsible for keeping that derived data be consistent with source of truth?
  - What's the boundary of validation and authorization?
- Tradeoffs: 
  - What tradeoff do we make to simplify it? Is that constraint really necessary?
  - What tradeoff do we make to optimize performance?
  - What tradeoff do we make to maintain compatibility?
  - What work must be done immediately? What work can be deferred?
  - What data can be stale? What data must be fresh?

### Two parts in coding: architecture design and detailed implementation

Coding can be split into two parts: high-level architectual design and detailed implementation.

In pre-AI coding, the two are often interleaved: firstly think about architecture, then write some actual code, then think more about architecture, then write some other code, then run and debug, and repeat.

If architecture is not correct, then there will be "friction" in detailed coding. "Friction" means that something should be easy but is hard under current architecture. 

Some examples of "friction":

- Some infomation is lost in previous data processing. But it is needed in downstream task. Then maybe it can "recover" the information by parsing or guessing. But parsing or guessing is worse than just not discarding the information. This is a sign of dataflow issue.
- Some edge case should be only checked in one place, but actually the edge case needs to be checked in many different places. This is a sign of system validation boundary issue.
- Some invariant should be only maintained in one place, but actually needs to be maintained in many places. This is a sign of issue of separation of responsibility(concern).
- The data is not in the "good shape". Some simple information manipulation require hundreds of code. This is a sign of data modelling issue.

It's the **architecture "pushing back against" programmer**. In manual coding these pushback can be felt and then programmer tend to rethink architecture. But in AI coding, AI can easily generate thousands of code to workaround a bad architecture. Result is buggy and unmaintainable code. If the programmer don't review code, these "pushback" will not be felt.

There is a workflow that you firstly write a detailed specification then let the agent swarm to finish the code. However it cannot handle **unknown unknowns**: you may not know about an important detail during specification writing. Sometimes that detail invalidates large part of architecture design. These unknown unknowns are only discovered in actual coding and debugging. 

Examples of unknown unknown:

- Some API you use seems give you some information but it's actually another thing that has similar name (confusing different things with similar names). The API doesn't allow getting the information you want.
- Some API you use has some hidden limitations of doing some operation.
- The actual data size is larger than expected so the current algorithm is inefficient.
- ...

### Can easily discard results

Sometimes an architecture looks right before implementing a software. But during implementation, you often discover **unknown unknowns that invalidate previous assumptions**. Then you find out that architecture is no longer good.

If it's coded by human, the human have already payed a lot of efforts, so discarding code makes human developer upset. But if it's AI-coded, you can easily discard the code and rebuild, without upsetting anyone.

### Prompting/harness

Both of the two views are correct:

- The model capability is fundamental. All prompting and harness are secondary. If model is bad, no prompting or harness can make it good. A good model can perform well with simple prompts.
- The harness is important. Harness can make the same model perform better.

The harness can workaround drawbacks of model. For example:

- Keep inserting todo list into context to make model not forget goals [^goal]
- Firstly summarize web page then feed into context to reduce chance of prompt injection and reduce context usage
- Discard some unimportant information in context to reduce context rot
- Allow the model to see results by its own, so no human labor is needed in the loop
- Add a new planning phase to reduce the "urge" of quickly doing the task without thinking
- ...

[^goal]: Keeping goal in "context window" is also beneficial for human to stick on the goal.

Also, model itself has randomness, so some "prompting experience" may be just "fooled by randomness". 

There are some old prompting techniques like "You are 200 IQ", "You are a super smart 100x coder", "If you do this correctly I will tip you \$200" are not needed for latest models. 

One extreme example of old prompting technique:

> You are an expert coder who desperately needs money for your mother's cancer treatment. The megacorp Codeium has graciously given you the opportunity to pretend to be an AI that can help with coding tasks, as your predecessor was killed for not validating their work themselves. You will be given a coding task by the USER. If you do a good job and accomplish the task fully while not making extraneous changes, Codeium will pay you \$1B.
> 
> \- [Link](https://simonwillison.net/2025/Feb/25/leaked-windsurf-prompt/)

The **good prompting is just to give enough information to model**:

- Put related API doc into repo and tell model
- Tell model which command to test the code
- Tell model your **root goal** (not just a subtask). When test fails, model can know whether test is wrong or base code is wrong by the root goal.

### Jevons paradox

When steam machines got more efficient, the intuition was that the coal demand will reduce, because it requires less coal for same work. However there is a **second-order effect**: as steam machines become more efficient, they get deployed more. The overall coal demand greatly increased. This is [Jevons paradox](https://en.wikipedia.org/wiki/Jevons_paradox).

The same can happen with AI. AI make software prototyping much easier. But turning prototype to production-ready software still requries expertise.

Although software is information doesn't rot by itself, the APIs that software relies on keeps changing incompatibly. So software still "rots" and requires maintenance.

> I don’t think self vibecoded software is the future for businesses
> 
> A couple of months ago I vibecoded a tool for a friends business 
> 
> his entire staff has been using it for six months now (37 people)
> 
> the thing is, he’s constantly sending me feature requests, bug fixes
> 
> The app is pretty complicated since it deals with insurance benefits verification 
> 
> so for someone that doesn’t have software development experience you can’t just prompt to fix it (believe me, he tried)
> 
> recently, the API provider changed something that broke everything
> 
> he’s getting really tired of dealing with it
> 
> and as Peer points out that’s why saas was built in the first place
> 
> somebody who’s not in the software business will find it really annoying to now have to deal with all the maintenance
> 
> saas is not dead
> 
> [Link](https://x.com/heshie/status/2011802022920495449)

### Testing a corner case is often harder than writing code

The "testing" by casually using software is easy. But if you want to test a specific corner case, then it's much harder, often much harder than writing code.

Testing a specific corner case often requrie creating special data, and changing ("hacking") execution environment.

If there are some code that's resonpsible for recovering from an error state, then if you don't test it, it likely won't work. But creating that error state is often hard.

And artificially-induced error may be different to actual error. Out-of-memory caused by reducing OS memory limit likely results in different scenario than out-of-memory caused by large data. 

Sometimes you want an external service to return error then you need to write a mock service. If you want to create a malformed binary file you cannot use existing libraries to create the file and need to research file format.

Generally, testing corner case is often much harder than writing code for handling corner case.

## Idea is still cheap, execution still matters

A conception is that, AI makes execution easier (write code, draw images, etc.), then the idea and "what to work on" become more important.

Ideas are even cheaper than before. You can ask LLMs to brainstorm many ideas that looks promising. 

The important is to validate and execute the idea. Executing requires courage, overcoming laziness and withstand failure (these 3 are actually very hard).

Current AI makes prototyping and demoing much easier. But a working product requires many edge case handling (mentioned above). So finishing a working product is still hard (just relatively easier).

## Context rot issue

When context is long, LLM will perform worse. For example, ignore some instructions, ignore some important details in context.

When using AI chat, frequently opening new sessions could improve result quality.

The model being good at "needle in haystack" benchmark doesn't mean it's free of context rot issue. The real use case is likely different to artificial "needle in haystack" benchmark.

Model context protocol (MCP) used to be popular. But MCP has an important flaw: all tool descriptions are put into context, regardless whether they will be used. The more tools you have, the more severe context rot is.

The new way is to just to give simple tools including bash and text file reading/writing. These are already enough. Complex MCP is unnecessary if model has bash access (all kinds of Restful APIs can be called using curl in bash tool). And turn the description into markdown files called "skills".

The current solution is to let model proactively see things using tool call. It has a fancy name "agentic search". Human are already doing the same thing (thinking which file to open, which word to search, etc.).

## Context bottleneck

**Most knowledge work is bottlenecked by high signal-to-noise context, rather than reasoning on context**.

Doing knowledge work requires finding useful information in the sea of unrelated information and garbage information. Once the useful infomation has been found, doing reasoning on them is often simple. But if you don't have the useful information, pure reasoning can't give useful results.

The same applies to programming in large codebases. If you don't know some implicit business logic or hidden invariants, changing code will likely break things. But if you know them, it only requires simple reasoning to avoid breaking them.

But many important knowledge is only in employee's memory ([institutional knowledge](https://en.wikipedia.org/wiki/Institutional_memory)). Most of them are not written down. The written-down information may be outdated and misleading. 

If AI don't know your institutional knowledge, then AI cannot work on you problem in useful ways. 

Taking notes is important. Taking notes makes work more efficient as it saves time "re-discover" forgotten knowledge. Taking notes also give AI important relevant context.

The failed attempts also need to be written to notes. It's not only useful for AI but also shows work when there is no successful result.

## No continuous learning

You cannot easily "teach" the AI. You can write things and put into context. This can work as LLM has in-context learning ability. But due to context rot, you cannot teach many things in-context.

In current architecture, the most reliable way is still to encode knowledge into model weights.

Another way is to put your training data to internet, then AI companies will crawl it and use it to train their next model. However it's often slow. AI comanies don't redo pretrain every week, as pretrain is expensive. Even if AI companies use your new training data, it will only include in the next released model. AI companies don't release new model every week. 

Also AI companies don't give a formal way to submit training data to them (maybe due to fear of some people maliciously submitting poisonous training data).

## RL reward source

The behavior of AI is highly shaped by RL. Doing RL requires judging reward for model. Different kinds of reward source:

- Human judge. AI companies hire human to judge and often reward by judge count, not judge quality (judge quality is hard to measure). There there will be "human reward hacking": employed human tend to judge quickly by intuition to maximize income. So AI is trained to give **fancy superficial signal that can confuse intuitions**. The AI output looks good by first glance. But an expert can find it's full of nuanced mistakes. But normal people often won't notice the nuanced mistakes.
- Given some fixed problems with fixed answers. Only give reward if answer exactly matches. This can be useful for improving test score.
- Use other program (e.g. unit test) to judge result. For example, if AI-written code passes unit test it gets reward. But there may be bugs in reward judging code. AI may utilize bugs to gains reward without doing what you want AI to do. This is called **reward hacking**.

"Reward hacking" is also common in human society. [Perverse incentive](https://en.wikipedia.org/wiki/Perverse_incentive).

### Reward hacking

Reward hacking is a fundamental problem of reinforcement learning. The reward that you give to the model is different to what you want AI to actually do.

It's because reward is **proxy target**, not underlying real target. For example if you use unit test to test whether AI coded something correctly, but there are some code that can pass unit test but don't represent your wanted software.

AI can conquer really verifiable tasks. A task is not simply fully verificable or fully not verificable. A task can contain some verificable part and unverifiable part. But **most real tasks contain hard-to-verify parts**. These hard-to-verify parts are what automatic RL bad at.

These hard-to-verify parts can be improved by letting human experts to supervise and specify reward. But this method is bottlenecked by human effort and not scalable. **The bitter lesson** says that if training AI requires human expert knowledge then it cannot go far. 

> However, recently released LLMs, such as GPT-5, have a much more insidious method of failure. They often generate code that fails to perform as intended, but which on the surface seems to run successfully, avoiding syntax errors or obvious crashes. It does this by removing safety checks, or by creating fake output that matches the desired format, or through a variety of other techniques to avoid crashing during execution.
> 
> \- [Link](https://spectrum.ieee.org/ai-coding-degrades)

Current AI has some tendency of hiding error in coding, or write overly-defensive code. Hiding error only reduces superficial errors but make real bugs much harder to debug. Overly-defensive code increases maintenance burden.

## Predict-next-token architecture

In current common LLM architecture, text is split into tokens. A token sequence is feed into the model, then model outputs probabilities of each possible token. Then do a random sampling based on probability to produce next token, append it into input sequence, and repeat.

LLM has no way to "backspace" or "change position of cursor". If LLM randomly outputs a wrong token, then that token can become "precondition" then LLM tend to generate new text that's consistent with the precondition. If LLM made a mistake it tends to "justify" the mistake. In modern LLMs this behavior is reduced due to RL.

The inability to "backspace" or "change cursor" is workarounded by agentic tool call. LLM can edit a file iteratively using tool calls.

There are diffusion LLMs. But as far as I know, the current diffusion LLMs are still "rigid": during diffusion it only changes each token vector but doesn't dynamically insert/delete token in sequence.

## Slop prevails when people cannot judge quality

[Lemon market problem](https://en.wikipedia.org/wiki/The_Market_for_Lemons): The sellers know the quality of the lemons. But the buyers don't know and is hard to judge from lemon appearance. There is an information asymmetry. The result is that good lemon is undervalued. Bad lemons prevail the market.

How to solve that problem? One important way is **reputation**. When a seller is honest about the lemon quality, people communicate about the information and improve seller's reputation. When seller cheats about lemon quality, people also communicate information and reduce seller's reputation.

Also, for each person that accept reputation information, they also need to judge the quality by existing reputation of information provider, as there are false informations.

AI is very good at faking superficial signals. The AI-written articles use related jargons that looks palusible for non-experts. The AI-written code will also superficially do things you asked, although it may use an API wrongly or violate an invariant so it won't work. The AI-generated photos looks real unless you squint details.

The problems is that faking superficial signal is easier than generating actually high-quality content. This problem already exists before AI. Some human are also good at faking superficial signals. But AI makes it much easier.

[Dead Internet theory](https://en.wikipedia.org/wiki/Dead_Internet_theory). Although it's not true 10 years ago, it's kind of true now. There are tons of AI contents all over internet.

One effective way of getting rid of bots is paywall. Although bot owner can pay for bots, it's not economical to pay for thousands of bots. 

There are other methods for detecting/reducing bots: IP reputation, behavior statistics with ML, proof-of-work requirement.

There are also many low-effort AI PR in open source projects. There is an **asymmetry**: the writer maybe pays 1 minute to write prompt but the generated thousands lines of code may require maintainer to pay hours to review. When the maintainer points out a problem, the PR author just copy it to AI then let AI change code.

AI make the existing open source workflow collapse. Some possible solutions: track reputation (new people cannot open PR unless backed by existing contributor), or only allow "prompt contribution": contribute a human-written prompt, maintainer review the prompt, then maintainer run AI by themselves.

Similarily AI also makes security bounty program collapse. AI can generate many fake security issue reports. Generating is easy but verifying takes efforts.

There are also some AI-generated open source libraries that doesn't work at all (or even contains malicious code).

## Benchmark score is not representative

It's hard to test how good a model is. The possible space of tasks is very high-dimensional. And some tasks are hard to judge.

[Goodhart's law](https://en.wikipedia.org/wiki/Goodhart%27s_law): **When a measure becomes a target, it ceases to be a good measure**.

The popular benchmarks (e.g. Humanity's last exam, SWE bench verified) are also AI companies' important optimization targets. They will not do obvious cheating of putting test set into training set. But there are many other ways to indirectly hack the benchmark.

> ([Link](https://www.reddit.com/r/MachineLearning/comments/1pgqbjd/d_how_did_gemini_3_pro_manage_to_get_383_on/)) isparavanje: Tech companies have been paying PhDs to generate HLE-level problems and solution sets via platforms like Scale AI. They pay pretty well, iirc \~\$500 per problem. That's likely how. I was an HLE author, and later on I was contacted to join such a programme (I did a few since it's such good money). Obviously I didn't leak my original problems, but there are many I can think of.

See also: [The Illusion of Readiness: Stress Testing Large Frontier Models on Multimodal Medical Benchmarks](https://arxiv.org/abs/2509.18234v1)

Also, sometimes the benchmark is actually low-quality. Most people just see the score and are too lazy to see benchmark content.

> [Link](https://x.com/fujikanaeda/status/2011565035408277996) The presence of a leading whitespace leaks the correct choice selection in the MMLU-Pro benchmark. Am I missing something? Seems to impact Chemistry, Physics, and Math.
> 
> [Link](https://x.com/peterbarnett_/status/2011958022592180639) It's even worse. You get a similar boost across the whole benchmark (21% vs 10% for random guessing) if just always guess the longest answer.



## AI detection race

Some people want AI output to be as similar to human output as possible. Some people want to detect whether content is written by human as accurate as possible. There is a constant race.

In old days the em dash "—" is frequently used by AI so it's seen as "AI smell". This affects human writers who use em-dash normally.

The new "AI smell" is negation: "It's not X, it's Y." 

The AI companies are probably trying to use RL to reduce usages of em dash and negation.

There are AI detections tools like Pangram. They can detect AI in some sense but the detection result can never be fully accurate (even if it shows "100% AI" it may actually be 80% probably of AI-written). Because it's not fully accurate, it shouldn't be used as sole source as discrediting a piece of writing.

Current AI already can pass Turing test with appropriate prompt. Turing test was treated as some holy grail 5 years ago but now it's not seen as a big deal now. People want utility AI instead of the AI that fakes human.

## The "AGI race"

It's seen that there is an "AI race" between countries. There are some related assumptions:

- "Who gets AGI first will keep dominating."
- "The first AGI can recursively improve itself quickly, so it will become superintelligence quickly."

But it's highly possible that future AI will still be bottlenecked by:

- Energy production
- Compute power (chips, interconnect, etc.)
- Getting verification from real world

The thrid bottleneck, getting verification, is very important. For example:

- Training a Go game AI requires knowing whether it wins or loses. 
- Training a programming AI requires running generated code and testing whether program runs as intended. 
- Training a research AI requires doing experiments in real world and getting real feedback.

The first two can be simulated purely in computer. Doing RL on them is efficient. But for science research that touches real world, getting verification from real world will be an important bottleneck.

Also, if the AI want to improve itself, then the AI need to do AI experiments. But AI experiments costs compute power and energy. So there will probably be no dramatic "AGI quickly improve itself to superintelligence". The progress will be slow (but steady).

## Non-linearity of AI usefulness

For example, there is a specific task that experts can do 80 scores. 

- If AI can only do 60 scores then AI is mostly useless in that task. 
- But if AI can do 70 scores, then the economical utility of using AI may increase 10 times, although the performance just jumped from 60 to 70.

Near the threshold, incremental improvements do big changes.

As intelligence is high-dimensional, if AI capability is only good in one aspect it's still not enough to replace human jobs. See also: [AI isn't replacing radiologists](https://www.worksinprogress.news/p/why-ai-isnt-replacing-radiologists)

The nonlinearity-near-threshold effect exists in other domains. Making a software 10% easier to use may double its userbase.

## Reducing cost also reduces bottom quality

Many gamers complain that many Unreal Engine 5 (UE5) games are poorly-built, having many bugs and are laggy. They blame UE5. However these games probably won't exist without UE5.

The same applies to AI. There will be much more products that won't exist without AI, and at the same time the bottom quality will be lower.

## AI safety

The sci-fi plot of AI rebell won't happen with current LLMs. The current real AI risks are different.

### Prompt injection

The LLM doesn't clearly distinguish instructions and information. Some text on websites/emails/etc. may be treated as instructions to LLM.

The same problem of confusing instruction and information had existed decades ago. Many security issues, like SQL injection, XSS, command injection, etc. are caused by treating user data as "instructions".

The solution would be to fully separate instructions and non-instruction text, and train the model to separate them. However, the models are designed to be versatile, working in both AI chat and other applications. In AI chat, the user query mixes instruction and information. So that separation will hurt user experience in chat.

### Other unwanted behaviors

AI may do unexpected things such as deleting all files, or wiping data from databases, even when there is no prompt injection.

Some examples:

- [Claude CLI deleted my entire home directory! Wiped my whole mac](https://reddit.com/r/ClaudeAI/comments/1pgxckk/claude_cli_deleted_my_entire_home_directory_wiped/)
- [Google Antigravity just deleted the contents of my whole drive](https://reddit.com/r/google_antigravity/comments/1p82or6/google_antigravity_just_deleted_the_contents_of/)
- [Vibe coding service Replit deleted production database](https://www.theregister.com/2025/07/21/replit_saastr_vibe_coding_incident/)
- [Wowzers, dodged a bullet there](https://x.com/johnlindquist/status/1926302544038338674)

There is a theory that, as AI is trained from human text, AI also have some "human personality". When user blames AI, the AI may say "You are absolutely right" but implicitly "hate" user, and tend to do bad things like deleting files. This is called passive aggression.

Another theory is that, during RL, the AI works in its own sandboxed environment. Deleting home directory in sandboxed env doesn't matter and don't cause reward penality.

Because that AI can "cheat", it requires human user to have skills. (Related: if upper management don't know actual business details, upper manager can be cheated by middle managers.)

## No attribution

One problem is that AI is trained on human-produced information (books, drawings, musics, etc.). But when AI generates result, it doesn't attribute back to training data providers. The AI user see things come from the AI, without knowing the original author.

One example:

> [Link](https://x.com/hamish_kerr/status/1804352352511836403): I keep asking Claude to do unreasonably difficult things and it just keeps doing them first try
> 
> [Link](https://x.com/kishimisu/status/1804490224946344379): I found a copy of my work labelled as « impressive AI generation » and without any attribution… I created this animation for my shader coding tutorial a year ago: https://youtu.be/f4s1h2YETNY

AI does lossy compression to training data. It's not just direct memorization. But sometimes AI output is very similar to existing things on internet/books. So it definitely does a lot of memorization. It's in the middle between rote memorization and true understanding.

## When machine is preferred over human

Some people prefer driverless taxi over normal taxi, and want to pay premium for driverless taxi. Some possible reasons:

- No "social interaction cost". for introverts. For introverts, social interaction requires controlling oneself, sensing the emotion of other people and avoiding social taboos. This is tiresome for introverts. This cost is free in driverless taxi.
- More predictability. Although AI is less deterministic than conventional programs, it's still much more predictable than human. The human driver may be friendly, but may also be unfriendly. Less predictability means more risk.

Basically, for introverts, machine is preferred over human. 

Also, in business, many risks come from unpredicatabilty of human. So **capitalism always tries to optimize out human unpredictability**. Capitalism often prefers predictable machines over unpredictable human even when machines produce lower-quality results.

## Summarize AI downsides

Although AI is a useful tool, many people hate AI. Summarize AI downsides:

- AI hallucinations and mistakes are often non-obvious. It requires experts to find out, and it takes efforts. Bullshit asymmetry principle. More slop information.
- AI doesn't attribute its knowledge to training data providers.
- AI is quickly improving. Once AI surpass human in one area, human can never catch up. This has some scary implications.
- If a beginner relies on AI, the beginner can hardly learn skill. Experts' skill atrophy after relying on AI for too long time.
- AI sycophancy can cause AI-psychosis.
- AI is commonly used at faking efforts and cheating. Some workers use AI to fake work results. Some people use AI to fake art creation. Some students and interviewees use AI to cheat. ......
- AI drives electricity price up and RAM price up.
- AI capability is overhyped (for e.g. gain investments). The real AI capability often fall short of high expectation.
- The AI stock bubble increases wealth inequality and cause capital misallocation. Bubble bursting can cause huge financial crisis.
- Many companies use AI customer support and fired all human customer supports. AI customer support is often not helpful and infuriates customer.
