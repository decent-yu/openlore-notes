# Jev and the rise of decision models: what you need to know and why they matter

> **来源**: [https://www.thoughtworks.com/insights/blog/generative-ai/jev-decision-models-what-need-know-why](https://www.thoughtworks.com/insights/blog/generative-ai/jev-decision-models-what-need-know-why)  
> **作者**: Ken Mugrage | **发布时间**: 2026-10-12T00:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: 决策模型, 概率建模, 模型评估, 代码重构, 编码代理, 人机协同, 模型选型

---

## AI 导读

文章讨论在需要快速分类、评分和选择的任务中，通用生成模型可能带来较高成本、延迟及超出需求的文本输出，并介绍 Jev 这一受约束的决策模型。按正文描述，用户提供上下文和候选选项，模型返回概率分布或置信度，支持真假判断、数值评分和选项选择三种模式。文中以解析代码库并按代码异味筛选重构候选为案例，转述其相较 Claude 启动更快、输出更聚焦的使用体验；进一步提出将其嵌入编码代理的外围执行框架，评估生成代码，并通过置信度决定是否引入人工审核。文章认为，这类模型可与生成模型互补，拓展成本、速度与控制要求较高的应用选择，同时承认其能力更受约束，市场关注也受到命名和定位影响。局限在于没有披露模型架构、训练过程、接口代码、测试集、具体延迟或成本测量，也未验证置信度校准和门控有效性；概率输出能否提升可靠性、实际能节省多少成本及适用边界，均无法由本文确认。

---

## 原文整理

Jev was launched in the middle of September 2026 and has been met with considerable attention and acclaim. It’s an AI model, but not an LLM: an LLM generates text, with novelty and unpredictability a feature, while Jev delivers only structured, probabilistic decisions. It represents a step towards a different kind of AI model; one that might be termed ‘decision models’ or ‘system one’ models.


These are not only smaller and more lightweight but more importantly geared towards an entirely different set of problems and use cases. In turn, they’re helping to push a conversation about when and where different models are appropriate, challenging the dominance of frontier AI.

## How does Jev work?


To understand how Jev actually works, it’s helpful to think about it through the lens of determinism and nondeterminism. One of the features (and challenges) of LLMs is that they’re nondeterministic. Jev, though, isn’t nondeterministic: it’s *probabilistic*.


My colleague [Ben O’Mahony](https://www.thoughtworks.com/profiles/b/ben-o-mahony), Principal AI Engineer at Thoughtworks and the author of O’Reilly’s forthcoming [Building AI Agent Platforms](https://www.oreilly.com/library/view/building-ai-agent/0642572243906/), has been an early adopter of Jev. Comparing Jev to LLMs, he explains that while models the industry has been familiar with over the last couple of years have a mechanism to enable randomness “to nudge them into more novel and interesting responses,” Jev “doesn’t have that injection of randomness.” Instead of text, he continues, the model responds with the probability distribution of the set of choices it has been given by the user.


“It has three modes of operation,” O’Mahony says. “The most simple is a ‘noul’, which is basically a true or false. You provide it with the context — what true would be and what false would be — and then it makes a judgment and gives you what it thinks and it's confidence.


“The other two are ratings and choices. A choice is which of these options should you choose? and a rating is almost the same, but with numbers. So, you define what those numbers are and you then end up with a numerical rating.”

## Why Jev matters now


This matters now for a number of different reasons. First is the issue of managing costs: using frontier models has become [more and more expensive](https://www.thoughtworks.com/insights/blog/generative-ai/dangers-token-usage-billing), and so there has been increasing effort and innovation into new kinds of models and new ways of using them. There has, for instance, been growing interest in open-weight and local models as businesses and engineers seek alternatives to established models.


However, growth in those areas has largely been driven by improved capabilities; the gap between frontier AI and open-weight models is closing. But Jev is a different kind of story altogether. Yes, its lightweight nature improves cost effectiveness, but in terms of capabilities it is, in fact, far more constrained.


However, as Ben demonstrated to me, this is what makes it so attractive. It’s well-suited to tasks that require rapid evaluation and analysis and where a rank or a decision is more helpful than strings of prose. One particular use case that Ben demonstrates to me is a simple refactoring task: Jev crawls through a codebase, using a parsing tool, to find candidates to refactor (based on a list of refactoring code smells).


I ask Ben how this process would have compared to a world before Jev and what’s different now. “What I like about this is how fast it is. I could go now and use Claude, but it takes a while to even get started.” He shows me how Claude performs the same task, noting how many tokens it consumes compared to Jev. While Claude delivers far more options and more extensive and detailed feedback, this isn’t really necessary for this particular use case. “There’s benefits here” Ben notes, “but actually, which of these am I going to have more confidence in not breaking my code base? Because there's 10,000 things that \[Claude\] could do.”

## Using Jev and coding agents like Claude together


It’s important to emphasize that we’re not talking about an either/or here. In fact, Jev’s highly constrained, probabilistic and evaluative capabilities could play well with a tool like [Claude](https://www.thoughtworks.com/radar/tools/claude-code). Ben mentions Jev could be used in the [harness](https://www.thoughtworks.com/insights/podcasts/technology-podcasts/what-harness-engineering) around a coding agent, acting as a kind of evaluative guardrail for the code that the agent produces.


Over the last couple of years, the response to AI-generated content, including code included, has been a technique called [LLM as a judge](https://www.thoughtworks.com/radar/techniques/llm-as-a-judge), where an LLM is trained adversarially to evaluate the output of another, larger LLM. Such an approach is not only expensive, its inherent non-determinism makes it unreliable as a judge. In comparison, something like Jev gets around these issues by evaluating data in a more probabilistic manner; indeed, it can even supply users with confidence scores. In other words, Jev makes it easier to think about and integrate the [human-in-the-loop](https://martinfowler.com/articles/exploring-gen-ai/humans-and-agents.html): it’s an effective ‘confidence gate’.

## Why now?


Given Jev is a more constrained and smaller model than what we’re seeing playing out in the world of frontier AI, it’s reasonable to ask *why now? Surely this should have arrived earlier?*


Following the release of Jev, Nandakishor Mukkunnoth, the creator of [Laya](https://laya-ai.com/), a similar model, [published an article](https://laya.convaiinnovations.com/) bemoaning that a frontier lab was presenting something he had developed and published on more than a year earlier as a breakthrough. However, while the architecture long precedes Jev’s release, Laya was only released as a packaged open-weight model product a couple of days after Jev’s launch.


“There’s no major innovation here,” Ben notes. However, the combination of an OpenAI alumnus (Diogo Almeida spent four years at the company) and exceptional branding and positioning have allowed it to cut through and make an impact. “I think they've done really strong comparisons of how you'd actually use this thing, and how the latency of it actually makes it really useful for a lot of use cases that people really want to use large language models for, but latency gets in the way.”

### 'System one' models: a new category


He also highlights that TypesafeAI, Jev’s creators, have introduced a new concept called a ‘[system one model](https://typesafe.ai/blog/introducing-system-one-models-and-jev)’. Based on the framework developed by Daniel Kahnemann in his book *Thinking Fast and Slow* (system one thinking being fast and intuitive, while system two is analytical and slower) the term is intended to flag what makes this distinctive from established and familiar AI models.


While it’s easy to dismiss the use of terms like this as little more than marketing or branding, they nevertheless play an important role in making technological developments easier to understand and, in turn, easier to find actual valuable applications for. And although Mukkunoth’s chagrin is understandable, it’s likely that interest in Laya exploded following Jev’s launch.

![Ben O Mahony, Thoughtworks Ben O Mahony, Thoughtworks](assets/diagram-e662c7f66cc9.jpg) 

The idea of an 'agentic organization': how much value does it have to generate for it to be worthwhile to have people burning a thousand dollars of tokens a day? Now, with Jev, we're burning a dollar a day — that's a single Claude Code subscription.

## A new era of AI models


Good marketing, particularly in the technology industry, is largely about timing: saying the right thing and introducing ideas at the right time. That’s what the team behind Jev have done; they’re responding to and then driving a particular moment where organizations and individuals are becoming frustrated with the AI model status quo, whether that’s because of costs, performance or control.


What Jev and the category it’s pushed into the market has done is to create even more space for the engineers and others to rethink the way they use models. We’ve seen this already over the last six months with the rise of open-weight and local models and growing interest in model routing techniques and platforms, but Jev may have caught the imagination of the industry because it tells a particularly succinct story and has clear and neat use cases, the value of which can be seen almost immediately.


“I think for me, the idea of an 'agentic organization' — how much value does it have to generate for it to be worthwhile to have people burning a thousand dollars of tokens a day?” Ben says. “But now, \[with Jev\] if we're burning a dollar a day — $30 at the end of the month — that's a Claude Code subscription, no problem.”
