# ttok 1.0

> **来源**: [https://simonwillison.net/2026/Oct/9/ttok/](https://simonwillison.net/2026/Oct/9/ttok/)  
> **作者**: 未知 | **发布时间**: 2026-10-09T00:34:43+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: Agent, Engineering

---

## AI 导读

本文深入探讨了 ttok 1.0 的架构设计与工程实践，重点分析了系统实现机制与核心权衡取舍。

---

## 原文整理

<p><strong>Release:</strong> <a href="https://github.com/simonw/ttok/releases/tag/1.0">ttok 1.0</a></p>
        <p>I released <a href="https://simonwillison.net/2026/Oct/8/ttok/">ttok 0.4</a>, ran <code>uv tool upgrade ttok</code>, piped a file into the new version... and realized that it was defaulting to the GPT-4 tokenizer when it should very clearly default to GPT-5/GPT-6 instead!</p>
<p>I figured switching the default was a reasonable excuse to finally ship a 1.0.</p>
<p>OpenAI haven't actually confirmed that GPT-6 uses the same tokenizer as the GPT-5 family yet - there's an <a href="https://github.com/openai/tiktoken/issues/608">angry issue about it</a> - but I found <a href="https://github.com/williamliu-ai/token-count-compare/commit/8d8a2178538bb37beb548fc37378135e4ff4b0c8">this commit</a> by William Liu which reports on an experiment he ran confirming that the tokenizers are likely the same:</p>
<blockquote>
<p>All seven GPT models (5.5, 5.6 Sol/Terra/Luna, 6 Astra/Sol/Luna) report 44,794 tokens and match each other on every one of the 31 fixtures. GPT-6 introduces no input-count change on this corpus.</p>
</blockquote>
    
    
        <p>Tags: <a href="https://simonwillison.net/tags/projects">projects</a>, <a href="https://simonwillison.net/tags/ai">ai</a>, <a href="https://simonwillison.net/tags/openai">openai</a>, <a href="https://simonwillison.net/tags/generative-ai">generative-ai</a>, <a href="https://simonwillison.net/tags/llms">llms</a>, <a href="https://simonwillison.net/tags/tokenization">tokenization</a></p>
