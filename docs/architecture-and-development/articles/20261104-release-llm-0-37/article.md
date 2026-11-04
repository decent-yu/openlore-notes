# Release: llm 0.37

> **来源**: [https://simonwillison.net/2026/Nov/2/llm](https://simonwillison.net/2026/Nov/2/llm)  
> **作者**: Simon Willison | **发布时间**: 2026-11-02T00:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: LLM, 命令行工具, Python, OpenAI-compatible API, 多模态嵌入, AI 辅助代码审查, 开源维护

---

## AI 导读

这篇发布说明介绍 LLM 工具约两个月积累的功能和修复。为方便接入更多模型，新版加入两个模型，并提供配置兼容 OpenAI 接口规范的模型服务的命令。对于没有自行设置默认模型的用户，默认模型改为 gpt-6-luna，作者称其价格是此前默认模型的一半，但未说明计费口径。新版还允许用于嵌入处理的不同二进制内容分别声明内容类型；嵌入是将内容转换为向量表示，这项变化用于适配能够处理文本、音频、图像和视频的模型。作者另称，本次包含 12 位贡献者提供的修复，几乎都借助了人工智能；他使用模型辅助筛选，将这些贡献视为附带示例的缺陷报告，按情况直接合入或自行修复，感受到的审核负担低于预期。正文未展示具体修复、实现代码、使用示例或效果测量，因此可用于了解升级方向，尚不足以验证兼容性、成本收益或这种审核方式的普遍效果。

---

## 原文整理

2nd November 2026

There's a *lot* of stuff in this new release of my LLM CLI tool and Python library, representing around two months of accumulated features and fixes.

I added `gpt-6.1-sol` and `gpt-6.1-terra`, plus a new [llm openai endpoint add](https://llm.datasette.io/en/stable/other-models.html#configure-an-openai-compatible-model) command to make it easier to configure additional models that use OpenAI-compatible endpoints.

I switched the default model (if you have not set your own default) to `gpt-6-luna` - half the price of the previous default, `gpt-5.6-luna`.

LLM's support for embeddings now accepts that different binary content may have different content-type - necessary for models like [Gemini Embedding 2](https://deepmind.google/models/gemini/embedding/), which can embed text, audio, images, and video.

A lot of open source maintainers complain about a flood of AI-assisted PRs. LLM is my project that has been most affected by this and, to be honest, I'm not finding it to be a problem. That's mainly because I have Opus 5.5 or GPT-6 Astra help me with triage to spot the most useful fixes!

This release includes fixes from 12 authors - almost all AI-assisted - and the burden of reviewing them wasn't nearly as great as I expected. I really see them as bug reports with attached examples - sometimes I land them, sometimes I implement my own fix.

So thanks to [0d00ciallo0721](https://github.com/0d00ciallo0721), [ac1982](https://github.com/ac1982), [Allen Flux](https://github.com/allenflux), [Bian Yuhan](https://github.com/bianyh), [Bishop Kammeraad](https://github.com/porkedchop), [Chen Yufeiyang](https://github.com/feiiiiii5), [Christopher Krah](https://github.com/chrikrah), [Ethan Stoner](https://github.com/ethanstoner), [Jiachen Xie](https://github.com/er-s-an), [Olamide Faniyan](https://github.com/Pajimo), [Sharat Chandra Sai Boddu](https://github.com/sharatchandrasai999-sketch), [nosmile99](https://github.com/nosmile99) for helping out with (or spending tokens on) this release!

## Recent articles

- [The U.S. Census Bureau and “Super Intelligence”](https://simonwillison.net/2026/Nov/2/census-bureau-super-intelligence/) - 2nd November 2026
- [The October inflection point for vibe game development](https://simonwillison.net/2026/Oct/28/vibe-game-development/) - 28th October 2026
- [Battle Drones - a vibe coded RTS with audio-based multiplayer](https://simonwillison.net/2026/Oct/20/battledrones/) - 20th October 2026
- [Hello Python 3.15, goodbye 3.10, and now I can use 3.11](https://simonwillison.net/2026/Oct/12/python-3-15/) - 12th October 2026
