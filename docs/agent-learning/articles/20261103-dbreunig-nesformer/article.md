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

30th October 2026 - Link Blog

**[dbreunig/nesformer](https://github.com/dbreunig/nesformer)** ([via](https://news.ycombinator.com/item?id=50306442)) Inspired by [qwenputer](https://divagr.com/blog/qwenputer) by Divyansh Agrawal, Drew Breunig baked a working emulator for a Nintendo Entertainment System 6502 CPU into the weights of Qwen3.5-4B. You can now play NES ROMs such as Super Mario Bros (very slowly) running inside Qwen!

In [a comment on Hacker News](https://news.ycombinator.com/item?id=50306442#50307402) Drew reveals the Opus 5.5 prompts he used to build this system:

`Read this article and any relevant source code: https://divagr.com/blog/qwenputer. Then explore if you could emulate the chip and ram in a Nintendo Entertainment System (as people do today with FPGAs in MISTRs) using this technique. If so, what size model would you have to build and if you could use smaller faster ones, could you get it to be at the standard clockspeed.`


Followed by:

`I think we should build it! Just for fun. It's okay if it's not fast. We want to be able to have the weights, and a cli to run them that we can point at a ROM.`


Apparently Opus found and then used [this collection of conformance tests](https://github.com/christopherpow/nes-test-roms/blob/master/other/nestest.txt) for the NES CPU released 15 years ago by Christopher Pow.

## Recent articles

- [The October inflection point for vibe game development](https://simonwillison.net/2026/Oct/28/vibe-game-development/) - 28th October 2026
- [Battle Drones - a vibe coded RTS with audio-based multiplayer](https://simonwillison.net/2026/Oct/20/battledrones/) - 20th October 2026
- [Hello Python 3.15, goodbye 3.10, and now I can use 3.11](https://simonwillison.net/2026/Oct/12/python-3-15/) - 12th October 2026
- [A new feature for my blog, built using my voice](https://simonwillison.net/2026/Oct/9/built-using-my-voice/) - 9th October 2026
