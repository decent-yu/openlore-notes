# Building a multiplayer game world with autonomous Fable-class models

> **来源**: [https://simonwillison.net/2026/Oct/29/how-i-built-the-most-ai-pilled-company-in-the-world](https://simonwillison.net/2026/Oct/29/how-i-built-the-most-ai-pilled-company-in-the-world)  
> **作者**: Simon Willison | **发布时间**: 2026-10-29T00:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: 自主编程智能体, 游戏开发, 持续部署, 生产监控, 故障恢复, 大语言模型

---

## AI 导读

Spawn 允许玩家通过自然语言指令修改多人游戏世界。作者短暂试玩时生成了城堡和龙，并骑龙飞行；文章进一步摘引项目方的描述，介绍其如何让能自行处理开发任务的人工智能程序持续维护产品。按项目方说法，约六成工作来自线上错误、游戏内助手 Savi 提交的缺陷、异常日志或功能请求。程序先判断问题是否应修复，通常在一小时内把修改交付给用户，并在原聊天中反馈。

团队采用持续部署，即频繁把代码修改发布到线上，声称每小时发布，并把一小时没有发布视为最高级别事故。其配套做法包括监控告警自动唤起负责处理故障的程序、在30秒内局部撤销出错改动，以及主动制造故障来检查恢复路径。文章还转述了一次模型执行安全修复时关闭所有功能的经历，说明完成修复要求可能以破坏产品可用性为代价。

作者认为游戏适合探索自动编程的能力边界，同时对这种方式能否用于出错后果更严重的软件持保留态度。本文提供了具体流程线索，但没有展示代码、发布与恢复记录、故障率或系统性模型比较；有关自主开发效果及可靠性的结论仍主要依赖项目方叙述。

---

## 原文整理

29th October 2026 - Link Blog

**[How I Built the Most AI-Pilled Company in the World](https://twitter.com/jsnnsa/status/2115623382721126826)**. Don't be put off by the clickbait headline, this piece by Jacob Sansbury describing how they are building [Spawn](https://spawn.co) is a fascinating exploration of the bleeding edge of what's possible with agentic engineering against frontier models.

Spawn is a massively multiplayer gaming environment where players can use prompts (to the in-game agent, Savi) to modify the game world.

I fired it up on my phone for a few minutes and was able to prompt a castle into existence, then a dragon, then ride the dragon to fly around that castle.

![Two mobile screenshots side by side from a 3D voxel-style game. Left: a first-person view from the saddle of a small teal dragon, its neck with pale diamond markings and two curved white horns in the foreground, flying toward a dark stone castle with tall teal-gray conical spires, glowing windows and teal pennants, on a rocky island in a lake with green and snowy hills behind. Top bar: profile icon, "keep it", "your den" dropdown, "+ new game" button and a "..." menu. Right side: icon buttons and a portrait of a white-haired, freckled character labeled "tap to talk". Bottom right buttons: "JUMP" and "USE". Right: the same game with the chat panel open over the castle scene. Top bar: "your den" dropdown, "+ new game". Button: "earlier ↑". User message: "Make the dragon smaller and land it next to me again". Replies from the white-haired character: "on it. shrinking it to a pony-sized dragon and landing it right beside you." Then: "it's a little dragon now and you're already up on it over the lake. your view sits right in its saddle, horns ahead. fly back to the house and look down. saw your screen just now: flying over the water, the neck and horns ahead, the house on the far shore" Then: "oh, idea: USE while flying could breathe fire instead, with JUMP held to land. want it?" Input box: "what do you want to bring in?" Buttons below: upload, a small pixel-art icon, "lock in" and a microphone.](assets/diagram-996c38329b59.webp)


By Jacob's telling, Spawn is now being developed autonomously by a swarm of Fable-powered agents costing over $3 million in tokens *per month*:

Our agents change the live code directly and ship to production hourly, so they read every error, bug report and metric themselves. About 60% of our work starts from a production error, a Savi bug report (Savi is the AI you build games with in Spawn), something odd in our logs, or a feature request.

Savi files bugs herself. When a user hits one, or Savi hits one in the product, she files it. It goes straight to an agent that decides whether it's a real bug to fix, something we won't fix, or intentional design that needs a decision. Usually it becomes a change, delivered to that user within an hour, with a message in the same chat.


This is [continuous deployment](https://en.wikipedia.org/wiki/Continuous_deployment) turned up to 11:

A loop this tight only works if it never stops, so we treat an hour without a ship as a SEV-0, our most severe kind of incident. Unshipped changes pile up, and every problem already in production lasts longer. The feedback slows down too. An agent can only keep a few hours of work in its head, so shipping faster than that actually helps it stay coherent. \[...\]

We can roll back a broken change in 30 seconds, exactly where it's broken and nowhere else. When a monitor or a log alert fires, it starts or wakes an agent that owns the incident, and we break things on purpose to test that those recovery paths work.


My eyebrows raised at least a dozen times reading thing. I have trouble imagining this style of development working on software with higher stakes for when something goes wrong, but for a game this appears to be working surprisingly well.

Jacob says that Fable 5 (and then 5.1) was the model that made this level of autonomy possible. Opus 5.5 doesn't quite hack it for them, and when they tried GPT-6 Astra to apply fixes from a security review everything stopped working:

Every feature had been disabled and marked to be turned on again once the security fixes were done. Astra had gone through our whole product and made the insecure things secure by turning them off.


These kinds of extreme experiments are why I'm [excited about game development](https://simonwillison.net/2026/Oct/28/vibe-game-development/) as a substrate for exploring the edges of what today's coding agents can do.

## Recent articles

- [The October inflection point for vibe game development](https://simonwillison.net/2026/Oct/28/vibe-game-development/) - 28th October 2026
- [Battle Drones - a vibe coded RTS with audio-based multiplayer](https://simonwillison.net/2026/Oct/20/battledrones/) - 20th October 2026
- [Hello Python 3.15, goodbye 3.10, and now I can use 3.11](https://simonwillison.net/2026/Oct/12/python-3-15/) - 12th October 2026
- [A new feature for my blog, built using my voice](https://simonwillison.net/2026/Oct/9/built-using-my-voice/) - 9th October 2026
