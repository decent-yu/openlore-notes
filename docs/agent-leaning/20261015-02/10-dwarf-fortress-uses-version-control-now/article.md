# Dwarf Fortress uses version control now

> **来源**: [https://simonwillison.net/2026/Oct/11/dwarf-fortress/](https://simonwillison.net/2026/Oct/11/dwarf-fortress/)  
> **作者**: 未知 | **发布时间**: 2026-10-11T00:52:37+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: Agent, Engineering

---

## AI 导读

本文深入探讨了 Dwarf Fortress uses version control now 的架构设计与工程实践，重点分析了系统实现机制与核心权衡取舍。

---

## 原文整理

<p>I remember hearing a while ago that Dwarf Fortress didn't use version control. That fact has been lodged in my head, because given its inherent complexity I have trouble imagining a codebase that would benefit from version control <em>more</em>. That's art.</p>
<p>I went looking and I'm sad to report that the days without version control appear to have ended. Quoting Tarn Adams over time:</p>
<ul>
<li>March 2013: <a href="https://www.reddit.com/r/IAmA/comments/1avszc/comment/c919fo8/?solution=43270267461a564b43270267461a564b&amp;js_challenge=1&amp;jsc_token=2824be10929bdc604753c70a67a1c3310860c2cae0f946ff767fab3727807a35&amp;jsc_orig_r=">"I don't use version control -- I didn't like the feeling of having the code get committed into a black box thingy with no immediate upside"</a></li>
<li>March 2016: <a href="https://www.pcgamer.com/dwarf-fortress-creator-on-how-hes-42-towards-simulating-existence/3/">"I don’t even use version control. If you don’t know what that is then you’re not gonna yell at me. If you even know what version control is you’re gonna be like, ‘You don’t use version control? You don’t use source control? What is wrong with you? How can you even work? I’m just not cut out for it, in a sense. It’s been 15 years, so the reasons have change over time, but that’s my current understanding of it."</a> </li>
<li>March 2023: <a href="https://www.gamedeveloper.com/programming/how-tarn-adams-upgraded-and-optimized-dwarf-fortress-for-its-official-steam-release">"I've got Discord channels to check and version control stuff to use now, so the process is definitely a little more involved"</a></li>
</ul>
<p>That March 2023 quote comes from an interview about the release on Steam which <a href="https://www.pcgamer.com/after-spending-20-years-simulating-reality-the-dwarf-fortress-devs-have-to-get-used-to-a-new-one-being-millionaires/">made them millionaires</a>. I'm a tiny bit sad that they gave in before they hit that milestone, but what an impressive run.</p>

    <p>Tags: <a href="https://simonwillison.net/tags/version-control">version-control</a></p>
