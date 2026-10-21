# Battle Drones—a vibe coded RTS with audio-based multiplayer

> **来源**: [https://simonwillison.net/2026/Oct/20/battledrones](https://simonwillison.net/2026/Oct/20/battledrones)  
> **作者**: Simon Willison | **发布时间**: 2026-10-20T00:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: AI辅助编程, 即时战略游戏, 游戏平衡性测试, Three.js, 程序化地图生成, 确定性模拟, 音频调制解调器, 大型语言模型对战

---

## AI 导读

作者希望做出一款能在手机上玩的即时战略游戏，也就是玩家持续建造、调兵并争夺资源的游戏。最初版本已有基本操作，却缺乏吸引力；文章围绕这一问题，记录了借助 AI 编程工具持续修改玩法、画面和联机功能的过程。

作者主要通过自然语言提出目标，由工具编写代码、生成图像并运行模拟对战，自己反复试玩。一次要求“更好玩”的修改加入了开局选择、争夺目标、支援能力和更多模式，并用上千场模拟对战调整平衡。文中引用的发布说明称，普通难度下超过25分钟仍未结束的对局比例由14%降至0.9%，曾落后至少800资源的一方获胜的比例由24%升至46%；这些指标反映对局节奏和翻盘情况，不能直接等同于真人体验改善。作者还通过岛屿目标必须由运输机运送步兵占领的规则，让原本少用的单位获得明确用途。

后续版本改用 Three.js 三维图形库，以代码构造模型，并加入由大型语言模型控制单位、记录和回放对战的功能。为避免不同浏览器的微小计算差异让回放失步，工具替换了部分数学函数，使相同输入能够产生一致的模拟结果。

最后，作者尝试让附近设备通过声音传递消息，实现无需专门联机服务器的双人游戏，并报告已在笔记本、iPhone 和 iPad 之间试用。声音联机刚刚完成，正文没有提供通信协议细节或性能测量。作者几乎没有审查代码，因此文章主要支持这种开发方式能够产出可试玩作品的个案观察，尚不能证明代码质量、广泛兼容性或长期稳定性。

---

## 原文整理

## Battle Drones—a vibe coded RTS with audio-based multiplayer

20th October 2026

My latest vibe coded game experiment is a Command & Conquer style RTS (Real Time Strategy) game called [Battle Drones](https://github.com/simonw/battledrones#readme) ([play it here](https://simonw.github.io/battledrones/)). Unlike [my previous experiments](https://simonwillison.net/2026/Aug/5/raccoon-heist/), this one is *actually fun to play*. I’m somewhat embarrassed to admit that I got there by telling Claude to “Make it a LOT more fun”. It also supports multiplayer between nearby devices using audio to synchronize game state! Here’s what I’ve built so far, and some of the things I learned along the way.

#### Battle Drones

My initial goal was to build an RTS game I could play on my phone, in the style of [Command & Conquer](<https://en.wikipedia.org/wiki/Command_&_Conquer_(1995_video_game)>). I have fond memories of the 1995 original (I liked Red Alert too) and wanted to recapture a little bit of the magic.

After 12 major iterations (and [321 commits](https://github.com/simonw/battledrones/commits/main/) by coding agents, mainly Claude Code) I now have a browser-based game that works on mobile and desktop, has multiple play modes, several maps (including a random map generator) and even supports multiplayer for two devices within microphone range of each other.

Here’s a video demo of Battle Drones running in a desktop browser:

Features so far include:

- 9 buildings, 7 ground vehicles, 6 aerial vehicles, 3 infantry units—all 3D models, which you can [browse here](https://simonw.github.io/battledrones/models.html)
- Fog of war
- Different weapons are needed to target aerial or ground vehicles
- **Six maps**, across five biomes—temperate, desert, arctic, volcanic, tropical
- A random map generator
- **Skirmish mode**—red vs. blue on a rotationally symmetrical map
- **Last Stand mode**—one player holds out against as many waves of enemies as possible
- Area effect weapons: Barrage, EMP, Drop Pod, Sky Hammer
- **Crate drops** with bonus resources
- In-game music that responds to events—it gets more intense as fighting breaks out
- Different in-game **AI personalities** (Balanced, Rush, Air Swarm, Boom, Skyforce, Ironclad)
- **AI vs. AI** mode, to watch those personalities play each other
- The **Goliath**—a giant stomping mech with laser eyes and arm cannons and the ability to establish a new base
- **Mutators** that add extra rules or twists
- A “Daily Op” challenge
- Ranks and achievements (**Medals**)
- **Agent vs. Agent** mode where two LLMs can play Battle Drones against each other
- A **multiplayer mode** based on two nearby phones communicating via audio messages, an extension of my earlier experiment with [Chirp Pong](https://simonwillison.net/2026/Oct/15/chirp-pong/)

#### How I built this

Battle Drones is heavily vibe coded—I’ve reviewed almost none of the code, and it’s built in a language (TypeScript) that I’ve hardly used myself. I’ve playtested it quite a bit though, and it’s been through a large number of iterations since I started the project.

Here’s a rough history of the project so far, including key prompts and links to transcripts.

#### v1: the initial prototype

I started out with Claude Fable 5 back in July:

`We are going to build a mobile friendly RTS game - command and conquer style - called Battle Drones, in which two armies of drones (both flying and ground-based) battle for territory and resources`

`Let's start with a MVP design, ask me some questions`


Here’s a [shared transcript](https://claude.ai/code/session_015RYeKm4ji4N16oqthyX1b1). I’d given Claude Code for Web an environment variable with an OpenAI API key and told it to use the OpenAI [Image Generation API](https://developers.openai.com/api/docs/guides/image-generation) for assets, which got me some basic sprites.

You can [try v1 here](https://simonw.github.io/battledrones/v1/)—it looked like this:

![Game screen. Down the right are text icons to build Fab, Airpad, Turret, Harv, Sting, Flak, Scout, Wasp, Raptor. There is a blocky fog of war on a 2D battlefield. There are some gems, an HQ building being attacked by badly rendered tanks, and some blue vehicles dotted around. It's not very visually impressive but it does look like a basic RTS.](assets/diagram-c906646f0c8e.webp)


It’s recognizable as an RTS, has building mechanics and the ability to select and direct units, and a basic AI to play against.

This first version was barely a proof of concept, and it wasn’t any fun to play.

Claude used the [Phaser](https://github.com/phaserjs/phaser) 2D game framework for these early versions. I didn’t even realize this until several iterations later!

#### v2: full sprites

The second version ([try v2 here](https://simonw.github.io/battledrones/v2/)) added sprites for all of the vehicles and the “AI vs. AI” mode, so I could watch the game playing without having to directly control it myself.

Claude proved reasonably competent at prompting OpenAI’s image models to generate sprites. You can browse the v2 sprites in the [public/assets/sprites](https://github.com/simonw/battledrones/tree/v2/public/assets/sprites) folder.

![Four flying units, three turrets, a harvester and an airpad all have proper sprites now. The scrap field has a new design which doesn't look great but is at least using a graphic.](assets/diagram-a2cffa0190cc.webp)


#### v3: varied terrain

It still wasn’t much fun, and the graphics were boring. It was now October, and I had upgraded to Claude Opus 5.5. I told it:

`Massively improve this game. Make the graphics and gameplay better (keep it mobile friendly). Render new graphics using the latest OpenAI GPT image API (lookup details) with the API key in the environment variable. Add new unit types and balance the gameplay better. Also add new buildings and better, more varied terrain.`


This resulted in five new maps with generated background tiles in a variety of styles, [visible here](https://github.com/simonw/battledrones/commit/aa22f08a34347d7d9f94b33f44a73b4deb046aa1). Opus also regenerated the sprites, making them much more visually appealing. You can [try v3 here](https://simonw.github.io/battledrones/v3/)—it looks like this:

![The map is now an ice map with snow and ice and broken ice - it looks really good. The vehicles and buildings have all had a cosmetic upgrade too. This is starting to look like a real game.](assets/diagram-aa469ad70cb4.jpg)


This is starting to feel like a real game! It still wasn’t very fun to play though—I found myself getting bored pretty quickly.

#### v4: the fun update

So far my Battle Drones experiment was confirming my [previous belief](https://simonwillison.net/2026/Sep/27/2026-in-llms-so-far/#simon-willison-2026-in-llms.059.webp) that coding agents can build a thing that looks like a game but can’t actually make something that’s genuinely compelling to play.

I shared this theory with someone over dinner, and then thought to myself that I should give the models a proper chance to prove me wrong. I prompted Opus 5.5 on my phone:

`This game isn't fun enough yet. Make it a LOT more fun.`


This time I was running in [Ultracode mode](https://code.claude.com/docs/en/workflows), where Claude Code (in this case Claude Code for Web) runs “dynamic workflows”, described in their docs as:

A dynamic workflow is a JavaScript script that orchestrates many [subagents](https://code.claude.com/docs/en/sub-agents) at once.


This kicked off *many hours* of work. I started it in the evening and didn’t get a playable result back until late into the next morning. But the results were spectacular.

You can [try v4 here](https://simonw.github.io/battledrones/v4/)—it looks like this:

![The lava map. Uplink satellites are being taken, crystals look better, Elite text is shown above a unit, there is an arrow marked Fight to jump to a fight, the AI vs AI mode has a speed controller - plenty of new features here.](assets/diagram-408c230cc73b.jpg)


I had Claude [write its own release notes](https://github.com/simonw/battledrones/blob/main/Battle%20Drones%20The%20Fun%20Update%20%E2%80%94%20Release%20Notes.md)—here are the highlights:

Every skirmish and Daily Op now opens with a choice, gives the side that’s losing a way back, and pulls the armies into the middle of the map. There are also two new ways to play.

- **Openers**: pick Blitz, Industry or Air Wing before the clock starts.
- **Call-down powers**: Barrage, EMP, Drop Pod and the Sky Hammer orbital strike.
- **Objectives and salvage**: Uplink relays, supply drops and wrecks worth scrap.
- **Feel**: combat juice, adaptive music, unit voices and an announcer.
- **Command**: battle alerts, game speed, saved squads and touch placement.
- **New modes**: Last Stand survival waves and an 8-mission Ops campaign.
- **Progression**: heroes, an after-action report with awards, a career, 7 mutators and a Daily Op.

Fights run longer and swing more. In 1,200 AI-vs-AI test matches on Normal, games still undecided after 25 minutes fell from 14% to 0.9%. The share of wins by a side that had been 800+ scrap behind rose from 24% to 46%.


Part of the reason Claude took so long was that it ran over a thousand simulated playtest sessions, and used the results to rebalance the existing units, add new ones, and introduce new features.

#### v5: Battle Drones goes 3D

After the success of Ultracode at “Make it a LOT more fun”, I decided to be more ambitious:

`Port this game to three.js with 3D models for all of the buildings and vehicles and landscapes`


That was another overnight run. I woke up to [this v5 version](https://simonw.github.io/battledrones/v5/):

![Now it is in 3D - the battlefield is at a slight angle, the buildings and vehicles are all 3D models, the rocks and trees look a bit 3D as well.](assets/diagram-ec8c519318c4.jpg)


The new 3D models were entirely constructed by Claude in TypeScript code.

v6 ([play v6 here](https://simonw.github.io/battledrones/v6/)) added some more visual polish. v7 ([play v7](https://simonw.github.io/battledrones/v7/)) added robot infantry (troopers, rocket squads, snipers) and a “Condor” troop carrying aircraft.

#### v8: Archipelago

I was beginning to feel guilty at how little actual design input I had into the game—it really had been a case of me saying “make it more fun” and “add visual polish” for the past few releases.

I’d noticed that the AI didn’t use the new infantry and troop carrying aircraft from v7 very often—there wasn’t a great strategic reason to use them.

It would be neat if we had a new map with a new game mechanic that actively encouraged players to load up Condors with infantry robots and send them out into the map.

This resulted in my longest prompt yet:

`Design a new map. Goal is to make different strategic incentives that encourage use of Infantry and Condors and aerial vehicles.`

`To do this the map will have limited harvester resources but several more uplinks - but uplinks will be on islands that are inaccessible by land vehicles or infantry. These uplinks can also not be claimed using air vehicles. The ONLY way to get them is to fly infantry there in a Condor.`

`Armies always start with a Condor and a single squad. If an army has no Condor a new one will be provided for free, flying in from the edge closest to their HQ and landing there. That free condor will come with a single free squad on board.`

`The two HQs should be entirely separated by water, but still accessible by vehicles thanks to two bridges across the map in different places`

`I want to encourage a battle over uplinks.`

`The map should be the times the size of other maps - still square`

`First research and then critique this design. Only build after we have agreed on any modifications`

`You get to design a new biome for this`


The result was the tropical **Archipelago** map, with seven island uplinks that players can only capture by flying out infantry in a Condor. It’s definitely the most fun map to play, once you’ve got the hang of the basic game mechanics from the simpler maps. The [v9 edition](https://simonw.github.io/battledrones/v9/) made that map a little more interesting—it now looks like this:

![Three islands, each with an uplink satellite. Blue holds one, red holds another, A blue condor is flying towards the red island, which is guarded by a red plane. There is a bridge nearby. The theme is tropical islands and looks very pleasant.](assets/diagram-00a86343f605.jpg)


#### v10: Goliath, and agent battles

I decided to add giant robot mechs to the game—think Jaegers from [Pacific Rim](<https://en.wikipedia.org/wiki/Pacific_Rim_(film)#Plot>), or Titans from Warhammer 40,000.

Here are some of the prompts I used to brainstorm with Claude:

`Help me brainstorm ideas for adding giant robots to this game - think dreadnoughts or titans from W20K - how could we do this in a way that enhanced gameplay, kept things balanced, added unique twist? Don't overthink this yet`


`Let's keep brainstorming. I like the idea that it's a mobile base - that can accept harvested scrap and can have buildings built near it (even if it later moves on) and can produce infantry units too. An interaction with condor would be neat but I don't know what that could be.`


`Idea: as it gets damage it loses the ability to fire some weapons - by default it has eye laser (like lancers) and arm canons (like titan) and two shoulder mounted mortars, plus machine guns similar to a squad of troops`


`Damage knocks out mortars and then laser - it always has guns. You can't build next to a building that was only put there because of the mech, you need the mech there for off-base building and only within three squares of it`


`[...] Lets do two condors can carry the mech`


The end result is *really cool*.

Goliaths can be constructed using a Gantry—a building that can only be created after first building a Fab, Bay, Airpad, and Tech Lab. You can see the Goliath under construction, and it then walks out from the Gantry and can march around the map.

Goliaths can “anchor”, which digs them into the ground. In their anchored state they can act as a new HQ, constructing infantry units and additional buildings in a new area.

Two Condor aircraft can hook onto a Goliath, at which point it can be flown around the map!

The game now ends when you destroy the enemy’s HQ building *and* any Goliaths they might have built.

There are also a couple of mutators relating to Goliaths:

- **Goliath Armada** starts each side with four airborne Goliaths (each with a pair of Condors) and no other units.
- **Robot Love** adds a special rule: if enemy Goliaths get close to each other on the battlefield they fall in love and the game ends in a draw. (Thank Natalie for that one).

#### Agent battles

The other v10 feature is **agent battles**. In musing about how multiplayer could work I had an idea: what if different LLMs could fight it out in the game?

`Fun extra challenge: think about how we might make multiplayer available to coding agents. It would be neat if we could set up a battle between eg GPT-6 Sol and Claude Opus 5.5 where each model has access to a tool which gives them information about the game (presumably non-visual, but only the information they could see through fog of war etc) and lets them control their units etc [...]`

`Consult ~/dev/llm/docs and include notes in that file about how LLM could be used here via a new tools plugin`


Opus 5.5 went to work on this, producing [this AGENT-BATTLES.md proposal](https://github.com/simonw/battledrones/blob/7001ed186c8b8aef4acb27f680ecf4c09b607e54/AGENT-BATTLES.md) and then implementing and testing it. The feature also includes the ability to record and replay these battles—here’s [a replay of Claude Haiku 5.5 vs. GPT-6 Luna](https://simonw.github.io/battledrones/v11/?replay=battles/showcase/2026-10-14-frostbite-42-claude-haiku-5.5-vs-gpt-6-luna.json), and here’s that same battle [in a separate debug tool](https://simonw.github.io/battledrones/record-viewer.html?replay=battles/showcase/2026-10-14-frostbite-42-claude-haiku-5.5-vs-gpt-6-luna.json#r1) that shows the underlying prompt responses.

![Screenshot of a replay of a real-time strategy game between two AI models, shown from a top-down view of a snowy map with a dark frozen lake of cracked ice floes on the left and pine forests at the top and lower right. Top bar: pause button, "6:27", "1×", "anthropic/claude-haiku-5-5 · industry" in blue, "gpt-6-luna · industry" in red, a blue-and-red strength bar labeled "B" and "R", and "0:03". Left notification: "Blue: Pushing turrets at the enemy's south-east edge." Replay panel: "Blue anthropic/claude-haiku-5-5 · industry vs Red gpt-6-luna · industry · frostbite seed 42", a timeline slider at "6:27 / 7:48", and buttons "−30s", "+30s", "All", "Blue's eyes", "Red's eyes", "Notes". In the scene, a group of blue tanks and a large blue Goliath mech trailing smoke advance near the ice edge at lower center, while a column of red turrets (one burning) lines the ice to their north, red tanks and trucks sit by a rocky ridge to the east, and red base buildings occupy the top right. A minimap is in the top right corner and zoom in, zoom out and lightning-bolt buttons are on the lower left.](assets/diagram-07b18167b525.jpg)


The prompt that drives agent battles [can be seen here](https://github.com/simonw/battledrones/blob/d584e351d9d28c4ba058b78ddcca9b93cd9bcd33/llm-tools-battledrones/llm_tools_battledrones.py#L106-L117).

There are a few more replays like this [linked in the README](https://github.com/simonw/battledrones/blob/main/README.md#watch-ai-models-play-it). I hope to turn this into a full league table at some point in the future.

#### v11: Random Maps

This version exists purely because I glanced at my [Claude Usage](https://claude.ai/new#settings/usage) page one day and saw that I’d only used 75% of my weekly allowance and it was resetting in 45 minutes. I decided to see if I could get an extra Battle Drones feature from my remaining tokens, and picked “random map” as the target:

`Design and implement a mechanism for generating a random map using any of the available biomes - it should be the size of the largest map and rivers and bridges and rocks and forest and scrap and crystals are all in play. Make them rotationally symmetric and use a seed to allow them to be selected or reused later. You have 45 minutes to build this! Be creative.`


Here’s [the transcript](https://claude.ai/code/session_01CfwMwDaK1kLKnRV6di8VMT), and an image Claude generated to show me its work:

![Grid of ten top-down game maps, two per biome (temperate, desert, arctic, volcanic, tropical), each with a river or lava flow, bridges and roads. Title: "Battle Drones: generated maps (108x108, point-symmetric), 2 per biome, runtime ground painter". Each map is captioned with a name and layout type plus a seed and crossing counts, e.g. "Hawthorn Delta · fork" "random-temperate-1x8a6r2 · 6 bridges, 4 fords"; layout types are fork, badlands, crossing and lake.](assets/diagram-922e5c0b881e.jpg)


#### v12: Multiplayer via sound

I only *just* got this working, so I’ve not had much time to actively test it yet. The latest Battle Drones now supports multiplayer via two devices sat within microphone distance of each other!

This feature started out as my [Chirp Pong](https://simonwillison.net/2026/Oct/15/chirp-pong/) experiment from last week. I pointed Claude Code at that working proof of concept:

`Clone Simonw/tools from GitHub and look at recent branches for the example pong game that uses web microphone to implement multiplayer between two phones via audio signals`

`Build a version of that for battledrones`


Opus built [this Radio Probe tool](https://simonw.github.io/battledrones/radio-probe.html) to measure how much bandwidth we could get between two devices and, based on positive results from that, implemented a full audio modem system. I’ve tried it between laptops, iPhones and iPads so far and it seems to work really well!

Why use sound for this? Because I [love static HTML](https://simonwillison.net/2025/Dec/10/html-tools/#the-anatomy-of-an-html-tool) that costs nothing to run, and I don’t want to spin up a server somewhere and then stay responsible for keeping it operating for as long as anyone might want to play the game.

#### Some thoughts

I know almost *nothing* about game development. This makes it a great opportunity to try out coding agents in a field where my existing software engineering experience has less impact on the results.

Games are also super low stakes. The worst that can happen is one of my silly game experiments suffers a bug.

This morning ([via Hacker News](https://news.ycombinator.com/item?id=50168190)) I found [Steam Fans Are Begging for a No-AI Filter as Steam Next Fest Is Filled With Slop](https://www.ign.com/articles/steam-fans-are-begging-for-a-no-ai-filter-as-steam-next-fest-is-filled-with-slop). I wouldn’t personally dream of submitting Battle Drones to an event like this—it’s still way too low effort to warrant consideration as a legitimate indie gaming project.

... but I can now see a path to vibe coded games like this being taken seriously. Battle Drones is fun, it looks good, it has innovative features (the multiplayer-over-audio thing is a first for an RTS as far as I can tell).

I’d love to see what someone with better taste than me and a driving passion to produce something truly exceptional could build!

A few other observations from this project:

- Static HTML and JavaScript is a fantastic medium for games development. WebGL works great, and [Three.js](https://github.com/mrdoob/three.js/) (and [Phaser](https://github.com/phaserjs/phaser) before it) are excellent platforms to build on that the current models know inside out.
- This also lets them run as [Progressive Web Apps](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps)—I didn’t ask for it, but Battle Drones works offline. I was able to play it on a plane despite never having directly installed it—my phone had it cached.
- Giving Claude an OpenAI API key and telling it to generate images using OpenAI’s models works really well. It can view the images it has generated and iterate on the prompts.
- Opus 5.5 is spectacular at 3D modeling using Three.js (and Blender, and no doubt other 3D applications as well). The [same is true](https://til.simonwillison.net/llms/blender-coding-agents-macos) of GPT-6 Astra. They are absolutely capable of building models for 3D game projects.
- For features like replay and multiplayer it’s useful to have a deterministic simulation engine. At one point Claude had to replace calls to `Math.sin()` and `Math.cos()` and friends with [custom versions](https://github.com/simonw/battledrones/blob/v12/src/sim/fmath.ts) because different browsers (Chrome vs. Safari, for example) returned very slightly different numbers, desynchronizing the replay.
- Opus 5.5 can write music! There are no audio files in this project—the soundtrack is entirely generated through code, varies across different maps, and even gets more intense as the battle heats up. See [Scrimshaw Jukebox](https://simonwillison.net/2026/Oct/6/scrimshaw-jukebox/) for a previous experiment with Claude for music composition.
- I still can’t get over how well the prompt “Make it a LOT more fun” worked.
- Tools are effectively free. I prompted up the [Radio Probe](https://simonw.github.io/battledrones/radio-probe.html), the [Model browser](https://simonw.github.io/battledrones/models.html), the [Agent vs. Agent debug tool](https://simonw.github.io/battledrones/record-viewer.html?replay=battles/showcase/2026-10-14-frostbite-42-claude-haiku-5.5-vs-gpt-6-luna.json#r1), and Claude Code [built a bunch of tools](https://github.com/simonw/battledrones/tree/main/tools) for itself that I haven’t even explored myself.
- Audio-based multiplayer between static HTML games works! I hope to see a whole lot more games use this same pattern—in digging around today I’ve not found much prior art beyond [Siner](https://github.com/AlonHor/siner) by Alon Horesh and [ggwave](https://github.com/ggerganov/ggwave) by Georgi Gerganov (also of `llama.cpp` fame).
- Claude Code “Ultracode” burns tokens like there’s no tomorrow, but it can also solve complex problems (like adding audio-based multiplayer) from a single prompt.
- I’m still not quite over the *guilt* of building software in this way. I don’t feel like I’ve earned the right to create something this good, in a field in which I have so little prior experience! At this point I’ve shown it to enough people in private that it felt wrong not to publish it for everyone else to see and learn from.

## More recent articles

- [Hello Python 3.15, goodbye 3.10, and now I can use 3.11](https://simonwillison.net/2026/Oct/12/python-3-15/) - 12th October 2026
- [A new feature for my blog, built using my voice](https://simonwillison.net/2026/Oct/9/built-using-my-voice/) - 9th October 2026
- [Claude Haiku 5.5](https://simonwillison.net/2026/Oct/7/claude-haiku-5-5/) - 7th October 2026
- [We're going to need default hard budget caps on pretty much everything](https://simonwillison.net/2026/Oct/3/default-hard-budget-caps/) - 3rd October 2026
