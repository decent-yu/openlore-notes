# How Orange Business Built Live Intelligence with LangChain and LangSmith

> **来源**: [https://www.langchain.com/blog/how-orange-business-built-live-intelligence-on-langchain-and-langsmith](https://www.langchain.com/blog/how-orange-business-built-live-intelligence-on-langchain-and-langsmith)  
> **作者**: Sofia Sulikowski | **发布时间**: 2026-10-13T16:50:00+00:00  
> **标签**: Agent架构, 工程可靠性, 可观测性, 回归测试

---

## AI 导读

文章介绍了 Orange Business 基于 LangChain 与 LangSmith 构建企业级 Agent 平台 Live Intelligence 的工程实践。为解决分散系统下 Agent 难以可靠编排和规模化治理的痛点，方案设计了 Studio 开发者环境、Portal 终端门户与 Console 管理控制台，并通过 Agent Registry 与 MCP Registry 集中管控工具鉴权和提示词版本。在可靠性方面，构建了覆盖 RAG 质量、Agent 工具调用与防注入行为、生产追踪监控的三层评估栈，并在变更发布前通过回放历史 Trace 执行回归评估，保障了复杂多 Agent 流程的稳定迭代。

---

## 原文整理

[Orange Business](https://www.orange-business.com/) is the B2B arm of Orange Group, one of Europe's largest telecommunications and digital services companies. [Live Intelligence](https://www.orange-business.com/en/solutions/data-ia-iot/live-intelligence) is Orange Business's flagship AI platform. It’s deployed across 120,000 Orange employees and has already been adopted by 150+ customers in France & Europe.

## The Limits of Building AI Assistance From Scratch

When Orange Business developed their first AI products, they wanted to provide customers with safe ways to develop value with AI.

Live Intelligence Portal lets employees create simple AI agents and trigger basic API calls—capabilities that work for entry-level personal tasks. Orange Business wanted to build next: agents that could orchestrate complete business processes across fragmented enterprise systems, govern who can access what, and run reliably in production at scale.

Reproducing the orchestration, deployment, monitoring, and observability infrastructure that enterprise-grade agents require, though, would have required over a year of development. Rather than build from scratch, Orange Business went looking for the right foundation. This is how Live Intelligence Studio was born.

## An Open-Source-First Evaluation Across 20 Vendors

Orange Innovation (Orange Group's innovation scouting arm) evaluated roughly 20 major cloud providers and specialized AI vendor solutions. They selected LangChain for its model neutrality and portability—allowing the team to maintain full ownership of the agent architecture, and manage workloads in response to European data sovereignty requirements.

For evaluations and observability, LangSmith was the strongest fit. *"The observability capabilities in LangSmith were significantly ahead of what we could realistically build internally,"* said Mathieu Ducrot, Head of AI Products at Orange Business. *"Building it ourselves would likely have delayed our roadmap twelve to eighteen months."*

## Live Intelligence: Studio, Portal, and Console

Live Intelligence is structured around three components, each targeting a different user.

**Live Intelligence Studio** is the environment for AI engineers and agent developers to design complex, cross-system workflows: an LLM-as-a-Service catalog (models hosted on Orange Business infrastructure and public cloud), an agent builder powered by [LangSmith Fleet](https://www.langchain.com/langsmith/fleet), MCP support, agent observability through LangSmith and orchestration tooling.

**Live Intelligence Portal** is the interface for business end users: HR teams drafting job descriptions, legal teams reviewing NDA clauses, software teams auto-generating test scripts, helpdesks building IT support agents. Workflows built in Live Intelligence Studio can be deployed directly into Portal, for a clean, governed end-user experience.

**Live Intelligence Console** gives enterprise administrators visibility and control: usage monitoring, performance tracking, and enforcement of security and compliance policies.

## Agent and MCP Registries for Cross-Org Governance

Beyond individual agents, Live Intelligence adds two organizational layers that distinguish it from simpler deployments.

The **Agent Registry** lets teams discover, reuse, version, and govern agents and prompts across the organization. Rather than every team rebuilding the same capability, agents become shared assets with proper lifecycle management. For prompt management, Orange uses LangSmith Context Hub to maintain different prompt versions for each environment. This enables testing and validation of prompt changes before making changes available in production, and without making changes to code.

The **MCP Registry** provides a centralized catalog of trusted tools, connectors, and external capabilities. Agents access enterprise systems (Microsoft 365, CRMs, internal APIs, and custom services) through a controlled, auditable layer.

Together, these registries pull business logic into the platform layer and out of prompts so each team doesn’t need to re-solve authentication, security, and governance from scratch.

## LangSmith Across the Full Evaluation Stack

Orange Business runs a 3-layer evaluation in production, built on LangSmith.

1. **RAG quality:** golden datasets with question-answer pairs, expected source documents, retrieval recall, grounding quality, and answer faithfulness.
2. **Agent behavior:** tool selection accuracy, task completion rates, refusal behavior, robustness against prompt injection, and adherence to business constraints.
3. **Production trace monitoring:** latency, cost, model usage, failed runs, human feedback, and escalation cases.

*"Before, debugging an agent was difficult because failures were hidden inside model behavior,"* said Mathieu. *"With LangSmith, engineers can inspect each step of the execution and understand whether the issue comes from retrieval, the prompt, the model, the tool call, or the business logic."*

With that shared visibility, AI engineers, backend engineers, product managers, and QA can look at the same traces and evaluation results rather than working from separate, disconnected views.

## Validating quality and preventing regressions before they impact users

Orange Business also uses LangSmith to enforce running regression evaluations before pushing changes to production. Engineers can compare agent versions, replay historical traces, and validate behavior before any deployment.

For complex workflows with multiple agents and tool calls interacting with each other, visibility into the full execution tree enables engineers to identify and fix broken connections or issues faster. Inspections for Orange typically include:

- Where exactly the workflow failed
- Which agent or tool was being executed at that point
- What input was passed to that step
- What response was received
- How long individual steps took
- How many agents and tools were executed as part of the workflow
- How the different steps contributed to the overall execution

In addition, Orange builds datasets with trace examples that represent either good or bad agent behavior to use in offline evals for continuous prompt refinement.

## Results and what's next

The system Orange built has improved operational efficiency across teams, giving engineers and SMEs a shared workflow for agent development and improvement.

*“For us, the technical team owns the application and agentic workflow implementation, while the business team can actively participate in prompt improvement and quality evaluation. LangSmith turns prompt iteration, tracing, and evaluation into one continuous improvement workflow,” said Mathieu Ducrot, Head of AI Products.* Orange plans to build on that improvement loop by expanding team collaboration into online evals for even richer feedback cycles.

Looking ahead, the team is working on automating more complex business processes. For example, agents that support the entire Insurance claim process, with CRM and Legal workflow integrations, as well as an agent that can monitor security sensors in a port for field-engineer emergency intervention.

*Building an enterprise AI platform? Learn more about LangSmith and Live Intelligence.*
