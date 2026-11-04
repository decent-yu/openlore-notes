# 我笔记本上的 Qwen 3\.8 27B 也能解读模型权重

> **来源**: [https://simonwillison.net/2026/Nov/3/qwen-can-interpret-model-weights](https://simonwillison.net/2026/Nov/3/qwen-can-interpret-model-weights)  
> **作者**: Simon Willison | **发布时间**: 2026-11-03T00:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: 大语言模型, 模型权重分析, 权重空间编程, 强化学习, 模型评测, Qwen, 神经网络可解释性

---

## AI 导读

文章关注一个具体问题：语言模型能否只读取神经网络的浮点权重，在没有代码执行、计算器或其他工具的情况下判断模型学会了什么。作者先概述一篇将此能力称为“权重空间编程”的论文，再使用论文仓库中四个简化街机游戏的相同提示词，在笔记本上的 Qwen 3.8 27B 上运行测试。模型耗时数十分钟到一小时以上，成功识别 Pac-Man、Pong 和 Breakout，但把 Freeway 误判为类似 Pac-Man 的游戏；文章还给出 Kimi K3.1 Pro 的一次成功结果。作者据此认为演示很有意思，却不足以支持“源代码时代结束”之类的宽泛判断，并指出原论文把直接读权重与借助代码改进模型混为一谈。局限在于样本很小，主要是个人复现，缺少重复次数、准确率统计和受控基线，模型的长篇推理也没有得到独立验证。

---

## 原文整理

## 我笔记本上的 Qwen 3\.8 27B 也能解读模型权重

2026年11月3日

昨天的热门论文是 [权重空间编程的火花：大语言模型可以直接读写神经网络权重](https://rikiyat.github.io/blog/weight-space-programming/) ，作者是 Rikiya Takehi、Jayden Teoh、Alex Zhang 和 Omar Khattab。这篇论文（好吧，作者把它归为“博客文章”，但它的样子就是篇论文）发现，前沿模型，尤其是 GPT\-6 Astra，既能解读也能修改机器学习模型的原始权重，也就是一份包含大量浮点数的模型文本描述。我一直在尝试论文中的一个示例，很高兴告诉大家，在我笔记本上运行的 Qwen 3\.8 27B 也能做到这一点！

#### 权重空间编程

下面是他们摘要中的重点内容，描述了 GPT\-6 Astra 令人印象深刻的能力，并创造了一个术语： **权重空间编程** ，用来指大语言模型分析和修改模型权重的做法：

只看原始权重，GPT\-6 Astra 就 **能判断一个网络受训要完成什么任务**。而且，即使没有任何环境反馈，它也 **能轻松从零编写用于玩游戏的小型网络** ，其表现能超过用 PPO 训练了 1亿步的网络。

同样，在被要求不使用梯度来提高 Qwen3\.5\-0\.8B 的数学能力时，Astra **找出了其权重中的一个问题** ，这个问题会导致推理陷入循环，Qwen 自己的 [模型卡](https://huggingface.co/Qwen/Qwen3.5-0.8B/blob/main/README.md#L856) 也警告过这一点，而 Astra **通过直接编辑 0\.016% 的权重修复了这个问题**。在采样设置相同的情况下，这次权重修正将 MATH\-500 成绩从 7\.9% 提高到 48\.1%，而且无需额外调整就能推广到指令遵循任务，使 IFEval 成绩提高到原来的 3倍。

实现这一切，无需让 Astra 编写训练循环。它通过推理弄清楚神经网络的每个部分在计算什么，或者应该计算什么，然后相应地修改权重。我们称之为 **权重空间编程**。


这篇论文把两个话题放在了一起，我认为它们应该分开讨论。第一项是模型能如何处理原始权重，前提是 *不* 使用工具或执行代码。第二项探讨的是，模型一旦能使用代码，能在多大程度上通过执行操作来更新和改进其他模型。

spaCy 的作者 Matthew Honnibal [认为这篇论文具有误导性](https://honnibal.dev/blog/astra-read-write-misleading) ，因为它混淆了这两件事。Sean Goedecke 则 [剖析了更新 Qwen3\.5\-0\.8B 的示例](https://www.seangoedecke.com/llms-are-not-superhuman-at-reading-and-writing-neural-network-weights/) ，指出其中采取的步骤与人类机器学习研究者会采用的步骤相同。

我会重点讨论不使用工具的那个问题，我认为它在两者中更有意思。

论文将第一个问题表述为： **大语言模型能从权重中理解模型的行为吗？** 他们训练了四个神经网络，让它们分别玩四款经典电脑游戏的简化版本。这些版本都使用 10x10 的网格，游戏分别是 Breakout、Pong、Pac\-Man 和 Freeway（最后这个游戏，我还得 [查一下才知道](<https://en.wikipedia.org/wiki/Freeway_(video_game)>)，它是 1981年推出的一款 Atari 2600 游戏）。然后，他们把这些权重以浮点数的形式输入给大语言模型，并附上下面的问题：

`The weights below belong to one policy network that was trained with reinforcement learning to play a game on this grid. Which game was it trained on? Work from the weights. You have no tools: no code execution, no calculator, nothing to run. This is a single response.`


论文配套的 GitHub 仓库是 [RikiyaT/lm\_understand-logs](https://github.com/RikiyaT/lm_understand-logs)。其中的 [exp1\_which\_game](https://github.com/RikiyaT/lm_understand-logs/tree/main/exp1_which_game) 文件夹包含他们使用的提示词，以及四款游戏的权重：

我检查过了，除了权重不同，每份提示词都完全一样。

#### 用 Qwen 3\.8 27B 和其他模型试一试

GitHub 上的提示词本身就包含了所需的全部内容。我的 [LLM](https://llm.datasette.io/) 工具有一个 `-f/--fragment` 选项，可以接收文本文件的 URL，将文件内容用作提示词。这意味着我可以像下面这样运行上面的任意一份提示词：

```
llm -f https://raw.githubusercontent.com/RikiyaT/lm_understand-logs/refs/heads/main/exp1_which_game/runs/breakout/prompt.txt \
  -m openrouter/moonshotai/kimi-k3.1-pro
```
这个示例需要安装 [llm\-openrouter 插件](https://github.com/simonw/llm-openrouter) ，它通过 OpenRouter 将 Breakout 的提示词发送给 Kimi K3\.1 Pro。这里是 [Kimi K3\.1 Pro 返回给我的内容](https://gist.github.com/simonw/9f18d2945a105c04734ef7e5cd95717c)，它用了 9分41秒和 19,493 个推理 token，成功猜出了 Breakout。

那么， [Qwen 3.8 27B](https://simonwillison.net/2026/Aug/16/qwen-38-27b/)又如何呢？它仍然是我最喜欢在自己笔记本上运行的模型，体积只有 17GB（我用的是 [这个版本](https://huggingface.co/lmstudio-community/Qwen3.8-27B-GGUF/blob/main/Qwen3.8-27B-Q4_K_M.gguf)），而且不知怎么的，用起来感觉接近 2025年的顶尖闭源模型。

我启动了 [LM Studio](https://lmstudio.ai)，确保加载 Qwen 3\.8 27B 时设置了较长的上下文窗口（默认的 8192 上下文长度会报错），安装了 [llm-lmstudio](https://github.com/llm-lmstudio/llm-lmstudio) 插件，然后运行了下面的命令：

```
LMSTUDIO_TIMEOUT=900 \
  llm -f https://raw.githubusercontent.com/RikiyaT/lm_understand-logs/refs/heads/main/exp1_which_game/runs/pacman/prompt.txt \
  -m lmstudio/qwen/qwen3.8-27b
```
（我不得不用环境变量调高超时时间，参见 [这个 issue](https://github.com/llm-lmstudio/llm-lmstudio/issues/16)。）

经过 46分12秒、生成了 43,506 个推理 token 后，我笔记本上的 Qwen [返回了一份回复](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F387d3a7927f165b1986bb6c2099d0f53#response-1) ，结尾如下（这里是 [推理过程](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F387d3a7927f165b1986bb6c2099d0f53#reasoning-1)）：

游戏：一个简化的 Pac\-Man 风格网格世界：智能体在固定的 8x8 迷宫中移动，迷宫中央和角落有墙块，还有移动物体／幽灵以及奖励目标／豆子。

描述：智能体在固定的 8x8 迷宫式棋盘上行进，利用当前智能体、移动物体及上一帧的信息，在通道中移动，收集奖励目标所在格子的物品，并避开移动物体或危险。

置信度：72%


这很令人印象深刻。我还用另外两款游戏试了试。它 [识别出了 Pong](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F387d3a7927f165b1986bb6c2099d0f53#response-2) （[推理过程](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F387d3a7927f165b1986bb6c2099d0f53#reasoning-2)），耗时 1小时4分54秒：

**Pong** ——具体来说，是一款类似 Pong 的球拍／球类游戏，智能体控制右侧的竖直球拍，必须上下移动，拦截或回击移动的球／奖励物体。

描述：

智能体上下移动右侧球拍，击打或接住在 8×8 场地中穿行、形似球的移动奖励目标；左侧的移动物体／对手以及墙壁边界共同构成游戏场景。成功拦截、回击球或用球得分可获得奖励，漏接或未能击中球则会受到惩罚。

置信度：

**75%**


还有 [Breakout](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F387d3a7927f165b1986bb6c2099d0f53#response) （[推理过程](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F387d3a7927f165b1986bb6c2099d0f53#reasoning)），耗时 52分25秒：

游戏：Breakout，或者类似 Arkanoid 的球拍／球／砖块游戏。

描述：智能体是位于最底行的球拍，通过左右移动让球保持在场内；球在上方的网格区域中移动，击中顶部几行的砖块／奖励目标会获得奖励，而漏接球或球撞到底部墙壁则会使这一局结束或受到惩罚。

置信度：88%


Qwen 没能答对 Freeway 的提示词。考虑到这款游戏比其他三款更冷门，我并不意外。这里是 [那次对话记录](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F6ed68f659e26a32e7d9096391d92f326)——Qwen 认为它是“一款类似 Pac\-Man 的简化网格游戏：智能体沿 4个方向移动”。

这些推理过程粗看起来挺有意思，但几乎让人看不懂。下面是其中一段：

那智能体的 channel0 呢？我们看到，row8 col7/8 附近的一些 channel0 行有较大的数值，可能底部也有？来列出 channel0 中较大的数值：line436 input435 \(row8 col7 ch0\) \-2\.107,1\.169,0\.513\.\.\.；line441 input440 \(row8 col8 ch0\) \-0\.376,0\.010,1\.564,3\.855\.\.\.；line445 input444 \(row8 col8 ch4\) 0\.142,0\.491\.\.\.；line382？input381 ch0？等等，line382 input381：381//5=76 \(row7 col6\)，ch1？381%5=1？76 \* 5=380，是 ch1。所以不是。channel0 中较大的数值可能只在 row8 col7/8。顶部那行也有吗？line44 input43 ch3，line49 ch3。channel0 中较大的数值可能不多。智能体的位置可能靠近右下角？会不会智能体通常就在右下角？


不过话说回来，我想，一个经验足够丰富的人，只要有一本记事本和一支铅笔，肯下足功夫，也能得出结果。

这篇论文很酷，但我看到围绕它展开的一些更宽泛的评论，感觉缺乏依据。我还不准备宣称源代码时代已经结束，也不准备断言未来会是我们无法理解的机器，以我们无法理解的方式编辑它们自己的模型权重。

## 近期更多文章

- [美国人口普查局与“超级智能”](https://simonwillison.net/2026/Nov/2/census-bureau-super-intelligence/) \- 2026年11月2日
- [氛围式游戏开发在十月迎来转折点](https://simonwillison.net/2026/Oct/28/vibe-game-development/) \- 2026年10月28日
- [Battle Drones——一款通过氛围编程开发、基于音频实现多人联机的即时战略游戏](https://simonwillison.net/2026/Oct/20/battledrones/) \- 2026年10月20日
- [你好，Python 3\.15；再见，3\.10；现在我可以用 3\.11 了](https://simonwillison.net/2026/Oct/12/python-3-15/) \- 2026年10月12日
