# Qwen 3.8 27B on my laptop can interpret model weights, too

> **来源**: [https://simonwillison.net/2026/Nov/3/qwen-can-interpret-model-weights](https://simonwillison.net/2026/Nov/3/qwen-can-interpret-model-weights)  
> **作者**: Simon Willison | **发布时间**: 2026-11-03T00:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: 大语言模型, 模型权重分析, 权重空间编程, 强化学习, 模型评测, Qwen, 神经网络可解释性

---

## AI 导读

文章关注一个具体问题：语言模型能否只读取神经网络的浮点权重，在没有代码执行、计算器或其他工具的情况下判断模型学会了什么。作者先概述一篇将此能力称为“权重空间编程”的论文，再使用论文仓库中四个简化街机游戏的相同提示词，在笔记本上的 Qwen 3.8 27B 上运行测试。模型耗时数十分钟到一小时以上，成功识别 Pac-Man、Pong 和 Breakout，但把 Freeway 误判为类似 Pac-Man 的游戏；文章还给出 Kimi K3.1 Pro 的一次成功结果。作者据此认为演示很有意思，却不足以支持“源代码时代结束”之类的宽泛判断，并指出原论文把直接读权重与借助代码改进模型混为一谈。局限在于样本很小，主要是个人复现，缺少重复次数、准确率统计和受控基线，模型的长篇推理也没有得到独立验证。

---

## 原文整理

## Qwen 3.8 27B on my laptop can interpret model weights, too

3rd November 2026

Yesterday’s hot paper was [Sparks of Weight-Space Programming: LLMs can natively read & write neural network weights](https://rikiyat.github.io/blog/weight-space-programming/) by Rikiya Takehi, Jayden Teoh, Alex Zhang, and Omar Khattab. The paper (OK, the authors classify is as a “blog post”, but it’s paper-shaped) found that frontier models, in particular GPT-6 Astra, can both interpret and modify the raw weights of a machine learning model—literally a text description of the model containing a bunch of floating point numbers. I’ve been playing with one of the examples from the paper and I’m delighted to share that Qwen 3.8 27B running on my laptop can do this as well!

#### Weight-Space Programming

Here’s the highlight from their abstract, describing impressive capabilities for GPT-6 Astra and coining the term **Weight-Space Programming** for when LLMs analyze and modify model weights:

Just by looking at the raw weights, GPT-6 Astra **can tell what a network was trained to do**. And without any environment feedback, it **has no trouble writing, from scratch, small game-playing networks** that outplay networks trained with PPO for 100 million steps.

Likewise, asked to make Qwen3.5-0.8B better at math without gradients, Astra **identifies an issue in its weights** causing reasoning loops that Qwen’s own [model card](https://huggingface.co/Qwen/Qwen3.5-0.8B/blob/main/README.md#L856) warns about, and **fixes this by natively editing 0.016% of the weights**. With the same sampling settings, this weight correction raises MATH-500 from 7.9% to 48.1% and generalizes out of the box to instruction following, raising IFEval by 3x.

All of this is possible without allowing Astra to write training loops. It uses its reasoning to figure out what each part of a neural network computes, or should compute, and changes the weights accordingly. We call this **Weight-Space Programming**.


The paper combines two topics that I think deserve to be treated separately. The first is what models can do with raw weights *without* running tools or executing code. The second explores how well models, once armed with code, can run operations to update and improve other models.

spaCy author Matthew Honnibal [called the paper misleading](https://honnibal.dev/blog/astra-read-write-misleading) for conflating the two, and Sean Goedecke [deconstructed the Qwen3.5-0.8B update example](https://www.seangoedecke.com/llms-are-not-superhuman-at-reading-and-writing-neural-network-weights/) as taking the same steps that a human ML researcher would use.

I’m going to focus on the no-tools question, which I think is the more interesting of the two.

The paper presents the first question as **Can LLMs understand model behaviors from weights?** They trained four neural networks to play simplified 10x10 grid versions of four classic computer games—Breakout, Pong, Pac-Man, and Freeway (I had [to look that one up](<https://en.wikipedia.org/wiki/Freeway_(video_game)>), it’s an Atari 2600 game from 1981). Then they fed those weights (as floating point numbers) to an LLM along with the following question:

`The weights below belong to one policy network that was trained with reinforcement learning to play a game on this grid. Which game was it trained on? Work from the weights. You have no tools: no code execution, no calculator, nothing to run. This is a single response.`


The GitHub repository that accompanies the paper is [RikiyaT/lm\_understand-logs](https://github.com/RikiyaT/lm_understand-logs). The [exp1\_which\_game](https://github.com/RikiyaT/lm_understand-logs/tree/main/exp1_which_game) folder contains the prompts they used, with weights for the four games:

I checked and, aside from the weights, each prompt is identical to the others.

#### Trying it out with Qwen 3.8 27B and other models

The prompts on GitHub are self-contained. My [LLM](https://llm.datasette.io/) tool has a `-f/--fragment` option which can be passed the URL to a text file to use as a prompt, which means I can run any of the above like this:

```
llm -f https://raw.githubusercontent.com/RikiyaT/lm_understand-logs/refs/heads/main/exp1_which_game/runs/breakout/prompt.txt \
  -m openrouter/moonshotai/kimi-k3.1-pro
```
This example requires the [llm-openrouter plugin](https://github.com/simonw/llm-openrouter) installed, and runs the Breakout prompt against Kimi K3.1 Pro via OpenRouter. Here’s [what I got back from Kimi K3.1 Pro](https://gist.github.com/simonw/9f18d2945a105c04734ef7e5cd95717c), which successfully guessed Breakout after 9m 41s and 19,493 reasoning tokens.

But how about [Qwen 3.8 27B](https://simonwillison.net/2026/Aug/16/qwen-38-27b/)? That’s still my favorite model to run on my own laptop—it’s just 17GB (I use [this version](https://huggingface.co/lmstudio-community/Qwen3.8-27B-GGUF/blob/main/Qwen3.8-27B-Q4_K_M.gguf)) and somehow feels close to the leading proprietary models of 2025.

I fired up [LM Studio](https://lmstudio.ai), ensured Qwen 3.8 27B was loaded with a long context window (the default 8192 context failed with errors), installed the [llm-lmstudio](https://github.com/llm-lmstudio/llm-lmstudio) plugin and ran this command:

```
LMSTUDIO_TIMEOUT=900 \
  llm -f https://raw.githubusercontent.com/RikiyaT/lm_understand-logs/refs/heads/main/exp1_which_game/runs/pacman/prompt.txt \
  -m lmstudio/qwen/qwen3.8-27b
```
(I had to bump the timeout up using an environment variable, see [this issue](https://github.com/llm-lmstudio/llm-lmstudio/issues/16).)

After 46m 12s and 43,506 reasoning tokens, Qwen on my laptop [returned a response](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F387d3a7927f165b1986bb6c2099d0f53#response-1) that ended like this (here’s [the reasoning trace](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F387d3a7927f165b1986bb6c2099d0f53#reasoning-1)):

Game: A simplified Pac-Man-style gridworld: an agent moves through a fixed 8x8 maze with central and corner wall blocks, moving objects/ghosts, and reward targets/pellets.

Description: The agent navigates a fixed maze-like 8x8 board, using current agent/moving-object/previous-frame cues to move through corridors, collect reward-target cells, and avoid moving objects or hazards.

Confidence: 72%


This is impressive. I tried it on two of the other games as well. It [identified Pong](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F387d3a7927f165b1986bb6c2099d0f53#response-2) ([reasoning](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F387d3a7927f165b1986bb6c2099d0f53#reasoning-2)) in 1h 4m 54s:

**Pong** — specifically a Pong-like paddle/ball game where the agent controls a vertical paddle on the right side and must move up/down to intercept or return a moving ball/reward object.

Description:

The agent moves a right-side paddle vertically to hit or catch a moving ball-like reward target moving across the 8×8 field, while a left-side moving object/opponent and wall borders define the play; reward is obtained by successfully intercepting/returning/scoring the ball, and missing or failing to hit it is penalized.

Confidence:

**75%**


And [Breakout](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F387d3a7927f165b1986bb6c2099d0f53#response) ([reasoning](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F387d3a7927f165b1986bb6c2099d0f53#reasoning)) in 52m 25s:

Game: Breakout, or an Arkanoid-like paddle/ball/brick game.

Description: The agent is a paddle on the bottom row and moves left/right to keep a ball in play; the ball moves around the upper grid, and bricks/reward targets in the top rows give reward when hit, while missing the ball/hitting the bottom wall ends the episode or gives penalty.

Confidence: 88%


Qwen failed at the Freeway prompt, which didn’t surprise me considering that game is more obscure than the other three. Here’s [that transcript](https://tools.simonwillison.net/markdown-svg-renderer?url=https%3A%2F%2Fgist.github.com%2Fsimonw%2F6ed68f659e26a32e7d9096391d92f326)—Qwen thought it was “A simplified Pac-Man-like grid game: agent moves 4-directionally”.

The reasoning traces are fun to skim, and almost incomprehensible. Here’s a sample:

What about agent channel0? We see some large channel0 rows near row8 col7/8 and maybe bottom? Let’s list channel0 large: line436 input435 (row8 col7 ch0) -2.107,1.169,0.513...; line441 input440 (row8 col8 ch0) -0.376,0.010,1.564,3.855...; line445 input444 (row8 col8 ch4) 0.142,0.491...; line382? input381 ch0? Wait line382 input381: 381//5=76 (row7 col6), ch1? 381%5=1? 76 \* 5=380, ch1. So not. Channel0 large maybe at row8 col7/8 only. Also top row? line44 input43 ch3, line49 ch3. Channel0 large maybe not many. Agent position could be near bottom-right? If the agent is typically at bottom-right?


That said though, I expect a sufficiently experienced human being armed with a notepad and a pencil could get there with enough effort.

It’s a cool paper, but some of the wider commentary I’ve seen around it does not feel justified. I’m not ready to declare that the age of source code is over and our future involves incomprehensible machines editing their own model weights in incomprehensible ways.

## More recent articles

- [The U.S. Census Bureau and “Super Intelligence”](https://simonwillison.net/2026/Nov/2/census-bureau-super-intelligence/) - 2nd November 2026
- [The October inflection point for vibe game development](https://simonwillison.net/2026/Oct/28/vibe-game-development/) - 28th October 2026
- [Battle Drones - a vibe coded RTS with audio-based multiplayer](https://simonwillison.net/2026/Oct/20/battledrones/) - 20th October 2026
- [Hello Python 3.15, goodbye 3.10, and now I can use 3.11](https://simonwillison.net/2026/Oct/12/python-3-15/) - 12th October 2026
