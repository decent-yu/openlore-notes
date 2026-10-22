# 研究：sparqlite：用 Rust 编写，在 SQLite 内运行 SPARQL 1.1

> **来源**: [https://simonwillison.net/2026/Oct/16/sqlite-sparql-rust](https://simonwillison.net/2026/Oct/16/sqlite-sparql-rust)  
> **作者**: Simon Willison | **发布时间**: 2026-10-16T00:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: SQLite, Rust, RDF, SPARQL 1.1, Claude Code, 多智能体协作, 代码审查

---

## AI 导读

作者尝试让 Claude Code 用 Rust 编写 SQLite 扩展，使数据库能够直接查询和更新 RDF（用关系表达数据的图模型）数据，并支持 SPARQL 1.1（用于查询和更新这类图数据的语言）。按文中描述，开启 Ultracode 模式后，工具通过生成 JavaScript 来组织多个子代理协作，持续运行了超过12小时。作者随后要求补充各代码包的说明文件、安装与查询教程，以及用于填充示例数据的脚本，但正文没有展示这些要求的完成情况。开头的项目介绍声称，相比转换为 SQL 执行，计数和完整结果查询分别获得3.5倍、2.8倍加速，并通过639项适用测试中的636项；正文未提供测试条件和结果细节来支撑这些数字。作者最终关心的是：即使工具生成了看似完整的实现，如果自己无法投入时间审查代码、理解功能，其实际价值仍不明确。这是一份提出验收困境的实验记录，尚未给出解决办法，也不足以证明该扩展已可靠可用。

---

## 原文整理

2026年10月16日

[研究](https://simonwillison.net/elsewhere/research/)

[sparqlite：用 Rust 编写，在 SQLite 内运行 SPARQL 1.1](https://github.com/simonw/research/tree/main/sqlite-sparql-rust#readme)——sparqlite 是一个用 Rust 编写的 SQLite 可加载扩展。它将 RDF 四元组存储在普通的 SQLite 表中，并在 SQLite 进程内原生执行 SPARQL 1.1 查询和更新。它采用整数编码的项、覆盖索引、基于成本的规划器、批量连接，以及可选的 SQLite B 树直接读取器。与先转换为 SQL 再执行相比，它在计数模式下的速度达到 3.5 倍，在返回完整结果时达到 2.8 倍，同时通过了 639 项适用的 W3C 评估测试中的 636 项，以及数百万次差分模糊测试……

这项研究任务其实是一次实验：让网页版 Claude Code 在“Ultracode”模式下运行。Claude Code 文档中的[通过动态工作流大规模编排子代理](https://code.claude.com/docs/en/workflows)介绍了这种模式。

开启 Ultracode 后，Claude Code 会大量使用动态工作流：它会生成 JavaScript，用来定义和协调一个子代理网络，共同处理复杂问题。

我最初的提示词如下。我本来想写的是“直接访问数据行”，“delete”是我在手机上不小心打错的词：

> 用 Rust 编写一个实现 SPARQL 的 SQLite 扩展，要运行得非常快，因为它可以直接访问要删除的数据行。

这让网页版 Claude Code 持续忙活了超过 12 小时。第二天，我查看了进展，注意到它已经构建了很多 Rust crate，于是只追加了一次提示：

> crates/ 目录下的每个文件夹都需要一份 README。
>
> 提供一份 tutorial.md，教用户如何安装扩展，以及如何在 SQLite CLI 中运行它。除了介绍如何使用这个 SQLite 扩展，还要教他们 SPARQL。
>
> 随教程附上一个脚本，用有用、贴近实际的示例数据填充 SQLite 数据库，供教程使用。

这个项目感觉像是更进一步的氛围编程：审查这里生成的代码根本不现实，即使只是照着文档操作，也要投入大量精力。我不太确定该拿它怎么办。看起来，我现在已经有了一个用 Rust 编写、在 SQLite 内运行的完整 SPARQL 实现。但如果我只是一时兴起创建了它，又不能（或不愿）投入必要的时间去了解它能做什么，那它到底有多大价值？

## 近期文章

- [Battle Drones：通过氛围编程制作、基于音频实现多人联机的即时战略游戏](https://simonwillison.net/2026/Oct/20/battledrones/) - 2026年10月20日
- [你好，Python 3.15；再见，3.10；现在我可以用 3.11 了](https://simonwillison.net/2026/Oct/12/python-3-15/) - 2026年10月12日
- [用语音为我的博客开发了一个新功能](https://simonwillison.net/2026/Oct/9/built-using-my-voice/) - 2026年10月9日
- [Claude Haiku 5.5](https://simonwillison.net/2026/Oct/7/claude-haiku-5-5/) - 2026年10月7日