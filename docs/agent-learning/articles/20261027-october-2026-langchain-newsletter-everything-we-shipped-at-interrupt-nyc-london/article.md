# October 2026 LangChain Newsletter: Everything we shipped at Interrupt NYC & London

> **来源**: [https://www.langchain.com/blog/october-2026-langchain-newsletter](https://www.langchain.com/blog/october-2026-langchain-newsletter)  
> **作者**: The LangChain Team | **发布时间**: 2026-10-15T16:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: AI代理工程, 可观测性与执行追踪, 语义检索, 模型评估与红队测试, 模型微调, MCP工具集成, AI治理

---

## AI 导读

文章围绕团队构建、理解和改进生产环境 AI 代理的需求，汇总一组产品能力及学习资源。主要介绍通过自然语言语义搜索定位追踪记录，按会话时间顺序查看主代理与子代理的执行路径，利用红队测试及指标趋势发现问题，并宣称可自动测试和验证修复建议；同时介绍按运行上下文定制代理、用户级记忆、工具使用与任务调度，以及在 LangSmith 内发布自定义界面。模型改进方面，提出将生产执行轨迹转为训练数据，连接 Baseten 或 Fireworks 完成微调、评估和部署；开源更新提及通过 langchain.mcp 命名空间支持 FastMCP。文章还介绍结合人员、流程、持续评估与治理来扩展生产 AI 的框架，并列出相关培训活动。正文传达的主要判断是，这些能力能够增强团队对代理系统的控制并提高交付效率，但没有提供架构细节、复现步骤、评测数据或成本对比。客户案例仅定性声称功能交付变快，不能据此确认改善幅度或因果关系；链接指向的实现和案例详情不在本次材料范围内。

---

## 原文整理

Any company can use AI. The ones that see lasting value will [own their intelligence](https://www.langchain.com/blog/own-your-intelligence): the model, the harness, and the context that turn a generic model into one that knows your business.

At Interrupt NYC and London, we shipped new features to enable teams to own their intelligence, giving them more control over how they build, understand, and improve their agents in production.

## New in LangSmith

🔎 **LangSmith AI Search:** Find conversations based on a semantic understanding of your traces, with questions like “did the user get frustrated?” or “did PII leak?” without relying on traditional filters.

🤖 **Managed Deep Agents v0.9:** Build dynamic, personalized agents that are tailored to each run's context, react in Slack, have user-level memory and tool use, schedule tasks, and more. Plus, built-in web search for free.

🛡️ **LangSmith Engine v2:** Catch issues before they reach production with Red Teaming. Engine v2 also detects harder-to-spot issues like inefficient agent work and metric trends, and automatically tests and validates its proposed fixes.

🧭 **LangSmith Trajectories:** Navigate trace data and follow your agent’s path with a chronological view of each session across the main agent and its subagents.

🧩 **LangSmith Custom Apps:** Build, publish, and run custom interfaces directly in LangSmith, turning recurring workflows into shared tools built on your LangSmith data.

🎛️ **LangSmith Fine-Tuning:** Turn production trajectories into training data, fine-tune models with Baseten or Fireworks, then evaluate and serve the tuned models.

## Open source

🔧 **LangChain v1.4.0:** LangChain now supports FastMCP through the `langchain.mcp` namespace.

⚡ **Building production agents with Jev and LangGraph:** See how LangGraph orchestrates Jev, TypeSafe AI’s decision model, to build agents that are faster and cheaper to run.

## The Agentic Operating Model: How teams are running production AI at scale

![The Agentic Operating Model](assets/diagram-6a96f8897e43.png)

The [Agentic Operating Model](https://info.langchain.com/guide/the-agentic-operating-model) looks at how leading teams bring people, processes, and technology together to scale production AI. It covers rethinking the agent development lifecycle, building continuous evaluation practices, and embedding governance: what it takes to move beyond experimentation and run agents reliably at scale. Join an upcoming virtual session to explore the framework and what it looks like in practice.

**🌍 EMEA · Oct 29**10:00 GMT / 11:00 CET

[Register →](https://events.langchain.com/webinar/emea-the-agentic-operating-model/)

**🌎 North America · Nov 5**10:00 AM PT / 1:00 PM ET

[Register →](https://events.langchain.com/register/namer-agentic-operating-model/)

## Speak the Lang

Live workshops on the fundamentals of building reliable agents.

**🧪 Agent Evals 101: What are agent evals?**Wed, Oct 28 · 11:00 AM PT · Virtual

Learn how to measure agent performance by assessing its execution trajectory: the sequence of messages and tool calls it produces.

[Register now →](https://events.langchain.com/register/762756f4-4344-4760-b4b9-a07dd63e76af/)

**🧠 Agent Memory 101: Context engineering**Tue, Nov 3 · 10:00 AM PT · Virtual

A live workshop on how agents remember. See how Deep Agents, our open source agent harness, helps keep context windows lean.

[Register now →](https://events.langchain.com/register/agent-memory-101-context-engineering/)

## Upcoming events

### Hosted by LangChain

**🇩🇪 Munich, Germany · Oct 27 · 17:00–21:00 CET**LangChain x Data Reply Hackathon: Building with agent harnesses

[Register →](https://luma.com/g6kvvori)

**🤖 Chicago, IL · Oct 27 · 10:00 AM CT**LangSmith Roadshow: Building agents with agents

[Register →](https://events.langchain.com/LangSmithRoadshow/Chicago/)

**💻 Virtual · Oct 28 · 11:00 AM PT**GTM Engineering Live: How we built our design agent

[Register →](https://events.langchain.com/gtm-engineering-live-series/design-agent/)

**🏛️ Vienna, VA · Oct 29 · 6:00 PM ET**DMV Agent Builders Meetup

[Register →](https://luma.com/afqb2si8)

**🍁 Toronto, ON · Nov 4 · 5:15 PM ET**Own your intelligence with open‑weight models

[Register →](https://luma.com/htp51nvv)

### Hosted by the LangChain community

**🇲🇽 Mexico City · Oct 29 · 5:00 PM CT**Agents in production

[Learn more →](https://luma.com/mxtw-e354f23b)

## What teams are building with LangChain

Before, adding a feature to an early version of **Zip**’s chatbot or agent took weeks, and its in-house traces pipeline was missing details the team needed. Now, LangSmith and LangGraph make shipping new features faster.


**How can you follow along with the Lang Latest?** Check out the [LangChain blog](https://www.langchain.com/blog), [Changelog](https://docs.langchain.com/langsmith/changelog), and [YouTube channel](https://www.youtube.com/@LangChain) for more product and content updates.
