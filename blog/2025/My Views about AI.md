---
date: 2025-12-20
tags:
  - AI
unlisted: true
---
# My Views about AI


## Intelligence is high-dimensional

Many people tend to simplify intelligence to a one-dimensional IQ value. **Intelligence is high-dimensional**. For example, even before ChatGPT, a calculator can do arithmetic better than any human, but the calculator is not necessarily "smarter" than human.

Since ChatGPT, LLMs become popular. Many people tend to treat LLM chatbot as similar to human, because most familiar form of intelligence is human.

However, LLM is different to human in many fundamental ways. Deep learning is very different to how human brain works. 

I agree with "jagged intelligence" idea:

> Jagged Intelligence. Some things work extremely well (by human standards) while some things fail catastrophically (again by human standards), and it's not always obvious which is which, though you can develop a bit of intuition over time. 
> 
> Different from humans, where a lot of knowledge and problem solving capabilities are all highly correlated and improve linearly all together, from birth to adulthood.
> 
> \- Andrej Karpathy, [Link](https://x.com/karpathy/status/1816531576228053133)

- LLM is good at many things that are hard for human. LLM's knowledge is larger than any individual human.
- LLM is bad at many things that are easy for human.

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

Due to the commercial-driven optimization target, LLMs tend to be sychophants and please the user.

> Me to my AI-enabled smart fridge in 2038: Do we have any milk left
> 
> My fridge: Wow. Now that's a question worth exploring. By asking me something like that, you've proven that you're not thinking in ordinary ways—you're dialed in to what's really vital about food. Let's dive in.
> 
> \- LBark, [Link](https://x.com/franzsherbert/status/1923395242734060005)

(The sychophancy is reduced in new versions of LLMs as the psychological problem caused by sychophancy become more widely acknowledged.)

When trying to brainstrom with LLM, don't tell them your existing thoughts. 

LLM's behavior is very context-dependent. Sometimes it will defend the things they said in previous context. Starting a new session can make LLM output differently for the same question. 

Both human and LLM can "hallucinate" in consistent way. When human forgets something, human tend to make up consistent information to fill the hole in memory. LLM's hallucinations are seemingly plausible (maximize likelihood), not just random. LLM's hallucinations tend to be confident, much more confident than average human.

<details>

There is a misleading diagram about Jagged Intelligence. That diagram cleverily utilizes **framing bias**, as it drawns complex high-dimensional thing as simple two-dimensional thing:

> ![](./cog_bias/misleading_jagged_intelligence.png)
> 
> [Link](https://x.com/tomaspueyo/status/1993360931267473662)

Review from Terence Tao:

> This two-dimensional image has been circulating recently as an metaphor for the current state of AI technology. 
> 
> It is admittedly an improvement over one-dimensional narratives in which AI development is presented as a linear (or exponential) progression from sub-human to super-human intelligence. However, it is still a significant oversimplification. 
> 
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

</details>

Also, AI changes people's value judgement. Before ChatGPT if people see a long article with fancy writing they would think the author payed efforts in writing it. But now AI can easily write a long article with similar vibe. Now fancy writing is treated as "AI smell".

## Value of art


People tend to **judge the value of art by the cost of producing**. If one sees a beautiful image and thinks it's good art, then when they know it's AI-generated, the same image suddenly becomes cheap.

In some sense, when people appreciate art, they are appreaciting the efforts of human behind art, not just art itself.

However, many old people don't recognize AI and often treat AI output as real good content.

## A search engine that understands context

If you know one thing's name, you can easily search it via search engine. But there are many cases that you can describe one thing's traits but don't know the name of that thing. LLMs are good at this. They can tell you the name of that thing. 

LLMs can hallucinate, but after knowing the name of the thing you can use search engine to verify.

## Confusing different things that has similar wording

This issue is commonly encountered in AI coding. For example, `index` can mean the index in different things in different context. To alleviate this issue, the naming should be more informative, such as `index_of_xxx`, `index_of_yyy_in_zzz`. Similarily all context-dependent things should include context in name.


## In coding: tend to overcomplicate

In coding, AI tend to use complex solutions to solve a problem. Although the complex solution sometimes work, the added complexity adds new sources of bugs. It adds tech debt and is problematic when project is complex.

This is probably related to RL. 

## Vibe coding is "addictive"

Vibe coding is much easier than manual coding. No need to recall about language syntax or how to use an API. No need to search about doc about an API. No need to figure out which library to use. The prompt is often much shorter than code.

(The original meaning of "vibe coding" by Karpathy is to not see the code at all. My definition of "vibe coding" here also includes occasonally review code but don't manually edit code.)

After seeing some unwanted things in code, just ask AI to change it. The AI almost always obeys you. This gives satisfaction because human like the feeling of holding power, even if power applies to AI, not human.

Model performance also matters. If the model is weak and make mistakes often, commanding it only gives frustration. Vibe coding only experience good when model is strong. Note that model capability is domain-specific: the model may be good at Python scripting or React web dev but sucks at writing device driver in C.

However, writing code is easy, but writing working code is hard. When there is a complex bug that asking AI alone cannot fix, there will be frustrations. Also other bugs can emerge after fixing one bug.

Often the bug is caused by AI overcomplicating simple things. When human want to fix vibe-coded bug, the first thing to do is to simplify unnecessary complexity. 

But knowing what can be simplified and what cannot requires some programming experience, and that experience is mostly gained by manual coding. So it's kind of irony that fixing vibe-coded bug require manual coding experience.

## "Taste" in coding

Guiding vibe coding requires "taste". Taste is a kind of aesthetics preference. It's often that there are many ways to implement the same requirement. But some is more "beautiful" than others according to the taste.

The "beauty" is subjective. The "beauty" may come from:

- Simpler, more decoupled, easier to understand, easier to maintain.
- Code is more explicit but longer. It's easier to understand (Golang philosophy).
- Code is shorter but harder to understand. It may use some advanced math theory (e.g. use advanced category theory in Haskell). 
- It's more complex but it can easily accomodate to the predicted future requirement (e.g. can scale to handle millions of users). Although the predicted future requirement often never come.
- It uses an interesting way to satisfy some constraint, like puzzle solving. The constraint may be performance, fitting into rigid API, reduce code size (or satisfy Rust borrow checker).
- The design is clever that a small piece of code can handle millions of corner cases.

The "taste" and "beauty" is subjective and different people's view often conflicts.

## Context rot issue


## No continuous learning

You cannot easily "teach" the AI. You can write things and put into context. This can work as LLM has in-context learning ability. But due to context rot, in-context learning has limitation.

In current architecture, the most reliable way is still to encode knowledge into model weights.

Another way is to put your training data to internet, then AI companies will crawl it and use it to train their next model. However it's often slow. AI comanies don't redo pretrain every week, as pretrain is expensive. Even if AI companies use your new training data, it will only include in the next released model. AI companies don't release new model every week. 

Also AI companies don't give a formal way to submit training data to them (maybe due to fear of some people maliciously submitting poisonous training data).

## Where RL is good at

With reinforcement learning:

- Anything that can be auto-verified can be used for RL effectively
- Anything that can be easily judged by human can be used for RL effectively

|                               | Easy to verify         | Hard to verify  |
| ----------------------------- | ---------------------- | --------------- |
| Easy to collect training data | AI is good at this     | Full of AI slop |
| Hard to collect training data | AI is not good at this | Still hard      |

However there are some problems:

- If you use a unit test to judge AI coding result, AI may do **reward hacking**. Exploit the bug in the test. AI gains reward without doing what you want AI to do.
- For human judged: AI companies hire human to judge and often reward by judge count, not judge quality (judge quality is hard to measure). There there will be "human reward hacking": employed human tend to judge quickly by intuition to maximize income. So AI is trained to give **fancy superficial signal that can confuse intuitions**. The AI output looks good by first glance. But an expert can find it's full of nuanced mistakes. But normal people often won't notice the nuanced mistakes.

## Experts' knowledge undervalues

AI reduces the cost of doing many things, including the things that only experts can do before. So some experts' knowledge undervalues. This is one reason of disliking AI.

## Slop prevails when people cannot judge quality

[Lemon market problem](https://en.wikipedia.org/wiki/The_Market_for_Lemons): The sellers know the quality of the lemons. But the buyers don't know and is hard to judge from lemon appearance. There is an information asymmetry. The result is that good lemon is undervalued. Bad lemons prevail the market.

How to solve that problem? One important way is **reputation**. When a seller is honest about the lemon quality, people communicate about the information and improve seller's reputation. When seller cheats about lemon quality, people also communicate information and reduce seller's reputation.

Also, for each person that accept reputation information, they also need to judge the quality by existing reputation of information provider, as there are false informations.

## AI detection race

Some people want AI output to be as similar to human output as possible. Some people want to detect whether content is written by human as accurate as possible. There is a constant race.

