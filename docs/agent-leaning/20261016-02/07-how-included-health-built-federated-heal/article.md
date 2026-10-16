# How Included Health Built Federated Healthcare Agents with LangGraph and Deep Agents

> **来源**: [https://www.langchain.com/blog/how-included-health-built-federated-agents-for-healthcare-navigation-with-deep-agents-and-langgraph](https://www.langchain.com/blog/how-included-health-built-federated-agents-for-healthcare-navigation-with-deep-agents-and-langgraph)  
> **作者**: Sofia Sulikowski | **发布时间**: 2026-09-17T15:14:00+00:00  
> **标签**: LangGraph, Multi-Agent, 人机协同, 可靠性评测

---

## AI 导读

文章介绍了医疗健康平台 Included Health 基于 LangGraph 与 Deep Agents 构建联合多智能体架构 Dot 的工程实践。针对传统决策树导航僵硬、无法应对多意图和紧急情况的痛点，核心方案采用主 Supergraph 进行全局路由与跨子工作流调度，利用虚拟文件系统管理上下文与共享技能（渐进式加载），并通过 LangGraph 的持久化执行机制实现无缝的人机协同（Human-in-the-loop）挂起与恢复；结合 LangSmith 临床标注队列与多轮用户仿真评测保障系统的可靠性与安全性。上线后实现交互率提升 75%、临床分诊匹配度超 95% 及高危识别率超 99%。

---

## 原文整理

[Included Health](https://includedhealth.com/) is an all-in-one healthcare platform that partners with employers and health plans to provide their employees and members with healthcare navigation to services like virtual primary care, behavioral health, urgent care, specialty care, and more. The product experience centers answering medical, financial, or administrative questions via Dot—an AI-powered healthcare guide built on top of a federated multi-agent architecture using Deep Agents and LangGraph.

## The challenge: healthcare navigation doesn't fit a decision tree

Healthcare is one of the few domains where what a person asks for and what they actually need can be entirely different. A member asking *"is an artery plaque scan covered by my insurance?"* might, with a few follow-up questions, reveal that they are managing elevated cholesterol and have a family history of heart disease. The right response includes the dollar figure—but it may also mean recognizing an opportunity to encourage a conversation with a primary care physician.

Historically, health systems handled this kind of routing with structured navigation trees. That approach made complex needs manageable for software, but only by flattening them into a series of predefined decisions. As Kartik Darapuneni, Engineering Manager, described it: *“For the member, it feels really rigid, and it’s just not a good experience.”* The limitations become even more consequential when a conversation begins with *“I’m having chest pain.”* The system needs to recognize the potential emergency in the first turn, not after seven clarifying questions.

This is the broader tradeoff that has shaped software for decades. To scale, technology has typically had to standardize complex human situations around the average case. In healthcare, where context is often the difference between a merely correct answer and a helpful one, that tradeoff is especially costly.

LLMs, combined with an agent harness, change what is possible. They can process dense individual health records, reason about ambiguous needs, and ask clarifying questions without forcing members through predetermined paths. *“LLMs addressed all three of those blockers all at once,”* said Kartik. Conversations not only become more natural, software no longer has to choose between personalization and scale. Included Health built a healthcare experience that adapts to each member’s context, responding with the urgency, guidance, and next step that their situation calls for.

## Agent architecture: a federated supergraph with Deep Agents

Included Health's production architecture centers on a main LangGraph graph they call the Dot supergraph. Within it, Dot acts as the primary conversational router for transactional interactions (e.g. handling coverage questions, billing inquiries) and also navigation to the right care point. A set of sub-workflows handle domain-specific member journeys including urgent care intake, appointment scheduling, finding a specialist, behavioral health, and more.

Different product teams at Included Health own different parts of this graph. Scheduling alone, for example, has to account for which services a member is eligible for, their coverage details, whether they're a primary member or dependent, and a range of clinical nuances.

Included Health added Deep Agents for consistency across those services. *"Originally, you would jump into a different agent and suddenly it was a lot more short and brusque. It didn't have the same voice and tone,"* said Rohan Bhandari, Staff Machine Learning Engineer. With Deep Agents, the team created a global platform prompt for voice and tone that could be passed across all agents without each team having to manage it independently. When routing from one workflow to another, Deep Agent’s filesystem and built-in context management allow the outgoing agent to summarize the conversation and pass both the summary and a file path to the full conversation history for the receiving agent. This setup ensures members never have to repeat themselves across different agents owned by different teams.

For example, the shared coverage question skill: coverage questions don't necessarily arrive at the start of a conversation. A member could be mid-way through finding a specialist and want to know what it will cost. Before Deep Agents, handling this required threading a coverage capability through every sub-workflow's routing logic. Now, *"we decomposed it into a platform sub-agent that all the Deep Agents can inherit. Meaning every agent can answer coverage questions,"* said Rohan.

LangGraph enables consistent composition and distributed development so each product team can build and own their service independently. Deep Agents adds the shared filesystem that keeps tone and behavior consistent as customer conversations move across those services.

## Skills as a capability registry

Included Health gives agents clinical capabilities and services using Deep Agent skills. Each skill describes what a service is, when it's appropriate, when it isn't, and how to handle edge cases. For example, what to do when a dependent wants to book a service that has eligibility nuances.

The model uses progressive disclosure as a way to drive the right conversation for navigating a member. For example, if a member says they want to see a doctor, there could be 3 or more appropriate ways to help (e.g. virtual urgent care, virtual primary care, find an in-person doctor). The agent has a skill registry managed through a virtual filesystem, and up front it gets a short description of each skill. Upon invocation, the model decides which skills are relevant and can then load full skill files. With those skill files, it learns about the nuances and what questions to ask to best navigate the member (e.g. do they want a virtual or in-person visit? Is the issue they are describing acute or better managed through a long term provider relationship?).

Included Health supports third-party employer benefits in addition to its own services, and is working toward encoding those as skills too, to include the 20 to 30 benefits per employer plan.

*We have a clinical team who reviews chats and confirms whether they agree with which care spot we sent a member to, given their issue,"* added Rohan. That feedback loop has allowed Included Health to tune skill definitions over time and stay above their target level of clinical routing agreement of 95%.

## Making human handoff a core design constraint, not an edge case

A distinctive aspect of Included Health's agentic system is how they incorporated human-in-the-loop to improve the experience for patients. *"We think about LangGraph as our entire messaging platform,"* said Kartik. LangGraph's durable execution allows the agent to maintain full context across the conversation, supporting indefinite pauses and context retention. When the agent reaches a point of uncertainty, it pauses the graph, routes to a human member care advocate for a multi-turn exchange, and then resumes—with the agent holding the full context of what the human did and said.

This design reflects the long-lived nature of healthcare relationships. A member can come back to the same thread days or weeks later with a follow-up question, and the agent can pick up where they left off, including the full context of any human-assisted portions. *"From a human perspective, they’re helping the agent get unblocked, as opposed to doing all of the work,"* said Kartik.

The architecture also leaves room for the next evolution: running a parallel agent thread while a human is handling a conversation, so the agent can do background research and surface recommendations to the care advocate in real time.

## Observability and continuous improvement with LangSmith

LangSmith annotation queues are central to how Included Health runs clinical oversight. Right now, every conversation goes into a queue for clinical team review. Reviewers assess whether the agent's navigation recommendation was correct, whether emergency guardrails triggered appropriately (or correctly did not trigger), and flag anything that needs follow-up. Those labels are exported from LangSmith into Included Health's data warehouse, where the data science team builds the operational metrics dashboards.

Multi-turn user simulation evals using LangChain's user simulation [package](https://docs.langchain.com/langsmith/multi-turn-simulation) became the safety net for architectural changes. The migration from standard agents to Deep Agents across the supergraph affected four product teams, all wary of breaking changes. *"We were able to run our whole eval suite, see that we got, for the most part, better performance, and then we had the confidence to share that out to the other teams,"* said Rohan. Thanks to these evals, the migration happened in under 2 weeks, with no significant regressions and no team resistance.

## Results

Dot launched to clients in August, in what Rohan described as “*the smoothest launch the team has seen in the past few years*.” Early metrics are tracking in the right direction across three areas:

- **Engagement:** Members engage with agents at a much higher rate leading to a 75% lift in chat engagement.
- **Clinical accuracy:** Clinicians agreed with Dot's care recommendation well over their 95% target in the conversations they graded. Included Health's clinical team labels conversations through LangSmith annotation queues, judging whether Dot pointed the member to the right care.
- **Clinical safety**: Dot identifies over 99% of high-risk situations, as validated by regular clinical audits. This high detection rate enables the team to proactively engage and support vulnerable members as quickly as possible.

*Interested in building production-grade agent systems with Deep Agents? Learn more about Deep Agents.*
