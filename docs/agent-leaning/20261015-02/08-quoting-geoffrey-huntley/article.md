# Quoting Geoffrey Huntley

> **来源**: [https://simonwillison.net/2026/Oct/13/geoffrey-huntley/](https://simonwillison.net/2026/Oct/13/geoffrey-huntley/)  
> **作者**: 未知 | **发布时间**: 2026-10-13T13:32:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: Agent, Engineering

---

## AI 导读

本文深入探讨了 Quoting Geoffrey Huntley 的架构设计与工程实践，重点分析了系统实现机制与核心权衡取舍。

---

## 原文整理

<blockquote cite="https://ghuntley.com/agency/"><p>If the agent isn't using my diagnostics library, I could introspect and update AGENTS.md, but there's still a delta, and it'll drift. Instead, I write a pre-commit hook that checks the git diff for direct calls to observability modules that bypass my diagnostics module. Boom: fail the commit, inject a prompt telling the agent to use the diagnostics library. All of a sudden it's ten out of ten.</p></blockquote>
<p class="cite">&mdash; <a href="https://ghuntley.com/agency/">Geoffrey Huntley</a>, nobody should type code by hand anymore. here's what's left.</p>

    <p>Tags: <a href="https://simonwillison.net/tags/coding-agents">coding-agents</a>, <a href="https://simonwillison.net/tags/generative-ai">generative-ai</a>, <a href="https://simonwillison.net/tags/git">git</a>, <a href="https://simonwillison.net/tags/agentic-engineering">agentic-engineering</a>, <a href="https://simonwillison.net/tags/ai">ai</a>, <a href="https://simonwillison.net/tags/geoffrey-huntley">geoffrey-huntley</a>, <a href="https://simonwillison.net/tags/llms">llms</a></p>
