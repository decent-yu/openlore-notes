# Rendering huge pull requests in the GitHub Copilot app

> **来源**: [https://github.blog/engineering/user-experience/rendering-huge-pull-requests-in-the-github-copilot-app](https://github.blog/engineering/user-experience/rendering-huge-pull-requests-in-the-github-copilot-app)  
> **作者**: Alberto Gimeno | **发布时间**: 2026-09-23T18:29:49+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: 前端性能优化, 列表虚拟化, 动态布局, 滚动锚定, 代码审阅, 缓存策略, 自动化性能测试, 可观测性

---

## AI 导读

文章解决超大 pull request 中代码与评论混排的渲染问题。代码行高度可预先计算，但评论高度受 Markdown 换行、折叠区、回复框和图片加载影响；统一估高再回写布局容易造成留白、裁切和滚动跳动。主要方法是将确定性的代码高度与动态块高度分开索引，动态块通过稳定标识、文件与行侧锚点、内容指纹和宽度分桶复用测量，优先采用有效实测值，其次使用匹配缓存，最后使用估计值。测量集中在受空闲与滚动状态控制的流程中，观察器主要负责标记待测，避免直接写回引发反馈循环；滚动修正还需区分用户操作与界面自身产生的滚动。数据侧采用增量加载、后台语法高亮、临近视口时构建复杂内容，以及有界的近期 diff 缓存。验证侧使用结构化探针、合成大规模夹具、无头流程和桌面自动巡检，检查空白、缺失评论及深层滚动区域。作者称百万行、数百评论的差异视图已可正常打开、滚动并恢复位置。但正文未给出量化性能对比、预算阈值或完整复现材料，且若干规则列表和执行循环缺失，结论主要由工程叙述及自述验证支撑。

---

## 原文整理

### [Alberto Gimeno](https://github.blog/author/gimenete/)

Principal Design Engineer

How we rebuilt the diff surface in the GitHub Copilot app to open a million-line pull request with hundreds of inline review comments.

Broad refactors and migrations often have to land as one change.

[Stacked pull requests](https://docs.github.com/pull-requests/how-tos/stacked-pull-requests) are a great way to split work into smaller changes, which makes reviews easier and helps teams ship with less risk. But some changes, like this one, can’t be split cleanly. That leaves you with a single pull request that can get very large, and the review conversation causes it to grow.

The review experience needs to remain fast and smooth even when the diff and its conversation are enormous. In the [GitHub Copilot app](https://github.com/features/ai/github-app), we rebuilt the pull request view with that requirement in mind.

To see how far that goes, we opened the biggest pull request we could find: an open source one with 2,200 files, over a million changed lines, and more than 400 inline review comments. Here’s how we made even this extreme pull request performant.

Rendering a large diff at speed is well-understood: virtualize the rows, keep the mounted DOM small, and lean on the fact that every row is a line of code at a known height.

Comments are the hard part. A comment’s height depends on how its markdown wraps, the expandable sections, whether there’s a reply box in it, and whether its images have loaded yet. You find all of that out at render time. This forces a different architecture.

Three problems:

The first step is to understand the geometry that makes a code-only diff fast. Once comments enter the picture, that geometry is no longer enough.

You cannot put a million DOM nodes on a page. The standard answer is **virtualization**: mount only the rows that are on screen, plus a small margin, and recycle those same DOM elements as the user scrolls. The list behaves as if all million rows exist. The scrollbar is the right size, scroll-to-row works. But only about 100 rows are ever real at once.

For this illusion to hold, something has to supply the geometry. The scrollbar height is the sum of all row heights. The position of row *N* is the sum of the heights of the rows above it. Jumping to a row, drawing the scrollbar, deciding what’s on screen, it’s all arithmetic over a table of heights. You can build that table from estimates and correct it as rows get measured, and general-purpose variable-height virtualizers do exactly that.

But if every row is a line of code at a known font size, you don’t have to. You can compute the whole table up front and it never changes, so there’s nothing to correct later.

Call this the “all heights known before paint” contract. Our diff surface is built around it:

None of it scales badly, because no per-frame work grows with the total row count. On pure code this design is the right one, and we kept all of it.

Now put a review thread in the middle of the diff. How tall is it?

You don’t know, and you can’t know without rendering it. Its height depends on things that only exist at render time, and they can keep changing after first paint:

`<details>` blocks the user can expand or collapse in place
The obvious answer is to reserve a fixed-height slot for each comment, sized by an estimator. It falls apart on a big pull request. An estimator that’s right on average is still wrong at the extremes. It over-reserves most comments, leaving gaps of whitespace, and under-reserves the expensive ones, which clip or sprout a nested scrollbar. If you measure the real height after paint and write it back into the shared offset table, everything below moves, **while the user is already scrolling.** That’s a scroll jump, and on a big pull request it’s a large one.

So comments need a different contract. “All heights known before paint” is unachievable for this content. What we could promise instead: heights are bounded, measured lazily, and corrections are small and anchored to whatever the user is looking at.

The idea that made this tractable was to stop forcing one geometry to serve both kinds of content. We split the document’s height into two independent domains:

```
total height = deterministic code height          (exact, known up front)
             + Σ dynamic block effective heights   (estimated, then measured)
             + scroll padding
```
**Code geometry** keeps the original world. It’s deterministic, prefix-summed, exact, never rebuilt when a comment resizes.

**Dynamic block geometry** covers everything whose height we can’t predict, such as review threads, drafts, and reply composers. Each one is a block identified by what it is rather than where it currently sits. It has a stable key that survives its content loading, and it’s anchored to a file, line and side rather than to a pixel coordinate, so a reflow can’t lose track of it. We also keep a fingerprint of everything that could change the block’s height: its content, whether a `<details>` is open, whether a composer is active. And we record the width it was last measured at, rounded into buckets, so an ordinary window resize doesn’t invalidate every measurement in the document.

A block’s effective height is then simple: the measured height if we have a valid one, a cached height if the fingerprint and width still match, and the estimate otherwise. Those heights live in their own index, separate from the code rows, so a resizing comment never forces the code geometry to be rebuilt. And the number of blocks is bounded by comments, not by rows. A few thousand blocks is fine, as long as first paint never mounts or measures all of them at once.

This part took the longest to get right, because our first design was wrong in an instructive way.

The obvious way to measure dynamic content is one `ResizeObserver` per block, which watches the element and writes its measured height back into the layout whenever it changes. This is what we designed and then rejected during performance hardening. It is the feedback loop that big virtualized surfaces have to avoid. An observer that writes a height back into the layout of the element it’s watching can retrigger itself, and the cost grows with every mounted block.

What shipped instead is a single idle- and scroll-gated measurement pass, held to the same discipline as the deterministic side:

`ResizeObserver`, but by default all it does is flag the block so the idle pass re-reads it. It never writes a height itself, which is what would close the feedback loop we rejected. It disconnects on unmount, and an inactive pull request tab observes nothing.
When a measured height differs from its estimate, the scrollbar arithmetic changes, and the naive result is that the viewport jumps. The fix is to correct by identity rather than by pixel:

Plus a few rules that keep it from feeling wrong:

That last rule has a sharp edge, and it bit us. “Don’t correct while the user is scrolling” was implemented as a guard on the last observed scroll, and programmatic scrolls refreshed that timestamp too. Toggling the file-tree sidebar changes the width of the diff pane. With line wrapping on, every wrapped line above you reflows to a different number of visual lines, the whole coordinate space shifts, and the surface emits a small scroll of its own as it settles. The guard read that as “the user just scrolled” and skipped the very correction that was supposed to keep your place, so the file you were reading drifted off screen. The fix was to tell user scrolls apart from ones the surface caused itself. Any “is the user interacting?” check has to be one your own side effects can’t satisfy.

So corrections stay small, they reuse measurements we already have, and they follow whatever you’re looking at.

A diff surface can only be as fast as the data feeding it, and three habits from that side of the work shaped what the UI could do. The first is stream structure before content. The diff is requested incrementally, so the file tree and metadata paint while the document is still loading, and the full set of review threads is resolved up front rather than trickling in. The second is defer per-item work until something needs it. Syntax highlighting runs off the main thread, so rows appear as plain text immediately and get colored when the results arrive. Highlighting improves the surface instead of blocking the scroll. Large markdown bodies and suggested-change context work the same way: nothing is built until it approaches the viewport.

The third habit is about which costs are worth keeping. Releasing a diff document when you navigate away is the right default. These documents are large, and holding on to every one you’ve visited is how a long session ends up eating memory. But pull request metadata persists, so the shell around the diff, the header and the file tree, repaints instantly when you go back, and then sits there for several seconds waiting for a diff it had complete moments ago. An instantly-drawn shell around an empty diff looks broken, even though you’re waiting less time overall. So the policy stayed and we added a cache: keep the last few diffs resident, evict anything beyond that, and let the background refresh notice when one has gone stale.

Almost every bug in this project was invisible until it wasn’t, and reproducing one by hand is miserable. A typical report reads: “a strip of whitespace appears below some comments, but only sometimes, only on big pull requests, and it heals if you scroll past and back.” You can’t debug that by staring at the screen, so we built tooling to debug it mechanically.

The naive workflow is to sprinkle `console.log` calls, exercise the flow by hand, copy the output, paste it to someone (or something) that can analyze it, delete the logs, and repeat. It’s slow, it needs a human in the loop, and worst of all you end up measuring your own hand-rolled instrumentation rather than the app’s real behavior.

So the surface carries permanent, structured probes for its own invariants. They’re plain questions it answers about itself on every render:

These are the objective pass/fail signals, and they’re asserted as budgets in an end-to-end test against a synthetic many-comment huge-pull-request fixture. CI can now tell us whether the surface is healthy.

The centerpiece was an autonomous **change → measure → improve** loop. Two lanes:

**A headless probe lane** ran a declarative flow (open a pull request, scroll to a fraction, toggle a details block, resize the window) against a mock server, reading the app’s own production instrumentation: React render counts, the performance timeline, and a `requestAnimationFrame` sampler for jank. It did the whole instrument, drive, collect, analyze, rank cycle by itself and printed the bottlenecks in order. Because the flow is just JSON handed to the probe at runtime, an agent could profile any flow by describing it in plain English, without editing a line of source.

**An autopilot** drove the actual desktop app through the huge-pull-request flow, unattended, on a loop: first cold, with comments still skeletons, then warm, with comments loaded, toggling `<details>` blocks, opening and cancelling reply composers, collapsing and expanding files, toggling the sidebar tree, sweeping deep into the file list, resizing the window. Every measurement was mirrored to the app’s on-disk log, so an agent could read runtime behavior with nobody at the keyboard. Each sample carried a health signal, and that was the objective check. A warm sample counted as healthy only if there were no unfilled gaps between comments, no comment blocks left blank, and real thread content actually mounted, across the entire scroll range, deep-file sweep included.

The loop we ran was:

Reviewing a pull request this large used to mean one of two things: waiting, or giving up and reading it somewhere else. A review isn’t a document with known dimensions. It’s a conversation that changes shape while you’re reading it, and the surface underneath has to be built for that from the start rather than patched into it afterwards.

The result is a pull request view where a million-line diff with hundreds of threaded comments opens, scrolls, and behaves like a normally sized pull request. Comments render in full instead of clipping into a scrollable box. Expanding a collapsed section moves the code below it and nothing else. Coming back to a pull request you just left puts you where you were.

If you review code for a living, it’s worth feeling the difference on a pull request you already know is painful. Open the worst one you’ve got.

Learn how to move an agent session from the GitHub Copilot app into VS Code without losing the conversation, code changes, or working context.

Put native GitHub features to work to review outside collaborator access and archive stale repositories.

Our new repo star history API now provides trends without returning individual identities or requiring unbounded database work. These are the engineering tradeoffs we made.
