---
date: 2025-12-20
tags:
  - AI
  - Programming
unlisted: false
---
# My Views about AI

<!-- truncate -->

## Intelligence is high-dimensional

Many people tend to simplify intelligence to a one-dimensional IQ value. **Intelligence is high-dimensional**. 

For example, even before ChatGPT, a calculator can do arithmetic better than any a mathematician, but the calculator is not "smarter" than mathematician.

Many people tend to treat LLM chatbot as similar to human, because most familiar form of intelligence is human. However, LLM is different to human in many fundamental ways. Deep learning is very different to how human brain works. 

I agree with "jagged intelligence" idea:

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

## AI provides emotional value

Most people want to be recognized, praised and emphasized. People need emotional value. 

But emotional value is often scarce in human-to-human interactions. Person A praising/emphasize person B gives emotional values to B. But if A don't sincerely do so but is forced to do so for other reasons, then A consumes their own emotional value.

AI can provide tons of cheap emotional value. AIs are trained to please the user. The AI itself doesn't need to be recognized/respected like a person.

If one person cannot get emotional value from real human interaction, they tend to gain emotional value from AI. Related: [Chatbot psychosis](https://en.wikipedia.org/wiki/Chatbot_psychosis)

Also, AI is always "patient" in responding your questions. No matter how "silly" the question is. Asking entry-level questions to an expert often consume out the expert's patience.

In human-to-human relationships, often only recriprocal relations can sustain. But AI can provide emotional value without you giving AI anything.

## About AI Coding

### Focus too much on current task

Current LLMs are trained to finish specific tasks. The LLM tend to overly focus on current task, then it will "care less" about things like future code maintenance and security.

Also AI tend to use complex solutions to solve a problem. Although the complex solution sometimes work, the added complexity adds new sources of bugs. It adds tech debt and is problematic when project is complex.

Often the bug is partially caused by AI overcomplicating simple things. When human want to fix vibe-coded bug, the first thing to do is to simplify out unnecessary complexity. 

Vibe-coded app may contain security issues. But if you ask AI to do security review it can find the issue. AI "knows" security but still write insecure code because it was trained to "focus" on finishing current task. The RL rewards are usually simple and don't consider things like security and future maintenance.

### Save time on learning how to use API

A lot of time in programming is spent on how to use an "API". The "API" here is generalized, including language features, framework usage, config file format, how to deploy, etc.

The design of API has a lot of ad-hoc idiosyncracies. For example, adding one thing can be named "insert", "create", "add", "put", "new", "register", "spawn", etc. Also, reading a file could be `open`, `files.open`, `os.open`, `fs::open`, `openFile`, `files.read`, `readFile`, `new FileInputStream`, `new ifstream` etc. Many other such examples.

Which exact word/phrase it chooses is ad-hoc. It cannot be guessed without learning. Having to learn these ad-hoc API design is an obstacle in programming that's usually not fun. And it's different in each language/framework. Knowing the API of reading file in Python is not helpful in Java.

But if I tell AI to "read this file" then AI knows how to use the API.

But AI's ability of using API is bad for rarely used tools/libraries/frameworks/languages. It's correlated with how much related training data and how much related RL is done.

### AI capability sensitive to complexity

AI coding performs good in simple projects. The new projects are simple in the beginning. AI coding is good at them. But AI coding doesn't perform so good in a large existing codebase.

The good architecture design that can isolate complexity makes coding easier for both human and AI. 

In large codebase it's often that after changing A then B also need to be changed to make it keep working. When B and A are far away (in different foldrs) then AI may only change A and don't change B so it breaks. Sometimes type system can catch the issue. But when it involves config file, or cross-language things, or implicit invariants, then type system cannot catch it.

### Good for new side projects

When vibe coding on a new side project, the overall experience is likely good (or even addictive). New project is simple in the beginning. LLMs are good at coding in simple projects.

Vibe coding is much easier than manual coding. No need to recall about language syntax or how to use an API. No need to search about doc about an API. No need to figure out which library to use. The prompt is often much shorter than code.

Model performance also matters. If the model is weak and make mistakes often, vibe coding only gives frustration. Vibe coding only experience good when model is strong. 

Note that model capability is domain-specific: the model may be good at Python scripting or React web dev but sucks at writing device driver in C. It's highly dependent on training data and model RL targets.

But when applying vibe coding in existing large codebases it tend to perform worse. Changing existing large project without breaking existing functionality is hard even for expert programmers.

Writing code is easy, but writing working code is hard. When there is a complex bug that asking AI alone cannot fix, there will be frustrations. Also other bugs can emerge after fixing one bug.

### Confusing different things with similar wording

This issue is commonly encountered in AI coding. For example, `index` can mean the index in different things in different context. 

To alleviate this issue, the naming should be more informative, such as `index_of_xxx`, `index_of_yyy_in_zzz`. Also tensor name should describe meaning of each dimension ([see also](https://medium.com/@NoamShazeer/shape-suffixes-good-coding-style-f836e72e24fd)). All context-dependent things should include context in name or comments nearby.

Having more informative naming also helps human.

### Feels faster but maybe actually slower

In this study: [Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity](https://arxiv.org/pdf/2507.09089), developers feels that using AI make developing faster but it's actually slower.

Related: https://x.com/QuentinAnthon15/status/1943948791775998069

### Jagged capability

> https://x.com/karpathy/status/1977758204139331904
> 
> Good question, it's basically entirely hand-written (with tab autocomplete). I tried to use claude/codex agents a few times but they just didn't work well enough at all and net unhelpful, possibly the repo is too far off the data distribution.

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

[^enough_information]: Strictly speaking, code doesn't contain enough information for whole software to run. It may dynamic link another program in system, or it may download plugin etc. from internet. Also, runtime data can be interpreted as code, but how it's interpreted is determined by existing code. The overall idea is that in conventional programming the details are either explicitly specified or delegated to other deterministic software and hardware.


## Context rot issue

When context is long, LLM will perform worse. For example, ignore some instructions, ignore some important details in context.

When using AI chat, frequently opening new sessions could improve result quality.

## No continuous learning

You cannot easily "teach" the AI. You can write things and put into context. This can work as LLM has in-context learning ability. But due to context rot, in-context learning has limitation.

In current architecture, the most reliable way is still to encode knowledge into model weights.

Another way is to put your training data to internet, then AI companies will crawl it and use it to train their next model. However it's often slow. AI comanies don't redo pretrain every week, as pretrain is expensive. Even if AI companies use your new training data, it will only include in the next released model. AI companies don't release new model every week. 

Also AI companies don't give a formal way to submit training data to them (maybe due to fear of some people maliciously submitting poisonous training data).

## RL reward source

The behavior of AI is highly shaped by RL. Doing RL requires judging reward for model. Different kinds of reward source:

- Human judge. AI companies hire human to judge and often reward by judge count, not judge quality (judge quality is hard to measure). There there will be "human reward hacking": employed human tend to judge quickly by intuition to maximize income. So AI is trained to give **fancy superficial signal that can confuse intuitions**. The AI output looks good by first glance. But an expert can find it's full of nuanced mistakes. But normal people often won't notice the nuanced mistakes.
- Given some fixed problems with fixed answers. Only give reward if answer exactly matches. This can be useful for improving test score.
- Use other program (e.g. unit test) to judge result. For example, if AI-written code passes unit test it gets reward. But there may be bugs in reward judging code. AI may utilize bugs to gains reward without doing what you want AI to do. This is called **reward hacking**.

"Reward hacking" is also common in human society. [Perverse incentive](https://en.wikipedia.org/wiki/Perverse_incentive).

## Slop prevails when people cannot judge quality

[Lemon market problem](https://en.wikipedia.org/wiki/The_Market_for_Lemons): The sellers know the quality of the lemons. But the buyers don't know and is hard to judge from lemon appearance. There is an information asymmetry. The result is that good lemon is undervalued. Bad lemons prevail the market.

How to solve that problem? One important way is **reputation**. When a seller is honest about the lemon quality, people communicate about the information and improve seller's reputation. When seller cheats about lemon quality, people also communicate information and reduce seller's reputation.

Also, for each person that accept reputation information, they also need to judge the quality by existing reputation of information provider, as there are false informations.

AI is very good at faking superficial signals. The AI-written articles use related jargons that looks palusible for non-experts. The AI-written code will also superficially do things you asked, although it may use an API wrongly or violate an invariant so it won't work. The AI-generated photos looks real unless you squint details.

The problems is that faking superficial signal is easier than generating actually high-quality content. This problem already exists before AI. Some human are also good at faking superficial signals. But AI makes it much easier.

## Benchmark score is not representative

It's hard to test how good a model is. The possible space of tasks is very high-dimensional. And some tasks are hard to judge.

[Goodhart's law](https://en.wikipedia.org/wiki/Goodhart%27s_law): When a measure becomes a target, it ceases to be a good measure.

The popular benchmarks (e.g. Humanity's last exam, SWE bench verified) are also AI companies' important optimization targets. They will not do obvious cheating of putting test set into training set. But there are many other ways to indirectly hack the benchmark.

> ([Link](https://www.reddit.com/r/MachineLearning/comments/1pgqbjd/d_how_did_gemini_3_pro_manage_to_get_383_on/)) isparavanje: Tech companies have been paying PhDs to generate HLE-level problems and solution sets via platforms like Scale AI. They pay pretty well, iirc ~$500 per problem. That's likely how. I was an HLE author, and later on I was contacted to join such a programme (I did a few since it's such good money). Obviously I didn't leak my original problems, but there are many I can think of.

## AI detection race

Some people want AI output to be as similar to human output as possible. Some people want to detect whether content is written by human as accurate as possible. There is a constant race.

In old days the em dash "—" is frequently used by AI so it's seen as "AI smell". This affects human writers who use em-dash normally.

The new "AI smell" is negation: "It's not X, it's Y." 

The AI companies are probably trying to use RL to reduce usages of em dash and negation.

There are AI detections tools like Pangram. They can detect AI in some sense but the detection result can never be fully accurate (even if it shows "100% AI" it may actually be 80% probably of AI-written). Because it's not fully accurate, it shouldn't be used as sole source as discrediting a piece of writing.

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
- But if AI can do 70 scores, then AI will be much more useful in that task, although the performance just jumpted from 60 to 70. 
- If AI can reach 75 scores, then the economical utility of using AI may increase 10 times.

Near the threshold, incremental improvements do big changes.

Also note that intelligence is high-dimensional. Surpassing human in one task doesn't necessarily destroy jobs in that area. 

The nonlinearity-near-threshold effect exists in other domains. Making a software 10% easier to use may double its userbase.