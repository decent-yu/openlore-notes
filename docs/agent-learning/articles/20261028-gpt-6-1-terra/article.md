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

26th October 2026 - Link Blog

**[GPT-6.1 Terra](https://openai.com/index/introducing-gpt-6-1-terra/)** ([via](https://news.ycombinator.com/item?id=50248124)) It's always interesting to note which aspects of a new model are emphasized in the release announcement.

In the case of GPT-6.1 Terra, OpenAI went with visual reasoning and computer use:

We’re introducing **GPT-6.1 Terra**, a fast, efficient model that combines detailed visual reasoning and precise computer use alongside clearer, more natural writing. A significant step up from GPT‑5.6 Terra, it excels at the communication and computer-based tasks that fill the workday, from drafting proposals and interpreting technical drawings to filling out forms and editing videos.


Terra sits in the same position as Sonnet in the Anthropic model collection - not the cheapest or the most expensive, making it harder to decide when to use it.

Computer use is a good example of a task that benefits from a less expensive model, since you might put it to work on automated, repetitive tasks.

OpenAI's Ted Sanders provided some useful advice in [a comment](https://news.ycombinator.com/item?id=50248124#50249160) on Hacker News:

In the case of GPT-6.1 Terra, I'd say Terra is probably better than Astra and Sol at technical communication, similar on vision and computer use (but much faster!), and a bit worse on most other things. For API developers, it's a nice midrange option if you want something cheaper than Sol but a lot smarter than Luna (especially for vision / computer use / communication). For subscribers, it's not a bombshell release - if you're hitting usage limits, maybe it helps you go a bit further, and if you want clearer communication or faster computer use, it can be a nice option to select.


Here are [the pelicans](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F855d951d12c68eccca5374cd7d4f834e) for the different reasoning levels. This pelican [for "max"](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F855d951d12c68eccca5374cd7d4f834e#reasoning-2) cost [11.3161 cents](https://www.llm-prices.com/#it=16&ot=22629&sel=gpt-6.1-terra) and took 4 minutes and 11 seconds:

![It's a good pelican and a good bicycle - correctly shaped frame, feet n pedals either side of the frame, has the same color scheme as the others in the GPT-6 and GPT-6.1 families.](assets/diagram-0d67da728aa4.webp)


Here's [a new comparison grid](https://static.simonwillison.net/static/2026/openai-pelicans-october-2026.html) showing pelicans for every reasoning level of OpenAI's current frontier lineup: GPT-6 Astra, GPT-6.1 Sol, GPT-6.1 Terra, and GPT-6 Luna.

GPT-6.1 Terra is priced at $1/million input tokens and $5/million output - almost half the price of GPT-5.6 Terra and GPT-6.1 Sol, and around 10x the price of GPT-6 Luna.

## Recent articles

- [Battle Drones - a vibe coded RTS with audio-based multiplayer](https://simonwillison.net/2026/Oct/20/battledrones/) - 20th October 2026
- [Hello Python 3.15, goodbye 3.10, and now I can use 3.11](https://simonwillison.net/2026/Oct/12/python-3-15/) - 12th October 2026
- [A new feature for my blog, built using my voice](https://simonwillison.net/2026/Oct/9/built-using-my-voice/) - 9th October 2026
- [Claude Haiku 5.5](https://simonwillison.net/2026/Oct/7/claude-haiku-5-5/) - 7th October 2026
