# Deno is joining Cloudflare

> **来源**: [https://simonwillison.net/2026/Oct/9/deno-is-joining-cloudflare/](https://simonwillison.net/2026/Oct/9/deno-is-joining-cloudflare/)  
> **作者**: 未知 | **发布时间**: 2026-10-09T22:48:37+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: Agent, Engineering

---

## AI 导读

本文深入探讨了 Deno is joining Cloudflare 的架构设计与工程实践，重点分析了系统实现机制与核心权衡取舍。

---

## 原文整理

<p><strong><a href="https://deno.com/blog/cloudflare">Deno is joining Cloudflare</a></strong></p>
The Deno team released the first version of <a href="https://github.com/denoland/celld">celld</a> back in August - their open source implementation of the Durable Objects pattern from Cloudflare Workers.</p>
<p>Today, Cloudflare are acquiring Deno outright, with the goal of building on <code>celld</code> to "make workerd self-hosting a first-class supported way to build and run apps using the Workers programming model" (see <a href="https://blog.cloudflare.com/deno-joins-cloudflare/">the Cloudflare blog</a>.)</p>
<p>The bad news is that Deno itself will not be maintained by Cloudflare beyond the next year:</p>
<blockquote>
<p>We will support the <a href="https://github.com/denoland/deno">Deno runtime</a> for another year with monthly releases containing bug fixes and security updates. After that year we will end our development of the Deno runtime. Deno will remain open source, and we welcome others who want to continue its development.</p>
</blockquote>
<p>Deno (and Node.js) creator Ryan Dahl explained that decision in <a href="https://news.ycombinator.com/item?id=50019911#50023277">a comment</a> on Hacker News:</p>
<blockquote>
<p>It's a joint decision and I agree with it. I'm most invested in its success and have put the most work into it - and I no longer think it's where I can do the most important work. There are some good ideas in Deno and it's well engineered - but it ultimately is not solving big problems. It has been sucked into the gravity well of node compatibility, which forces it to behave exactly as Node does. Why reimplement Node? It works. Marginal performance or UX or security benefits are not enough.</p>
<p>I'm interested in building powerful new abstractions. celld has been working remarkably well, depending only on object storage for coordination and persistence. It is not just a slightly different API to interact with the file system or network - it's an entirely new model for server development.</p>
</blockquote>
<p>My favorite feature of Deno has long been the permissions system, where you can run a Deno script and specify exactly which files and folders it can read and write to, and which network hosts it can access.</p>
<p>Node.js <a href="https://nodejs.org/api/permissions.html">has a similar permissions model</a> these days, added <a href="https://nodejs.org/en/blog/release/v20.0.0">in Node v20.0.0</a> in April 2023 and declared stable <a href="https://nodejs.org/en/blog/release/v22.13.0">in Node v22.13.0</a> in January 2025. They don't yet support allow-listing specific network hosts though - networking is either on or off.

    <p><small></small>Via <a href="https://news.ycombinator.com/item?id=50019911">Hacker News</a></small></p>


    <p>Tags: <a href="https://simonwillison.net/tags/cloudfront">cloudfront</a>, <a href="https://simonwillison.net/tags/javascript">javascript</a>, <a href="https://simonwillison.net/tags/nodejs">nodejs</a>, <a href="https://simonwillison.net/tags/ryan-dahl">ryan-dahl</a>, <a href="https://simonwillison.net/tags/sandboxing">sandboxing</a>, <a href="https://simonwillison.net/tags/deno">deno</a></p>
