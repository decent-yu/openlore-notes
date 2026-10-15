# web-edit 0.1a0

> **来源**: [https://simonwillison.net/2026/Oct/12/web-edit/](https://simonwillison.net/2026/Oct/12/web-edit/)  
> **作者**: 未知 | **发布时间**: 2026-10-12T15:54:36+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: Agent, Engineering

---

## AI 导读

本文深入探讨了 web-edit 0.1a0 的架构设计与工程实践，重点分析了系统实现机制与核心权衡取舍。

---

## 原文整理

<p><strong>Release:</strong> <a href="https://github.com/simonw/web-edit/releases/tag/0.1a0">web-edit 0.1a0</a></p>
        <p>I was working on a project from my phone using Codex Remote when I found myself wanting to edit a credentials file without round-tripping those credentials through the LLM itself.</p>
<p>So... I had Codex and GPT-6 Astra Extra High build this Python web app which provides a web UI for viewing and editing files in a folder. It's intended to be run over Tailscale, so I can have an agent start this running and then edit files via my Tailscale network directly from my phone.</p>
<p>Here's the interface for browsing files in a folder:</p>
<p><img alt="Screenshot of a web file manager interface. Header reads &quot;WORKSPACE&quot; above the title &quot;Files&quot;, with buttons &quot;Refresh&quot;, &quot;Upload files&quot; and &quot;New file&quot; (dark green). A breadcrumb shows &quot;web-edit / web_edit / static&quot;. A table with columns &quot;Name&quot;, &quot;Size&quot;, &quot;Modified&quot; lists four files, each with &quot;Download ↓&quot; and &quot;Delete&quot; links: &quot;app.css&quot; 8.4 KB Oct 12, 2026; &quot;app.js&quot; 671.5 KB Oct 12, 2026; &quot;index.html&quot; 4.3 KB Oct 12, 2026 (underlined, as if hovered); &quot;THIRD_PARTY_LICENSES.txt&quot; 32.3 KB Oct 12, 2026. Below the table: &quot;Images under 2 MB appear inline. Select a text file to edit it.&quot; A panel below is labelled &quot;UPLOAD QUEUE&quot; with heading &quot;Add files&quot; and a &quot;Close&quot; button. It has a &quot;Destination folder&quot; text field containing &quot;web_edit&quot;, then a dashed drop zone reading &quot;Choose files or drop them here&quot; and &quot;Set a path and filename for each file before uploading.&quot; with a &quot;Choose Files&quot; button and the text &quot;no files selected&quot;. A greyed-out &quot;Upload files&quot; button sits at the bottom right." src="https://static.simonwillison.net/static/2026/web-edit.webp" /></p>
<p>Editing uses <a href="https://codemirror.net">CodeMirror 6</a> - I was going to use <a href="https://microsoft.github.io/monaco-editor/">Monaco</a> but that still doesn't have a great mobile experience yet.</p>
    
    
        <p>Tags: <a href="https://simonwillison.net/tags/projects">projects</a>, <a href="https://simonwillison.net/tags/python">python</a>, <a href="https://simonwillison.net/tags/ai">ai</a>, <a href="https://simonwillison.net/tags/tailscale">tailscale</a>, <a href="https://simonwillison.net/tags/generative-ai">generative-ai</a>, <a href="https://simonwillison.net/tags/llms">llms</a>, <a href="https://simonwillison.net/tags/codex">codex</a></p>
