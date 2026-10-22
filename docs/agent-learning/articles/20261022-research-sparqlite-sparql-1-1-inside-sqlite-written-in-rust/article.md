# Research: sparqlite: SPARQL 1.1 inside SQLite, written in Rust

> **来源**: [https://simonwillison.net/2026/Oct/16/sqlite-sparql-rust](https://simonwillison.net/2026/Oct/16/sqlite-sparql-rust)  
> **作者**: Simon Willison | **发布时间**: 2026-10-16T00:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: SQLite, Rust, RDF, SPARQL 1.1, Claude Code, 多智能体协作, 代码审查

---

## AI 导读

作者尝试让 Claude Code 用 Rust 编写 SQLite 扩展，使数据库能够直接查询和更新 RDF（用关系表达数据的图模型）数据，并支持 SPARQL 1.1（用于查询和更新这类图数据的语言）。按文中描述，开启 Ultracode 模式后，工具通过生成 JavaScript 来组织多个子代理协作，持续运行了超过12小时。作者随后要求补充各代码包的说明文件、安装与查询教程，以及用于填充示例数据的脚本，但正文没有展示这些要求的完成情况。开头的项目介绍声称，相比转换为 SQL 执行，计数和完整结果查询分别获得3.5倍、2.8倍加速，并通过639项适用测试中的636项；正文未提供测试条件和结果细节来支撑这些数字。作者最终关心的是：即使工具生成了看似完整的实现，如果自己无法投入时间审查代码、理解功能，其实际价值仍不明确。这是一份提出验收困境的实验记录，尚未给出解决办法，也不足以证明该扩展已可靠可用。

---

## 原文整理

16th October 2026

[Research](https://simonwillison.net/elsewhere/research/)

[sparqlite: SPARQL 1.1 inside SQLite, written in Rust](https://github.com/simonw/research/tree/main/sqlite-sparql-rust#readme)— sparqlite is a Rust-based SQLite loadable extension that stores RDF quads in ordinary SQLite tables and executes SPARQL 1.1 queries and updates natively inside the SQLite process. Its integer-encoded terms, covering indexes, cost-based planner, batched joins, and optional direct SQLite B-tree reader deliver 3.5× speedups over SQL translation in count mode and 2.8× when returning full results, while passing 636 of 639 applicable W3C evaluation tests and millions of differential-fuzz…

This research task is really an experiment in running Claude Code for web in "Ultracode" mode, as described in [Orchestrate subagents at scale with dynamic workflows](https://code.claude.com/docs/en/workflows) in the Claude Code documentation.

With Ultracode turned on, Claude Code makes aggressive use of dynamic workflows - it generates JavaScript that defines and coordinates a network of subagents to take on complex problems.

My initial prompt was this - I meant "hit the rows directly", the word "delete" was an accidental typo on my phone:

`Write a SQLite extension in Rust that implants SPARQL and runs really fast because it can hit the delete rows directly`


This caused Claude Code for web to churn for over 12 hours. I checked in on its progress the next day and sent a single follow-up prompt when I noticed how many Rust crates it had built:

`Every folder in the crates/ directory needs a README`

`Include a tutorial.md teaching users how to install the extension and how to run it in the SQLite CLI and then teaching them SPARQL in addition to how to use the SQLite extension`

`Include a script with the tutorial for populating a SQLite database with useful and realistic example data for that tutorial`


This project feels like next-level vibe coding - it's not remotely feasible to review the code that was produced here, and even following the documentation is a significant investment in work. I'm not entirely sure what to do with this - I appear to have a full SPARQL implementation in Rust inside SQLite now, but how valuable is that if I created it on a whim and can't (or won't) invest the necessary time to understand what it can do?

## Recent articles

- [Battle Drones - a vibe coded RTS with audio-based multiplayer](https://simonwillison.net/2026/Oct/20/battledrones/) - 20th October 2026
- [Hello Python 3.15, goodbye 3.10, and now I can use 3.11](https://simonwillison.net/2026/Oct/12/python-3-15/) - 12th October 2026
- [A new feature for my blog, built using my voice](https://simonwillison.net/2026/Oct/9/built-using-my-voice/) - 9th October 2026
- [Claude Haiku 5.5](https://simonwillison.net/2026/Oct/7/claude-haiku-5-5/) - 7th October 2026
