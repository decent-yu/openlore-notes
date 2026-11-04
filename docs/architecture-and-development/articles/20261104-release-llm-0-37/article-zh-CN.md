# 发布：llm 0\.37

> **来源**: [https://simonwillison.net/2026/Nov/2/llm](https://simonwillison.net/2026/Nov/2/llm)  
> **作者**: Simon Willison | **发布时间**: 2026-11-02T00:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: LLM, 命令行工具, Python, 兼容 OpenAI 的 API, 多模态嵌入, AI 辅助代码审查, 开源维护

---

## AI 导读

这篇发布说明介绍 LLM 工具约两个月积累的功能和修复。为方便接入更多模型，新版加入两个模型，并提供配置兼容 OpenAI 接口规范的模型服务的命令。对于没有自行设置默认模型的用户，默认模型改为 gpt-6-luna，作者称其价格是此前默认模型的一半，但未说明计费口径。新版还允许用于嵌入处理的不同二进制内容分别声明内容类型；嵌入是将内容转换为向量表示，这项变化用于适配能够处理文本、音频、图像和视频的模型。作者另称，本次包含 12 位贡献者提供的修复，几乎都借助了人工智能；他使用模型辅助筛选，将这些贡献视为附带示例的缺陷报告，按情况直接合入或自行修复，感受到的审核负担低于预期。正文未展示具体修复、实现代码、使用示例或效果测量，因此可用于了解升级方向，尚不足以验证兼容性、成本收益或这种审核方式的普遍效果。

---

## 原文整理

2026年11月2日

我的 LLM 命令行工具和 Python 库这次发布的新版本包含 *大量* 内容，汇集了约两个月来积累的功能和修复。

我新增了 `gpt-6.1-sol` 和 `gpt-6.1-terra`，还新增了一条 [llm openai endpoint add](https://llm.datasette.io/en/stable/other-models.html#configure-an-openai-compatible-model) 命令，让配置更多使用兼容 OpenAI 的端点的模型变得更容易。

我将默认模型（如果你没有自行设置过）改为了 `gpt-6-luna` ，价格只有此前默认模型的一半，此前默认使用的是 `gpt-5.6-luna`。

LLM 的嵌入功能现在允许不同的二进制内容使用不同的内容类型，这是支持诸如 [Gemini Embedding 2](https://deepmind.google/models/gemini/embedding/)这类模型所必需的，因为它能对文本、音频、图像和视频进行嵌入处理。

许多开源项目维护者抱怨，大量借助 AI 完成的 PR 涌入了他们的项目。在我的项目中，LLM 受这一现象的影响最大，但说实话，我并不觉得这是个问题。主要是因为我会让 Opus 5\.5 或 GPT\-6 Astra 帮我初步筛选，找出最有用的修复！

这次发布包含了 12 位作者贡献的修复，几乎都借助了 AI，而审查这些修复的负担远没有我预想的那么大。我确实把它们看作附带示例的缺陷报告：有时直接合入，有时自己实现修复。

因此，感谢 [0d00ciallo0721](https://github.com/0d00ciallo0721)、 [ac1982](https://github.com/ac1982)、 [Allen Flux](https://github.com/allenflux)、 [Bian Yuhan](https://github.com/bianyh)、 [Bishop Kammeraad](https://github.com/porkedchop)、 [Chen Yufeiyang](https://github.com/feiiiiii5)、 [Christopher Krah](https://github.com/chrikrah)、 [Ethan Stoner](https://github.com/ethanstoner)、 [Jiachen Xie](https://github.com/er-s-an)、 [Olamide Faniyan](https://github.com/Pajimo)、 [Sharat Chandra Sai Boddu](https://github.com/sharatchandrasai999-sketch)、 [nosmile99](https://github.com/nosmile99) 为这次发布提供帮助（或花费 token）！

## 近期文章

- [美国人口普查局与“超级智能”](https://simonwillison.net/2026/Nov/2/census-bureau-super-intelligence/) \- 2026年11月2日
- [凭感觉用 AI 开发游戏：十月的转折点](https://simonwillison.net/2026/Oct/28/vibe-game-development/) \- 2026年10月28日
- [Battle Drones：一款凭感觉用 AI 编写、通过音频实现多人联机的即时战略游戏](https://simonwillison.net/2026/Oct/20/battledrones/) \- 2026年10月20日
- [你好，Python 3\.15；再见，3\.10；现在我终于可以用 3\.11 了](https://simonwillison.net/2026/Oct/12/python-3-15/) \- 2026年10月12日
