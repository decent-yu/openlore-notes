# Improving site performance by shipping more CSS

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

Josh Black is a Software Engineer based in Austin, Texas. He loves working on Design Systems, creating accessible experiences, and eating chilaquiles.

How we fully migrated github.com away from CSS-in-JS.

The [Primer Design System](https://primer.style/) powers many of the experiences you see on GitHub today. From buttons to banners to breadcrumbs, these foundational components are required to be accessible, flexible, and performant across a wide variety of scenarios.

Back in 2023, the number of components on certain pages began to explode. This led to several performance-related challenges with our existing CSS-in-JS solution:

It became clear that the Primer team needed to address the issue at the source. We needed to find an alternative that would completely avoid the client and server costs that we were seeing with our current solution. Most importantly, any alternative we pick would need to work in a way that would avoid any breakage to GitHub during the migration.

The Primer team found a solution that met all of our criteria: [CSS Modules](https://github.com/css-modules/css-modules). This format would allow us to do one of our favorite things: write and use native CSS features, while still allowing some amount of the colocation and encapsulation that we had come to expect from CSS-in-JS.

With CSS Modules, styles would be authored in a CSS file alongside the JavaScript source for the component. It would also allow us to treat all class names as local by default, preventing some of the collisions and challenges that can come from global selectors. This format also removes the need for any client or server runtime behavior. Instead, styles would roll up into CSS stylesheets that were sent as part of the HTML for a page.

However, this solution was radically different from the CSS-in-JS solution we had at the time. This change would require an update to every Primer component and every component at GitHub authored using this technique. Thankfully, design systems are a perfect vehicle to deliver this kind of change at scale.

The situation for moving towards CSS Modules was clear. The Primer team would need to deliver updates to each of its components, moving them from CSS-in-JS to CSS Modules. At the same time, updates we made to these components could not break any usage in GitHub. Finally, the underlying technique we used for CSS-in-JS also had to continue working for any components in GitHub that were currently using it.

With all these constraints in place, we decided on an incremental migration strategy that would allow us to safely ship component updates without breaking the world. For each component, our plan was to:

This process created a strong feedback loop where issues were flagged early in the process as Primer continuously delivered these changes to GitHub. The use of feature flags allowed us to do this migration safely while giving us clear signals on the performance benefits of CSS Modules.

By December 2024, all components in Primer were migrated over to CSS Modules using this process. We saw performance wins across the board, in particular:

With clear performance wins from doing this work in Primer, we began to wonder if we could see similar performance wins by doing these conversions in other parts of GitHub. Similarly, how long until we could ultimately drop support for CSS-in-JS across the company?

One of the trickiest parts about removing our CSS-in-JS solution from Primer was due to the usage of the `sx` prop. This prop was *the* way to style and customize components from Primer. Teams could provide an inline object to customize everything about the component. It represented the best and worst parts of CSS-in-JS:

`sx`
As a result, the first part of our journey to move away from CSS-in-JS was to reduce `sx` usage across GitHub. This would allow us to immediately improve performance similar to the wins we saw when migrating Primer components. It also set us up perfectly for removing CSS-in-JS entirely from the product.

It’s important to note that, while the design system itself was officially off `styled-components`, a large part of the GitHub codebase itself wasn’t. With `sx` props having been the de facto styling standard at GitHub for years, we were looking at thousands of `sx` props that needed migration before we could even think of getting GitHub onto the sleek new `@primer/react` version which didn’t rely on `styled-components`.

So… how did we do this immense amount of work while increasing confidence and reducing risk? The answer: not all at once.

The original CSS migration was a little bit more nuanced than we led on: in addition to migrating the components to CSS modules, feature flagging to test in production and slowly rolling them out, we also created “wrapper” components in a transitive library we called [`@primer/styled-react`](https://github.com/primer/react/blob/main/packages/styled-react/README.md). The whole purpose of this package was to allow `sx` usage into the newly migrated components. This way, the instances of the GitHub UI codebase that were using this prop could continue to consume them by importing the same component through `@primer/styled-react`, while we realized the performance gains of importing straight from `@primer/react` for the cases that didn’t.

The next phase of the migration process was as follows:

`@primer/styled-react` imports with `@primer/react` imports
Curiously enough, while we were getting ready to undertake this massive effort, `styled-components` [maintenance mode](https://opencollective.com/styled-components/updates/thank-you) was announced, offering further confirmation that we were taking steps in the right direction.

The work kicked off April 2025 with a peak of \~7,760 `sx`props to be migrated; we wouldn’t see it realized until May 2026. Initially, one of our great in-house developers, [Ian Sanders,](https://github.com/iansan5653) created a [VS Code plugin](https://marketplace.visualstudio.com/items?itemName=ian-sanders.sx-to-css) that would assist with per-prop migration. A similar codemod was developed internally and utilized to migrate entire files in the GitHub codebase. The work, while requiring a bit of manual oversight and careful validation, was mostly automated. A rotation of 8 engineers migrated 6,419 props over the course of 6 months, observing Server-Side Rendering time performance gains ranging from 1% up to 22% in some pages.

In a different side of GitHub, Copilot’s capabilities were increasing exponentially. AI was getting smarter, more capable; we released Copilot coding agent and Copilot code review while this work was still underway.

By the time we picked this work back up, now April of 2026, the panorama was different; we were able to get down from 895 to 0 `sx`props in the span of three weeks with a team of two engineers, relentless determination, and a whole lot of Copilot coding agents.

It was a big day: we had finally completed the `sx` migrations that were standing between us and full `styled-components` removal, years in the making… we can finally clean up these dependencies and move on to different, more exciting work, right? Wrong!

GitHub supports [seven different themes](https://docs.github.com/en/get-started/accessibility/managing-your-theme-settings), all of which offer a high contrast mode variation. All of it enabled through, you guessed it, `styled-components`. Before we can even think of removing these dependencies, we need to decouple our theming.

Now, this isn’t as huge a deal as it sounds. Our theming variables have always been defined in CSS through our `@primer/css` package, and we already planned forward for non-styled Theming when we migrated `@primer/react` in late 2025. It’s the JavaScript usage and utilities that are enabled by `styled-components` that we needed to remove. Once again, we got to work.

You know the drill by now: perform the migrations, roll it out slowly, feature flag everything. Two months and a few hiccups along the way later, we were all-systems go for dependency removal; we even feature flagged that. Better safe than sorry.

GitHub has been running on 100% CSS modules as of June 2026. The safeguards we put in place enabled us to roll out significant architectural changes safely, stress test in production, catch errors, pivot and repair efficiently, ultimately allowing us to succeed in our goals and realize great performance gains along the way.

What looked at first like a CSS migration turned out to be a gradual re-platforming of how GitHub styles, themes, and ships UI at scale. By the end, we had not only removed `sx`, `styled-components`, and `styled-system` from dotcom, but completed it without breaking GitHub along the way. Enhancing the performance, user experience and delight of our products continues to be top of mind for all of us here at GitHub.

Put native GitHub features to work to review outside collaborator access and archive stale repositories.

Our new repo star history API now provides trends without returning individual identities or requiring unbounded database work. These are the engineering tradeoffs we made.

We’re rebuilding GitHub’s Git infrastructure while GitHub keeps running, creating a foundation for agent-scale software development.
