# The five components of building a "company harness"

> **来源**: [https://www.langchain.com/blog/components-building-company-harness](https://www.langchain.com/blog/components-building-company-harness)  
> **作者**: Harrison Chase Caspar Broekhuizen | **发布时间**: 2026-10-24T23:49:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: 企业智能体, Slack 集成, 身份认证与授权, 智能体记忆, 沙箱代码执行, 凭据代理, 定时任务与事件驱动, Managed Deep Agents

---

## AI 导读

文章讨论企业智能体投入日常使用后遇到的五类问题，并以 LangChain 的销售、工程和营销应用说明处理方式。首先，把入口放在员工已经使用的 Slack 中，允许智能体另开线程，减少对人工讨论的干扰；遇到大量待审核邮件时，则使用独立界面。其次，根据工具和任务决定以谁的身份执行：用户发起的部分操作使用用户身份，后台任务或共享服务使用智能体自己的账号。邮件工具只在私聊中开放，公司社交账号的部分操作只允许指定用户调用，说明权限还要结合使用场景控制。
文章把记忆分为全体用户共享和个人专用两层，由智能体决定事实保存在哪一层，但作者明确表示这部分探索尚少。代码在沙箱（隔离的执行环境）中运行，智能体本身位于沙箱之外；不同应用使用不同依赖环境，并借助凭据代理连接需要认证的服务。任务既可由用户直接发起，也可按时间或事件自动运行，例如审查新建的代码合并请求、发送每日简报。
作者将这些做法归纳为企业智能体运行所需的通用配套能力，并说明内部应用使用同一托管平台搭建。正文给出了设计实例，但没有展示授权校验、记忆隔离、凭据代理的具体实现，也没有量化使用效果、成本或故障表现，因此更适合作为设计检查清单，尚不足以验证其通用性和运行保障。

---

## 原文整理

*How do you build a company harness?*

*We've built internal agents at LangChain across marketing, sales, and engineering, and they have completely changed how those teams work. They weren't easy to get right. We spent a lot of time solving the problems that come with a multiplayer harness, and every team landed on the same core primitives:*

- ***It lives where people already work (for us that's Slack)***
- ***It knows who's asking and acts OBO that person for some services and with its own identity for others***
- ***It has memory, scoped globally and/or at the user level***
- ***Code runs in a sandbox and can make requests to services which require auth***
- ***Large % of its runs are scheduled rather than explicitly invoked by someone (ambient agents!!)***

*Thank you to @caspar\_br for help on this blog*

## LangChain Internal Agents

To ground this discussion, a brief overview of the three main agents we have at LangChain:

**GTM Agent**: a proactive, always-on agent that assists with GTM activities. Monitors and responds to emails, to new qualified leads, sends daily briefs, can answer questions about accounts, can run outbound motions

**OpenSWE**: our engineering agent, can work on and draft PRs, but also runs code review over all PRs

**Content Studio**: our marketing agent, can draft assets (blogs, images, PowerPoints, videos) in our style and with knowledge of our positioning

All of these agents are built in the same way. We think this is a general recipe for building company harnesses.

![](assets/diagram-1777ef9d3773.png)

## Problem: User Experience

All our agents live where people already work, which for us is Slack. They each have their own [Slack apps](https://docs.langchain.com/langsmith/python/managed-deep-agents-channels-slack), and can be DM’d or tagged in a channel. One benefit of working with these agents in channels is that others can see how people use it.

One nice thing we added was the concept of “breaking” out a conversation in Slack. Oftentimes, a great conversation would be happening and you’d want to add in your agent to get their opinion or get them to do something. But if they ran in that thread it would pollute the human conversation and distract. So we added an easy way for agents to “breakout” into their own thread in the channel.

![](assets/diagram-8a61b7bd7a2b.jpg)

There are some user experiences that are best NOT done in Slack. For example, when reviewing emails GTM Agent wants me to send, it's fine to do that in a one-off way in Slack, but if there is a big backlog it's not a fun experience at all to track those from Slack messages. For experiences like that, we did end up building a custom UI.

![](assets/diagram-ffdd550f2d43.jpg)

## Problem: Identity

The thorniest and trickiest problem we tackled was identity, specifically: when these agents take actions, who do they take actions on behalf of?

There are two main ways this is handled in the industry:

- *On-behalf-of*: the agent acts on behalf of the user invoking the agent**
- *Fixed credentials*: there is a fixed set of credentials that the agent uses no matter who is invoking it (eg a service account)

This is how we handled it for each agent (docs: [Add identity to Managed Deep Agents](https://docs.langchain.com/langsmith/python/managed-deep-agents-identity), [Manage connections](https://docs.langchain.com/langsmith/python/managed-deep-agents-connections)):

#### OpenSWE:

- When you message it, it acts on behalf of you (eg will open up PRs with your GitHub handle)
- When it runs in background (for example, as a code review agent) it acts with its own identity (GitHub app)

#### GTM Agent:

- It has some tools (eg Salesforce access) where it uses a fixed identity, so not everyone has to log in
- It has some tools (eg email) where it acts on behalf of the user invoking the agent. These tools are ONLY available in DMs. In shared channels, they are not available

#### Content Studio:

- It has some tools (eg Notion) where it acts on behalf of the user invoking the agent
- It has some tools (eg Twitter) where it uses a fixed set of credentials, in this case for our company twitter account. Only a whitelisted set of users can invoke certain operations here though. Eg not everyone can post to Twitter (they may only be able to read/draft)

![](assets/diagram-13f844df265d.png)

## Problem: Memory

All agents have [two tiers of memory](https://docs.langchain.com/langsmith/python/managed-deep-agents-memory):

- Global memory: memory that applies to all users
- User memory: memory that only applies to the specific user

Agents decide which facts they want to remember to which level of memory.

This is probably the least explored area so far - we have a lot of work we want to do here!

## Problem: code execution

All three of our agents can write and execute code, and often need to in order to accomplish tasks. They all follow the pattern of running outside the sandbox, but connecting to it and executing code in there. They all have slightly different base images with different dependencies. OpenSWE (the most coding related agent) is intelligent about which base image it spins up (differs based on channel/user). They all take advantage of the [secret proxy](https://docs.langchain.com/langsmith/sandbox-auth-proxy) on [LangSmith Sandboxes](https://docs.langchain.com/langsmith/sandboxes) (see also [Add a sandbox to Managed Deep Agents](https://docs.langchain.com/langsmith/python/managed-deep-agents-sandboxes)) to let the agent connect to services in a safe and secure way from inside the sandbox itself.

![](assets/diagram-35e1329a1d9b.jpg)

## Problem: scheduled runs

A large percent of runs are scheduled or triggered in the background by events, rather than being directly kicked off by a user.

Some of these are set up by the developer of the agent. For example: OpenSWE listens for PRs and when a new one is created it reviews it.

Others of these are set up by the user asking the agent to do so. For example: I asked GTM agent to send me a daily brief, and it created its own [cron](https://docs.langchain.com/langsmith/python/managed-deep-agents-schedules) to do so.

![](assets/diagram-d939456bd7bd.png)

## All built on Managed Deep Agents

All of these agents are built on [Managed Deep Agents](https://docs.langchain.com/langsmith/python/managed-deep-agents-overview), which sits on top of the open source [Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview) harness. It provides infrastructure to solve all these problems (channels, crons, sandboxes, memory, identity) while providing enough flexibility to customize agent behavior as needed. Try it out today with the [quickstart](https://docs.langchain.com/langsmith/python/managed-deep-agents-quickstart)!
