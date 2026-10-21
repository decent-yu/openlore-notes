# 通过发送更多 CSS 提升网站性能

> **来源**: [https://github.blog/engineering/architecture-optimization/improving-site-performance-by-shipping-more-css](https://github.blog/engineering/architecture-optimization/improving-site-performance-by-shipping-more-css)  
> **作者**: Josh Black; Marie Lucca | **发布时间**: 2026-09-25T15:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: CSS Modules, CSS-in-JS, 设计系统, 渐进式迁移, 前端性能, 服务端渲染, 主题系统

---

## AI 导读

文章讨论 GitHub 在组件数量增长后，如何消除既有 CSS-in-JS 方案的客户端与服务端运行时成本，同时保持现有组件用法兼容。方案采用 CSS Modules，将样式放在组件旁的 CSS 文件中，默认局部化类名，并将样式汇总为随页面 HTML 发送的样式表。迁移先从 Primer 设计系统推进，通过特性开关和逐步发布验证；同时引入 @primer/styled-react 包装层，让依赖 sx 属性的调用方继续使用新组件，不使用 sx 的场景则直接导入 @primer/react。随后利用 VS Code 插件、内部 codemod、人工复核及后期的 Copilot coding agents 清理业务代码中的 sx，最后解耦主题相关 JavaScript 工具并移除依赖。文中报告，8 名轮值工程师在六个月内迁移 6,419 个属性，部分页面 SSR 耗时改善 1%—22%；后续两名工程师在三周内将剩余 895 个 sx 清零，并称于 2026 年 6 月完成全面迁移。主要经验是利用设计系统、兼容层和发布控制分阶段完成架构替换。局限在于当前材料缺失数处关键列表，未给出转换实现、动态样式处理细节、性能基线、测试环境或统计方法；自动化及 AI 工具的独立收益、跨主题验证范围和无破坏结论均无法据此充分核验。

---

## 原文整理

### [Josh Black](https://github.blog/author/joshblack/)

Josh Black 是一名软件工程师，居住在得克萨斯州奥斯汀。他喜欢开发设计系统、打造无障碍体验，也喜欢吃墨西哥烩玉米片。

我们如何让 github.com 彻底告别 CSS-in-JS。

如今，你在 GitHub 上看到的许多体验都由 [Primer 设计系统](https://primer.style/)提供支持。从按钮、横幅到面包屑导航，这些基础组件都需要在各种各样的场景下兼顾无障碍、灵活性和性能。

早在 2023 年，某些页面上的组件数量就开始激增。这让我们现有的 CSS-in-JS 方案面临了几个性能方面的难题：

显然，Primer 团队需要从源头解决问题。我们需要找到一种替代方案，彻底消除当前方案带来的客户端和服务端开销。最重要的是，无论选择哪种替代方案，它都必须能保证迁移期间 GitHub 的现有功能不受破坏。

Primer 团队找到了满足我们所有条件的方案：[CSS Modules](https://github.com/css-modules/css-modules)。这种格式让我们能做一件非常喜欢的事：编写并使用原生 CSS 特性，同时仍在一定程度上保留我们已经习惯从 CSS-in-JS 中获得的样式与组件就近放置的能力，以及封装能力。

使用 CSS Modules 时，样式会写在组件 JavaScript 源码旁的 CSS 文件中。它还允许我们默认将所有类名视为局部类名，避免全局选择器可能带来的一些冲突和问题。这种格式也不再需要任何客户端或服务端运行时处理。样式会汇总成 CSS 样式表，作为页面 HTML 的一部分发送出去。

不过，这一方案与我们当时采用的 CSS-in-JS 方案截然不同。这项变更意味着，每个 Primer 组件，以及 GitHub 中所有用这种方式编写的组件，都需要更新。好在设计系统非常适合大规模推进这类变更。

迁移到 CSS Modules 需要做什么，已经很清楚了。Primer 团队需要逐一更新组件，将它们从 CSS-in-JS 迁移到 CSS Modules。同时，对这些组件的更新不能破坏 GitHub 中任何现有用法。最后，我们用于实现 CSS-in-JS 的底层技术，也必须继续支持 GitHub 中仍在使用它的组件。

考虑到这些限制，我们决定采用渐进式迁移策略，确保能安全发布组件更新，不破坏现有功能。对于每个组件，我们的计划是：

这一流程形成了有效的反馈循环：随着 Primer 持续向 GitHub 交付这些变更，问题能在流程早期被发现。特性开关让我们可以安全地推进迁移，同时清楚地看到 CSS Modules 带来的性能收益。

到 2024 年 12 月，Primer 中的所有组件都已通过这一流程迁移到 CSS Modules。我们看到了各方面的性能提升，尤其是：

在 Primer 中开展这项工作带来了明确的性能收益，于是我们开始思考：如果在 GitHub 的其他部分也进行这样的转换，是否能获得类似的性能提升？同样，我们还要多久才能最终在整个公司范围内停止支持 CSS-in-JS？

从 Primer 中移除 CSS-in-JS 方案时，最棘手的问题之一来自 `sx` 属性的使用。这个属性一直是为 Primer 组件设置样式和进行定制的*主要方式*。团队可以传入一个内联对象，定制组件的方方面面。它体现了 CSS-in-JS 最好和最糟的部分：

`sx`
因此，我们告别 CSS-in-JS 的第一步，就是减少 GitHub 中 `sx` 的使用。这能让我们立即获得性能提升，类似于迁移 Primer 组件时取得的收益。同时，这也为我们从产品中彻底移除 CSS-in-JS 做好了准备。

需要说明的是，虽然设计系统本身已经正式不再依赖 `styled-components`，但 GitHub 代码库中还有很大一部分仍在使用它。多年来，`sx` 属性一直是 GitHub 事实上的样式编写标准。因此，要让 GitHub 用上不依赖 `styled-components`、更精简的新版 `@primer/react`，我们首先就得迁移数千个 `sx` 属性。

那么……我们是如何在增强把握、降低风险的同时，完成如此庞大的工作量的？答案是：分批推进。

最初的 CSS 迁移其实比我们前面说的更复杂一些：除了将组件迁移到 CSS Modules、通过特性开关在生产环境中测试并逐步发布之外，我们还在一个名为 [`@primer/styled-react`](https://github.com/primer/react/blob/main/packages/styled-react/README.md) 的过渡库中创建了“包装”组件。这个包的全部目的，就是让新迁移的组件仍然支持 `sx`。这样一来，GitHub UI 代码库中使用这个属性的地方，可以通过 `@primer/styled-react` 导入同一个组件，继续使用它；而不使用该属性的地方，则可以直接从 `@primer/react` 导入，获得性能提升。

迁移过程的下一阶段如下：

`@primer/styled-react` 导入与 `@primer/react` 导入
巧的是，就在我们准备开展这项大规模工作时，`styled-components` 宣布进入[维护模式](https://opencollective.com/styled-components/updates/thank-you)，这进一步印证了我们的方向是正确的。

这项工作于 2025 年 4 月启动，待迁移的 `sx` 属性在高峰时达到 \~7,760 个；直到 2026 年 5 月，我们才完成这项工作。最初，我们一位出色的内部开发者 [Ian Sanders，](https://github.com/iansan5653)开发了一个 [VS Code 插件](https://marketplace.visualstudio.com/items?itemName=ian-sanders.sx-to-css)，帮助逐个迁移属性。我们还在内部开发了一个类似的代码转换工具，用来迁移 GitHub 代码库中的整个文件。这项工作虽然需要一定的人工监督和仔细验证，但大部分都实现了自动化。8 名工程师轮流参与，在 6 个月内迁移了 6,419 个属性，并观察到部分页面的服务端渲染耗时改善了 1% 到 22%。

与此同时，在 GitHub 的另一边，Copilot 的能力正以指数级速度增长。AI 越来越聪明，能力也越来越强；这项工作仍在进行时，我们发布了 Copilot coding agent 和 Copilot code review。

等到我们重新接手这项工作时，时间已经来到 2026 年 4 月，情况也发生了变化。凭借两名工程师、坚持不懈的决心，以及大量 Copilot coding agents 的帮助，我们在三周内就把剩余的 `sx` 属性从 895 个降到了 0 个。

那是个值得纪念的日子：经过数年的努力，我们终于完成了所有 `sx` 迁移，扫清了彻底移除 `styled-components` 的障碍……现在终于可以清理这些依赖，转去做其他更令人兴奋的工作了，对吧？并没有！

GitHub 支持[七种不同的主题](https://docs.github.com/en/get-started/accessibility/managing-your-theme-settings)，每种主题都有高对比度模式。而这一切，没错，都是通过 `styled-components` 实现的。在考虑移除这些依赖之前，我们需要先让主题系统与它脱钩。

不过，这件事并没有听起来那么棘手。我们的主题变量一直通过 `@primer/css` 包在 CSS 中定义，而且在 2025 年底迁移 `@primer/react` 时，我们就已提前为不依赖 styled 的主题方案做好了准备。我们需要移除的是那些依赖 `styled-components` 的 JavaScript 用法和工具函数。于是，我们又开始了工作。

到这里，你已经熟悉这套流程了：执行迁移、逐步发布，所有变更都用特性开关控制。两个月过去，中间遇到了几次小波折，最终我们做好了移除依赖的全部准备；就连移除依赖这一步，我们也加了特性开关。谨慎一点总比事后后悔好。

自 2026 年 6 月起，GitHub 已完全使用 CSS Modules 运行。我们设置的保障措施，让我们能够安全地推出重大的架构变更，在生产环境中进行压力测试，发现错误，高效调整和修复，最终实现目标，并在此过程中取得了显著的性能提升。

这项工作起初看起来只是一次 CSS 迁移，最终却演变成了对 GitHub 大规模设置样式、应用主题和交付 UI 的方式进行的渐进式底层重建。到最后，我们不仅从 dotcom 中移除了 `sx`、`styled-components` 和 `styled-system`，还在整个过程中保证了 GitHub 的正常运行。提升产品性能、改善用户体验，让用户用得开心，始终是 GitHub 所有人关注的重点。

利用 GitHub 原生功能，审查外部协作者的访问权限，并归档不再活跃的仓库。

我们新推出的仓库星标历史 API 现在可以提供趋势数据，同时不返回个人身份信息，也不需要执行工作量没有上限的数据库操作。以下是我们所做的工程权衡。

我们正在保持 GitHub 正常运行的同时，重建其 Git 基础设施，为智能体规模的软件开发打下基础。
