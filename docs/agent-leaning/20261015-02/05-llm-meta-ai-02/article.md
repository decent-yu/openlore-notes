# llm-meta-ai 0.2

> **来源**: [https://simonwillison.net/2026/Oct/12/llm-meta-ai/](https://simonwillison.net/2026/Oct/12/llm-meta-ai/)  
> **作者**: 未知 | **发布时间**: 2026-10-12T23:58:46+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: Agent, Engineering

---

## AI 导读

本文深入探讨了 llm-meta-ai 0.2 的架构设计与工程实践，重点分析了系统实现机制与核心权衡取舍。

---

## 原文整理

<p><strong>Release:</strong> <a href="https://github.com/simonw/llm-meta-ai/releases/tag/0.2">llm-meta-ai 0.2</a></p>
        <p>Meta AI release <a href="https://research.meta.ai/blog/muse-wonder">Muse Wonder</a> (and Muse Wonder Full), their new "most capable models". These look like they may be Meta's first <a href="https://simonwillison.net/2026/Sep/27/2026-in-llms-so-far/#simon-willison-2026-in-llms.042.webp">Fable class models</a> - Artificial Analysis <a href="https://artificialanalysis.ai/models/muse-wonder">scores Muse Wonder (Max)</a> as equivalent to GPT-6.1 Sol.</p>
<p>I updated my <code>llm-meta-ai</code> plugin for the new model. Here are <a href="https://tools.simonwillison.net/markdown-svg-renderer?url=https://gist.github.com/simonw/185cff7ff0b11379725abf7f78d5d2b2">pelicans for Muse Wonder</a> for the six reasoning levels: minimal, low, medium, high, xhigh, max. This is the max one:</p>
<p><img alt="Blue cycling helmet, bicycle is missing one bar but otherwise an OK shape, has mudguards, nice background, OK beak." src="https://static.simonwillison.net/static/2026/muse-wonder-max.webp" /></p>
<p>For comparison, here are <a href="https://tools.simonwillison.net/markdown-svg-renderer?url=https://gist.github.com/simonw/1f8c25686971da82077e34c088d408db">pelicans for Muse 1.3</a> (released September 2nd) and <a href="https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2Fce974a21202b0595e36ec2a5ddb51480">Muse 1.2</a> (released August 5th). Meta AI are putting out a new model a month, and the improvements between each model (at least in terms of drawing pelicans) are material.</p>
    
    
        <p>Tags: <a href="https://simonwillison.net/tags/ai">ai</a>, <a href="https://simonwillison.net/tags/generative-ai">generative-ai</a>, <a href="https://simonwillison.net/tags/llms">llms</a>, <a href="https://simonwillison.net/tags/meta">meta</a>, <a href="https://simonwillison.net/tags/pelican-riding-a-bicycle">pelican-riding-a-bicycle</a>, <a href="https://simonwillison.net/tags/llm-release">llm-release</a></p>
