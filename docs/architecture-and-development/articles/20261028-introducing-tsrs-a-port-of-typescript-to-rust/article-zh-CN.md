# tsrs 介绍：将 TypeScript 移植到 Rust

> **来源**: [https://blog.cloudflare.com/typescript-rust-port](https://blog.cloudflare.com/typescript-rust-port)  
> **作者**: 未知 | **发布时间**: 2026-10-20T13:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: TypeScript, Rust, WebAssembly, 类型检查, 编译器性能优化, 编译器兼容性

---

## AI 导读

TypeScript 项目检查代码需要时间和内存，tsrs 尝试通过将编译器移植到 Rust 来降低这些开销，同时严格跟随原版的语言行为。文章报告，在类型检查工作线程数相同的早期测试中，相比 TypeScript 7.0，tsrs 最高快 8 倍、内存占用约少 70%；它也指出，TypeScript 7.1 测试版本身已有最高 2.5 倍的提速。tsrs 默认还会按机器配置调整运行方式，以更多内存换取速度，因此这些结果不能脱离配置直接比较。
文章选择 Rust 的另一理由是便于编译为 WebAssembly，即可在浏览器等宿主环境中运行的程序格式。作者认为，Rust 的内存管理方式有利于程序与宿主交换内存，适合资源受限的开发环境，但正文未给出这些场景的实际测量结果。
项目已提供实验版本。维护方面，团队计划合并另一项移植工作，并沿用其他项目每天跟踪原版代码变化、运行兼容性测试和提出修复的流程，也希望将性能优化贡献回原版。当前仍有诊断信息输出顺序等差异，完整功能支持和稳定版兼容性属于后续目标。正文没有展开基准测试方法、具体优化实现或完整测试结果，因而目前更能说明项目方向，尚不足以确认不同项目中的普遍收益。

---

## 原文整理

# tsrs 介绍：将 TypeScript 移植到 Rust

![](assets/diagram-17659d4b7180.webp)

Cloudflare 和 VoidZero 正与 Owner.com 的 [Max Schwenk](https://building.owner.com/tsrs) 合作，开发 TypeScript 编译器的 Rust 移植版 [tsrs](https://github.com/oxc-project/tsrs)。我们在这项工作中与 TypeScript 团队密切协作，并将 tsrs 作为 [Oxc JavaScript 编译器工具链](https://github.com/oxc-project)的一部分进行孵化。

我们的目标是让开发者少花时间等待代码检查，多花时间构建应用。让 TypeScript 运行得更快、内存利用效率更高，也有助于让完整的开发环境在更多地方运行，包括直接在浏览器中运行。

需要明确的是，我们***并不是***要创建 TypeScript 的分叉版本。我们认为，TypeScript 团队应当继续全权掌握这门语言的设计和发展方向。他们在推动 TypeScript 在 JavaScript 社区中普及的同时，一直坚持着长远的愿景。我们不想改变这一点。

TypeScript 团队在很多事情上的推进速度都很快，但对于某些功能，他们可能会有意花上数年时间探索各种方案、制作原型，然后再选定合适的实现。Rust 移植版必须紧跟这些决策，否则最终就会难以为继，失去兼容性。我们必须严格以 TypeScript 的原始实现为准，保持这门语言的行为，而决定这些行为的责任应当始终完全由 TypeScript 团队承担。

## 性能和内存方面的改进

TypeScript Rust 移植版的早期基准测试结果令人鼓舞：在类型检查工作线程数相同的情况下，与 TypeScript 7.0 相比，`tsrs` 的速度最高可达其 8 倍，内存占用约减少 70%：

![](assets/diagram-18fab894d351.webp)

可以看到，TypeScript 的下一个版本本身就已经带来了许多性能改进，使 TypeScript 7.1 beta 的速度最高可达 TypeScript 7.0 的 2.5 倍。其中一些优化来自 `tsrs`，已经贡献给上游；我们希望还能找到更多这样的优化。

你可能看过近期针对不同 TypeScript 移植版的基准测试，以及各式各样的性能宣称。这些结果往往来自使用规格更高的机器，以及与 TypeScript 默认设置不同的选项。我们与 TypeScript 团队合作，确保[上述基准测试](https://github.com/oxc-project/tsrs/tree/main/bench)是在相同配置下进行比较的。

默认情况下，tsrs 会根据机器规格自适应地调整配置，占用更多内存，但速度也会更快：

![](assets/diagram-923389c0a73b.webp)

## 试用 `tsrs`

TypeScript 的 Rust 版本旨在忠实移植 TypeScript。你可以从 npm 安装 [`tsrs`](https://npmx.dev/package/tsrs) 包，试用实验版本：

```
npm install tsrs
npx tsrs
```

## 为什么是现在？

多年来，我们见过许多将 TypeScript 移植到其他语言的尝试。由于这项工作十分复杂，只有一个移植项目成功了：TypeScript 团队将 TypeScript 从 JavaScript 移植到了 Go。

不过，如今最前沿的 AI 模型几乎可以独立完成大规模移植，只需几天时间。我们已经见到了用 Rust 或 C++ 编写的各种 TypeScript 版本：有粗制滥造的分叉版本，有忠实的移植版，也有架构经过彻底重构的版本。显然，大家需要速度更快的 TypeScript。

## 为什么选择 Rust？

你可能会想：“TypeScript 的 Go 移植版已经让 TypeScript 的速度达到此前用 TypeScript 编写的版本的 10 倍，内存利用效率也高了很多。这还不够吗？”

不过，我们考虑的不只是速度。Rust 还具有一些独特的优势：[在 WebAssembly 中运行](https://www.npmjs.com/package/@tsrs/wasm)。Rust 与 Wasm 的结合一直很出色，其中一个原因就源于 Rust 语言设计的基本机制。将 Go 程序编译为 Wasm 时，必须把 Go 的垃圾回收器也一起编译进去。而 Rust 的内存管理模型让 WebAssembly 客体程序与宿主之间的内存交互变得直接。这正是 Rust 更适合 WebAssembly 使用场景的关键原因之一：内存利用效率高。

Cloudflare Workers 支持 Wasm 已经很久了。就在几周前，我们宣布借助 wasm-bindgen 新增的 Emscripten 目标，[在 Workers 中支持原生 Rust](https://blog.cloudflare.com/rust-workers-emscripten-target/)。当你在 Workers 上高频循环运行整套软件构建和部署系统时，较低内存占用的优势很快就会显现。这意味着，这些工作负载可以运行得非常高效，让封装完整的开发系统如今能够在过去无法运行它们的地方运行。

与智能体结合后，这又带来了新的可能：现在，你可以在 Cloudflare 上的完全沙箱化环境中，运行从头到尾的整套开发工具链。而借助 Wasm，你还可以在浏览器客户端、嵌入式设备或物联网应用中安全地运行这些环境。

## `tsrs` 移植版将如何维护？

首先，我们正在与 Theo Browne 合作，将 [tsc-rs](https://github.com/pingdotgg/ts-rust) 和 `tsrs` 合并为一个包，支持 TypeScript 的全部功能，包括语言服务器协议（LSP）和内容映射器。其次，我们在让 [Vinext](https://github.com/cloudflare/vinext) 与其上游 Next.js 保持同步方面，已经取得了很好的成效。我们在[最近的一篇博文](https://blog.cloudflare.com/vinext-nextjs-on-vite)中介绍了具体做法：

每天早上，一个智能体会审查变更、获取差异，并为任何可能影响 Vinext 的内容创建跟踪问题。每天晚上，我们会在 Vinext 上运行 Next.js 测试套件，并重新生成兼容性矩阵。当某项测试或某个问题暴露出差距时，如今的智能体已经能够对照两个代码库找出相关变更，构建复现示例，移植相关测试，并提出修复方案。

我们会将同样的流程应用于 TypeScript，并格外谨慎，避免引入与上游 TypeScript 不一致的行为。当我们发现性能改进时，会尝试将其贡献回 TypeScript 的 Go 代码库，让所有人都能受益。

最后，我们也清楚，目前这个版本与上游 TypeScript 仍有一些差异，例如诊断信息的输出顺序。我们的目标是在发布稳定版时，将这些差异降到最低。

## 接下来呢？

Cloudflare 已经汇聚了许多为 JavaScript 生态提供支持的自由开源 Rust 工具，例如 Vite、Rolldown、Oxlint 和 Oxfmt。超过 1,000 万名开发者在 Cloudflare 上构建应用，因此，我们非常重视让底层开发工具链更快、更易于移植，也更容易在更多地方运行。我们很期待尝试将 `tsrs` 更深入地集成到我们的技术栈中，例如用于 Oxlint 中能利用类型信息的代码检查规则。

与 Vinext 一样，我们的目标是将 `tsrs` 运营为一个维护良好、健康发展的开源项目，同时不妨碍 TypeScript 自身的发展。Oxc 团队已经开始重构代码库，以提高速度、降低内存占用，并让代码更易于维护。我们知道，TypeScript 团队也在认真对待这件事——他们正在尝试改进 `tsc` 的内存占用和性能。我们很期待与 TypeScript 团队和 JavaScript 社区合作，共同打造最好、最快的 TypeScript 实现。
