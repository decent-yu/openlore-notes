# GPT-6.1 Terra

> **来源**: [https://simonwillison.net/2026/Oct/26/gpt-61-terra](https://simonwillison.net/2026/Oct/26/gpt-61-terra)  
> **作者**: Simon Willison | **发布时间**: 2026-10-26T00:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: GPT-6.1 Terra, 大语言模型, 视觉推理, 计算机操作, 模型选型, 推理成本

---

## AI 导读

文章围绕一个选型问题展开：处于价格中间档位的 GPT-6.1 Terra，在哪些任务中值得使用。作者先摘录发布介绍，再引用一位 OpenAI 人员的评论，将图像理解、电脑操作和技术表达列为主要关注点。该评论认为，Terra 的技术表达可能优于 Astra 和 Sol，图像理解与电脑操作能力相近且速度更快，但多数其他能力稍弱；这些属于被引用者的判断，正文没有提供独立验证。文章报告其每百万输入词元收费 1 美元、每百万输出词元收费 5 美元；词元是模型处理和计费的文本单位。作者还展示了不同推理档位生成的鹈鹕图，并记录最高档位一次生成花费 11.3161 美分、耗时 4 分 11 秒。文章据此提供了偏重价格和任务类型的初步选择线索：需要图像处理、电脑操作或清晰表达时可以考虑该模型，订阅用户也可能借此缓解使用额度限制。不过，单次绘图记录不能证明这些任务中的整体表现，正文缺少统一条件下的速度、质量和可靠性测试，链接所指材料也未在正文中展开。

---

## 原文整理

2026 年 10 月 26 日 — 链接博客

**[GPT-6.1 Terra](https://openai.com/index/introducing-gpt-6-1-terra/)**（[转引自](https://news.ycombinator.com/item?id=50248124)）观察新模型的发布公告着重强调哪些方面，总是件有意思的事。

对于 GPT-6.1 Terra，OpenAI 重点介绍了视觉推理和电脑操作：

> 我们推出了 **GPT-6.1 Terra**。这款模型速度快、效率高，既能进行细致的视觉推理和精确的电脑操作，也能写出更清晰、更自然的文字。与 GPT‑5.6 Terra 相比，它有了显著提升，擅长处理日常工作中的各种沟通和电脑任务，从起草提案、解读技术图纸，到填写表单、编辑视频。

Terra 的定位与 Anthropic 模型系列中的 Sonnet 相同：既不是最便宜的，也不是最贵的，这让人更难决定什么时候该用它。

电脑操作就是一个很好的例子：这类任务能从价格较低的模型中受益，因为你可能会让模型去执行自动化的重复任务。

OpenAI 的 Ted Sanders 在 Hacker News 的[一条评论](https://news.ycombinator.com/item?id=50248124#50249160)中给出了一些有用的建议：

> 就 GPT-6.1 Terra 而言，我认为它在技术表达方面可能比 Astra 和 Sol 更好，视觉和电脑操作能力与它们相近（但速度快得多！），其他大多数方面则稍逊一些。对于 API 开发者，如果你想要一个比 Sol 便宜、又比 Luna 聪明得多的模型，Terra 是个不错的中档选择，尤其适合视觉、电脑操作和沟通任务。对于订阅用户来说，这次发布算不上轰动；如果你经常用到额度上限，它或许能让你多用一些；如果你想要更清晰的表达或更快的电脑操作，它也可以是个不错的选择。

这里是各个推理档位生成的[鹈鹕图](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F855d951d12c68eccca5374cd7d4f834e)。这张[使用“max”档位生成的鹈鹕图](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F855d951d12c68eccca5374cd7d4f834e#reasoning-2)花费了 [11.3161 美分](https://www.llm-prices.com/#it=16&ot=22629&sel=gpt-6.1-terra)，耗时 4 分 11 秒：

![鹈鹕和自行车都画得不错：车架形状正确，双脚踩在车架两侧的脚踏板上，配色与 GPT-6 和 GPT-6.1 系列生成的其他图片一致。](assets/diagram-0d67da728aa4.webp)

这里还有一张[新的对比图表](https://static.simonwillison.net/static/2026/openai-pelicans-october-2026.html)，展示了 OpenAI 当前最先进的模型阵容在各个推理档位下生成的鹈鹕图，包括 GPT-6 Astra、GPT-6.1 Sol、GPT-6.1 Terra 和 GPT-6 Luna。

GPT-6.1 Terra 的定价是每百万输入词元 1 美元、每百万输出词元 5 美元，接近 GPT-5.6 Terra 和 GPT-6.1 Sol 价格的一半，约为 GPT-6 Luna 的 10 倍。

## 近期文章

- [Battle Drones：一款通过氛围编程开发、利用音频实现多人联机的即时战略游戏](https://simonwillison.net/2026/Oct/20/battledrones/) — 2026 年 10 月 20 日
- [你好，Python 3.15；再见，3.10；现在我可以用 3.11 了](https://simonwillison.net/2026/Oct/12/python-3-15/) — 2026 年 10 月 12 日
- [用我的声音为博客开发了一项新功能](https://simonwillison.net/2026/Oct/9/built-using-my-voice/) — 2026 年 10 月 9 日
- [Claude Haiku 5.5](https://simonwillison.net/2026/Oct/7/claude-haiku-5-5/) — 2026 年 10 月 7 日