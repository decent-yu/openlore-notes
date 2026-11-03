# dbreunig/nesformer

> **来源**: [https://simonwillison.net/2026/Oct/30/nesformer](https://simonwillison.net/2026/Oct/30/nesformer)  
> **作者**: Simon Willison | **发布时间**: 2026-10-30T00:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: 6502, NES, CPU仿真, Qwen3.5-4B, 模型权重, 一致性测试

---

## AI 导读

文章介绍了一个实验：把任天堂娱乐系统（NES）所用6502处理器的仿真器编码进Qwen3.5-4B语言模型的权重。文中称它能够运行《超级马力欧兄弟》等游戏的ROM文件，即保存游戏程序和数据的文件，但运行速度很慢。作者展示了交给Opus模型的两轮提示词：先探索处理器与内存仿真的可行性和速度，再要求构建模型权重及可加载游戏文件的命令行工具。文章还称Opus似乎找到了并使用了已有的处理器一致性测试，用于检查指令执行行为是否符合预期。正文没有说明权重的构造方法、完整游戏运行所需的其他组件如何处理，也没有给出测试结果、硬件条件或实际运行速度，因此只能了解实验目标和所声称的成果，无法据此判断实现的正确性、可复现性或性能。

---

## 原文整理

2026年10月30日 \- 链接博客

**[dbreunig/nesformer](https://github.com/dbreunig/nesformer)** ([经由](https://news.ycombinator.com/item?id=50306442)\) 在 [qwenputer](https://divagr.com/blog/qwenputer) （作者：Divyansh Agrawal）的启发下，Drew Breunig 将一个可运行的任天堂娱乐系统 6502 CPU 仿真器写入了 Qwen3\.5\-4B 的权重中。现在，你可以在 Qwen 中运行 NES ROM（例如《超级马力欧兄弟》）来玩游戏了（速度非常慢）！

在 [Hacker News 的一条评论中](https://news.ycombinator.com/item?id=50306442#50307402) Drew 介绍了他用来构建这一系统的 Opus 5\.5 提示词：

`Read this article and any relevant source code: https://divagr.com/blog/qwenputer. Then explore if you could emulate the chip and ram in a Nintendo Entertainment System (as people do today with FPGAs in MISTRs) using this technique. If so, what size model would you have to build and if you could use smaller faster ones, could you get it to be at the standard clockspeed.`


随后是：

`I think we should build it! Just for fun. It's okay if it's not fast. We want to be able to have the weights, and a cli to run them that we can point at a ROM.`


据说，Opus 找到并随后使用了 [这组针对 NES CPU 的一致性测试](https://github.com/christopherpow/nes-test-roms/blob/master/other/nestest.txt) ；这些测试由 Christopher Pow 于 15 年前发布。

## 近期文章

- [vibe 游戏开发在 10 月迎来转折点](https://simonwillison.net/2026/Oct/28/vibe-game-development/) \- 2026年10月28日
- [Battle Drones——一款用 vibe 编程开发、支持基于音频多人游戏的 RTS](https://simonwillison.net/2026/Oct/20/battledrones/) \- 2026年10月20日
- [你好，Python 3\.15；再见，3\.10；现在我可以用 3\.11 了](https://simonwillison.net/2026/Oct/12/python-3-15/) \- 2026年10月12日
- [我的博客新增了一项用我的声音构建的功能](https://simonwillison.net/2026/Oct/9/built-using-my-voice/) \- 2026年10月9日
