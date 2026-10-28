# The October inflection point for vibe game development

> **来源**: [https://simonwillison.net/2026/Oct/28/vibe-game-development](https://simonwillison.net/2026/Oct/28/vibe-game-development)  
> **作者**: Simon Willison | **发布时间**: 2026-10-28T00:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: AI辅助游戏开发, Vibe Coding, three.js, JavaScript, 游戏设计, WebSockets, 大语言模型

---

## AI 导读

作者关注的问题是：模型能否从做出“看起来像游戏”的程序，进一步做出值得反复游玩的作品。他采用不阅读生成代码、主要靠自然语言要求和试玩反馈推进开发的方式，制作了四款使用 HTML 和 JavaScript 的浏览器游戏，常用 three.js 这个用于浏览器三维图形的库生成场景和模型。
树木找不同游戏据称一次生成即可游玩；奇幻角色扮演游戏先搭建村庄和建筑内部，再扩展地形、地下空间、剧情与魔法，经过多轮试玩修正任务人物位置等问题。即时战略游戏则从早期二维版本继续改进玩法并转为三维，增加多人对战、回放和地图生成，还让大语言模型按“推进五秒后暂停决策”的方式互相对战。南瓜农场尝试更个人化的题材，但作者承认种植时机等机制仍复杂，趣味性尚不满意。
作者据此认为，模型已能在较少人工创意投入下生成有趣的游戏，制作重点开始转向提出独特构想和持续打磨。不过，每个项目不超过四小时的说法仅指作者投入，另有大量模型运行时间，其中农场项目消耗数百万词元，即模型处理文本的计量单位，并运行近两天。文章展示了具体案例和可继续核验的链接，但没有系统玩家测试、代码质量或性能评估；对模型能力高低和游戏趣味性的判断仍是个人经验，不能据此推广到其他游戏类型或正式产品开发。

---

## 原文整理

## The October inflection point for vibe game development

28th October 2026

I’ve written before about how November 2025 was [an inflection point for coding agents](https://simonwillison.net/tags/november-2025-inflection/)—the point where Opus 4.5 and GPT-5.1 crossed a tipping point that meant their respective coding agents became reliable daily drivers for software engineers. I think we’ve reached a new inflection point in October 2026, and this time it’s for **vibe coding computer games**.

Claude Opus 5.5 in particular exhibits meaningful improvements in several areas needed to build a compelling game:

- 3D modeling
- Music generation (see [Scrimshaw Jukebox](https://simonwillison.net/2026/Oct/6/scrimshaw-jukebox/))
- Game design (!)

That last one is particularly important. Models have been able to create something that *looks* like a game for quite a while now, as seen in [my previous experiments](https://simonwillison.net/2026/Aug/7/moonlight-mayhem/). The challenge has been getting to a game that’s actually *fun to play*.

Game design is a deep subject area. I won’t pretend to be well versed in it—I know enough to have respect for it as a craft. Creating a novel, compelling gameplay loop which challenges players at the right level and provides a satisfying learning curve is a fine art. I didn’t think the models could do it.

Then I tried telling Claude Opus 5.5 to “Make it a LOT more fun”... [and it did](https://simonwillison.net/2026/Oct/20/battledrones/#v4-the-fun-update).

In the past few weeks I’ve vibe coded (in the [classic sense of the term](https://simonwillison.net/2025/Mar/19/vibe-coding/)—no looking at the code at all) four games that I’ve genuinely enjoyed playing.

I attended [Foo Camp](https://en.wikipedia.org/wiki/Foo_Camp) in Berkeley over the weekend and met several people who were having similar experiences. Building games that play well is shockingly easy now. The new challenge is coming up with concepts for games that are novel and delightful and that wouldn’t exist without AI assistance.

I haven’t done an exhaustive study, but my current impression is that GPT-6 Astra isn’t *quite* there yet. It can absolutely build a game, but its sense of fun doesn’t match that of Opus 5.5. I expect OpenAI will get there by the end of the year.

Here are the four game projects that have convinced me that the models are now capable of producing fun games with very little guidance. All four are built in HTML and JavaScript and run entirely in the browser.

#### Tree Appreciator

The smallest game I found to be genuinely delightful is [Tree Appreciator](https://tools.simonwillison.net/tree-appreciator). I used the regular Claude chat interface for this—it didn’t even need Claude Code.

I built this on a whim while we were driving along [Skyline Boulevard](https://en.wikipedia.org/wiki/California_State_Route_35) looking at trees. My prompt was:

`Build a three js game called "Tree appreciator" - it shows you two algorithmically generated trees and the game mechanics is about appreciating the differences between them. Think hard about how to make sure it is both fun and beautiful.`


I’ve found that telling models to use [three.js](https://threejs.org/) works really well. Claude doesn’t have its own image generation model but is capable of building full-color 3D models that look and feel excellent.

This game required no iteration or follow-up prompting at all—I got the game in a single shot. It gave me back a charming little mobile-friendly spot-the-difference game. You can [play that here](https://tools.simonwillison.net/tree-appreciator). Here’s [the chat transcript](https://claude.ai/share/ee7de555-f85c-4ff0-96cb-d51ed5e479e9) that built it.

![Screenshot of a web game called Tree Appreciator, a spot-the-difference puzzle showing two side-by-side 3D renders of the same leafy green oak on a grassy hill at golden hour, with orange low-poly hills and a hazy sky behind and pale rocks in the grass. The left tree (Fig. i) has a small brown-and-orange bird perched near the top and a cluster of red-orange autumn leaves in its left canopy; the right tree (Fig. ii) has a small orange squirrel on a branch right of center. Header text: "Tree Appreciator" "No. 001 Quercus aurata 'Silver Echo'" "Golden oak · A low hill at golden hour", with sound, book and help icons at top right. A popup over the scene reads: "These two trees grew from one seed." "Three small things differ between them. Tap one in either tree when you spot it." "Drag to turn both trees. Pinch or scroll to look closer." with a "Got it" button. Panel labels: "Fig. i" "Fig. ii".](assets/diagram-14086d0bff7f.webp)


It’s pleasingly relaxing. The graphics are fun, and collecting trees that you have completed in a “herbarium” provides a nice, cosy sense of accomplishment.

#### Emberwick

This was a more ambitious project that I again built using regular Claude chat. I started out by building an experimental prototype of an engine for a top-down fantasy RPG:

`Build a demo game engine three.js which is a top down (but at an angle) RPG - the player gets to move through the world with one or two companions. Create a fantasy village setting that demonstrates walking around outside and then shows how when the player enters a building the wall between the player and the camera goes transparent so the user can see what is happening`

`Add some NPCs and plenty of furniture and different types of building some with an upstairs - be creative, make it look great`


Here’s [that first version](https://simonw.github.io/emberwick/v01/emberwick.html). It worked really well!

![Screenshot of a low-poly 3D top-down fantasy game showing the interior of a cutaway tavern at dusk, with grass, a lamppost and benches outside. Inside, warm candlelight falls on wooden floorboards: a stone fireplace with a figure in a red armchair on a red rug, a blue armchair with an orange cat on it, a long bar with barrels and bottle shelves where a red-haired barkeep stands, a candle chandelier, a cloth-covered table, small round tables with stools and a blue banner with a gold emblem. A group of three adventurers (a white-haired figure in a blue robe, a red-armored figure with a hammer and a green-clad figure) stands just inside the doorway. Top-left panel text: "Harvest Eve" "17:09" "Tavern & lodging · Taproom" "The Gilded Goose" with buttons "Dawn" "Noon" "Dusk" "Night" and a pause button. A speech bubble above the fireplace reads "♪ la la la ♪". A circular minimap sits at top right marked "N". Bottom center prompt: "F Talk to Old Fennick Regular". Bottom-left party portraits labeled "You" "Brakka" "Elowen". A "?" button sits at bottom right.](assets/diagram-110464a2ea62.webp)


I was inspired to increase my ambitions:

`Make it not so flat - the buildings are great but the landscape itself is completely flat at the moment, fix that`

`Add basements to two of the buildings with a tunnel system joining them below the village`

`Add the start of a rocky mountain on one side of the village containing a cave with a sleeping troll`


Here’s [that version](https://simonw.github.io/emberwick/v02/emberwick.html).

![Screenshot of the same low-poly fantasy game, now in a dim underground cave lit by a campfire with a spit roast, glowing pale mushrooms and blue crystals. A huge green troll lies asleep on a straw bed at top right, surrounded by scattered bones and skulls. The cave also holds a small round pool ringed with stones, a yellow rug, chests, a fenced pen with a sheep, a pile of orange pumpkins and a stone stair down the left side. The party stands in the center: the green-clad figure and the red-armored figure facing the blue-robed figure holding a glowing staff by the fire, who says in a speech bubble "Smell that? Old brandy and older mud." The location panel reads "Harvest Eve" "17:25" "Underground · Under the mountain" "Grumblethorn's Cave".](assets/diagram-1b5d612196c7.webp)


When *that* produced a good result, I decided to push it all the way:

`Now turn this into an actual game - quadruple the size of the playable area, introduce fun game mechanics, add a plot that isn't too cliche, some twists, a magic system, make it really good`


We’re back to prompts like “make it really good” here! Asking for a plot that “isn’t too cliche” seemed to work too—it’s a bit cliché, but still novel enough that I enjoyed following the story.

Unlike Tree Appreciator, this took quite a bit of additional playtesting and prompting to get to a finished version. This was mainly due to gameplay problems—things like a character not being in the right place to complete a quest because a different story element had caused them to move to another location.

Here’s [that final game](https://simonw.github.io/emberwick/v11/emberwick.html). It has a full story that takes about 15 minutes to play through.

![Emberwick title screen. The 3D village sits in the background. The text reads Every fire in Emberwick was lit from the Hearthstone, and every fire dies at midnight unless it is lit again. Tonight the Hearthstone is gone, the wizard is missing, and the village is sure the troll did it. You came for the festival. Someone has other plans for you. Buttons for: Begin, Build your own places, Restore a save](assets/diagram-564973e40b75.webp)


There’s even an experimental level editor if you want to drop in your own houses, furniture, and landscape features.

Once it was all working (and I’d iterated through some gameplay bugs), I played it through from start to finish and found myself emotionally affected by the ending! I’m somewhat embarrassed by this, because *everything* in this game—the graphics, the plot, the characters, the music, the dialog—came from the model. My creative input was pretty much just “it’s a fantasy village and there’s a troll asleep in a cave”.

Here’s [the full shared Claude chat](https://claude.ai/share/840e0bc0-a68e-4894-8eec-8f83bd6ade26) in which I built this game.

I also fed that same initial sequence of prompts through ChatGPT GPT-6 Pro. Here’s [the transcript](https://chatgpt.com/share/6ae176eb-1eac-83e8-ae54-729281fbdb70) and [the finished game](https://lantern-hollow-rpg.simonw.chatgpt.site) for comparison. It looks great, but I don’t think it plays nearly as well.

#### Battle Drones

This is by far my most successful project, which I [wrote about last week](https://simonwillison.net/2026/Oct/20/battledrones/). It’s a Command & Conquer-style Real-Time Strategy game where two robot armies (or four [as of Saturday](https://simonwillison.net/2026/Oct/24/battledrones/)) build units, harvest scrap, and battle over territory.

This is also the project that convinced me that Opus 5.5 knows what “fun” is.

The first versions were built back in July using Claude Fable 5. It got me to a workable 2D Command & Conquer-style game, but it wasn’t great to play.

I prompted Opus 5.5 to “Make it a LOT more fun” and then later to “Port this game to three.js” and the game was transformed. There’s a blow-by-blow account (prompts included) in [my write-up](https://simonwillison.net/2026/Oct/20/battledrones/).

Today, Battle Drones has 2-4 player multiplayer (via a vibe coded WebSockets relay server), medals, replays, seven maps, a random map generator, and a set of “mutators” that add variants of the rules.

It also lets different LLMs play against each other, using a mode where the game advances five seconds and then pauses to give different models time to interpret the new game state and line up their next set of commands. Here’s a replay of [Claude Haiku 5.5 vs. GPT-6 Luna](https://simonw.github.io/battledrones/v11/?replay=battles/showcase/2026-10-14-frostbite-42-claude-haiku-5.5-vs-gpt-6-luna.json):

![Screenshot of a replay of a real-time strategy game between two AI models, shown from a top-down view of a snowy map with a dark frozen lake of cracked ice floes on the left and pine forests at the top and lower right. Top bar: pause button, "6:27", "1×", "anthropic/claude-haiku-5-5 · industry" in blue, "gpt-6-luna · industry" in red, a blue-and-red strength bar labeled "B" and "R", and "0:03". Left notification: "Blue: Pushing turrets at the enemy's south-east edge." Replay panel: "Blue anthropic/claude-haiku-5-5 · industry vs Red gpt-6-luna · industry · frostbite seed 42", a timeline slider at "6:27 / 7:48", and buttons "−30s", "+30s", "All", "Blue's eyes", "Red's eyes", "Notes". In the scene, a group of blue tanks and a large blue Goliath mech trailing smoke advance near the ice edge at lower center, while a column of red turrets (one burning) lines the ice to their north, red tanks and trucks sit by a rocky ridge to the east, and red base buildings occupy the top right. A minimap is in the top right corner and zoom in, zoom out and lightning-bolt buttons are on the lower left.](assets/diagram-07b18167b525.jpg)


Battle Drones is a particularly good example of Opus 5.5’s ability to create 3D models. This page lets you [browse all of the models in the game](https://simonw.github.io/battledrones/models.html).

I released Battle Drones under the Apache 2.0 License last Tuesday. On Wednesday Gabriel Turpin released [Panache](https://github.com/menufactory43/panache/) ([screenshots here](https://simonwillison.net/2026/Oct/21/panache/)), a fork of Battle Drones that reimagined the game for steampunk 1900s Belle Époque France, with new models and music and an animated intro sequence. Open source at work!

#### Pumpkin Farmer, aka Quarter Sun Cove

I live in [Half Moon Bay, California](https://en.wikipedia.org/wiki/Half_Moon_Bay,_California), home of the annual [Half Moon Bay Art and Pumpkin Festival](https://en.wikipedia.org/wiki/Half_Moon_Bay_Art_and_Pumpkin_Festival), which includes a giant pumpkin weigh-off competition.

I decided to try building a game that wasn’t just a cheap imitation of something that already existed, so I set out to create a game about farming pumpkins, growing giant pumpkins, and handling the rush of the October pumpkin season.

I called it “Quarter Sun Cove”.

I’m still iterating on this project, but it’s shaping up quite nicely. It’s a cosy farming simulator where you spend the year tending to your giant pumpkin and planting out regular-sized pumpkins for sale in October, then sell pumpkins during pumpkin season and enter your giant pumpkin in the weigh-off.

Every character in it is a (vegetarian) animal. The mayor of the town is a pelican. You can be a pelican, skunk, raccoon, possum, beaver, coyote, fox, owl, sea otter, or harbor seal. Those species all later show up (accompanied by their children) to buy pumpkins from you.

It’s *absurdly* cute.

![Screenshot of an isometric low-poly pumpkin farming game. In the bottom left, a huge giant pumpkin, far bigger than anything around it, sits in its own fenced plot under string lights. Beyond it are rows of pumpkin vines with a scarecrow and animal visitors (pelicans, beavers and otters) picking pumpkins and browsing the farm stand, plus a red barn, beehives, a cornfield, parked cars and a dock on the water, with fog drifting across. A side panel shows stats for the patch row "Row 3 · Warty" with a "Cut 7 → barn" button, and a customer review at the bottom reads "★★★★★ Carried our pumpkin like a baby. It was heavier than the baby."](assets/diagram-114bf505ddb4.webp)


You can [play the latest version here](https://simonw.github.io/pumpkin-farmer/). I’m not convinced by the game mechanics yet—it’s actually quite complicated: you have to decide when to plant pumpkins such that they will become ripe at the weekends in October with the most visitors.

Here’s the [initial brainstorming session](https://claude.ai/code/cshk_01V63sUyETi6UNbGUyjSYbHd13Mq93U6ty824Ha4eAxF4i), which resulted in [this spec file](https://github.com/simonw/pumpkin-farmer/blob/cdcf12083f403946b5c2a9e04312bbb42359bf58/SPEC.md). Then I had Opus 5.5 Ultracode burn millions of tokens and nearly two days building it.

I told Claude to “surprise and delight the user frequently” and the result is *so many* little details and easter eggs. There are mischievous raccoons, an elephant seal that runs the weigh-off, harbor seals that roll their pumpkins back into the ocean, a wandering goat, and superstitions about growing giant pumpkins that *sometimes* come true.

This isn’t my best game yet—it’s not as fun to play as Battle Drones—but it represents the kind of game I want more of: weird, fun, uniquely personal—something that wouldn’t have been built in the before-times.

#### Four times as insulting

If you are offended by AI-generated art, AI-generated music, AI-generated writing, and AI-generated code, I have bad news for you: vibe coded games combine **all of those at once**.

The wider games community already has an aversion to AI-generated assets in games. This has been a growing issue over the past few years as tools like Midjourney have enabled developers to generate assets for their projects. [This IGN story](https://www.ign.com/articles/steam-fans-are-begging-for-a-no-ai-filter-as-steam-next-fest-is-filled-with-slop) covering complaints about slop in Valve’s Steam Next Fest is a good recent example.

We now have games where *everything*—code, assets, dialog, and even the game design itself—can be outsourced to the models. And the results don’t suck!

Building a computer game used to be one of the most challenging projects in all of software development, combining a diverse range of skills and requiring intense focus and effort over many months or even years. And now here I am spitting out a playable multiplayer RTS by prompting on my phone!

I expect that, as we’ve already seen with programming, this will result in a split. Some players will refuse to have anything to do with games that were built in this way. Others may embrace this as opening up game development to a much wider audience. I lean in that direction!

I certainly won’t be entering any of my vibe coded games in traditional game jams or competitions—I don’t feel like I’ve earned that right. We’ll likely see new avenues emerge for vibe game developers to compare their projects.

#### What happens if someone actually puts the work in?

The thing I find both exciting and shameful about this is that I have got interesting results from *so little creative input*. The projects I’ve described here are loosely ordered by how much effort I’ve put into them. Not a single project represents more than four hours of my work (plus many more hours of Claude Code churning away by itself).

Now imagine what could be achieved by anyone who takes this seriously, invests significant creative time, applies better taste in game design, and then playtests and iterates on the result until it’s as good as they can make it.

I want to see games that wouldn’t have existed without these new tools.

#### You should build a game!

Building games in this way is *so much fun*. It’s the perfect application for vibe coding because it’s so low stakes: how much damage can you cause with a bug in a personal vibe coded game project?

(OK, you can do some damage... my attempt at audio-based multiplayer gave my wife Natalie [a nasty headache](https://simonwillison.net/2026/Oct/20/battledrones/#update-online-multiplayer-support)!)

All of my games so far have been HTML and JavaScript using static hosting (generally GitHub Pages or Cloudflare Pages). Modern browsers—including mobile browsers—are incredibly capable. Frontier models already know libraries like [three.js](https://threejs.org) and [Phaser](https://github.com/phaserjs/phaser). I’m finding these mobile web games to be indistinguishable from the native experience, at least for the kinds of games I’m trying to build.

I suggest starting with a prompt like this:

`Build a mobile-friendly three.js game where...`


This works on both [chatgpt.com](https://chatgpt.com/) and [claude.ai](https://claude.ai)—you don’t even need to fire up a full coding agent. As one last example, here’s “Build a mobile-friendly three.js game where a mouse has to hide from a cat” [by Claude Opus 5.5 Extra](https://claude.ai/artifact/RRyEndXLSPQbcig3BgzgCZ) and [by GPT-6 Extra High](https://squeak-and-seek-mouse-1027.simonw.chatgpt.site/).

Switch to Codex or Claude Code once your project passes the proof-of-concept phase.

And if the game isn’t fun enough, add this:

`Make it a LOT more fun`


## More recent articles

- [Battle Drones - a vibe coded RTS with audio-based multiplayer](https://simonwillison.net/2026/Oct/20/battledrones/) - 20th October 2026
- [Hello Python 3.15, goodbye 3.10, and now I can use 3.11](https://simonwillison.net/2026/Oct/12/python-3-15/) - 12th October 2026
- [A new feature for my blog, built using my voice](https://simonwillison.net/2026/Oct/9/built-using-my-voice/) - 9th October 2026
- [Claude Haiku 5.5](https://simonwillison.net/2026/Oct/7/claude-haiku-5-5/) - 7th October 2026
