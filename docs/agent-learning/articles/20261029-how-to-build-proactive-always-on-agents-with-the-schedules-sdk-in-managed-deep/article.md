# How to Build Proactive, Always-On Agents with the Schedules SDK in Managed Deep Agents

> **来源**: [https://www.langchain.com/blog/proactive-agents](https://www.langchain.com/blog/proactive-agents)  
> **作者**: Nathan Drezner | **发布时间**: 2026-10-28T07:15:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: 智能体, cron, 任务调度, Managed Deep Agents, LangChain, 长期记忆

---

## AI 导读

智能体若只在用户发消息时运行，就难以定期整理历史对话、检查新信息或按时提醒。文章提出用 cron（按时间规则触发任务的调度机制）周期性启动智能体，把要完成的任务写成提示，并结合用户的对话和记忆执行。文中将这种做法用于记忆整理、目标推进、信息筛选、提醒，以及定期检查多个智能体的任务进度。
实现上，作者认为需要一个支持灵活设定时间、携带所需能力和权限、在集中触发时错峰执行的调度服务，以及供智能体受控创建和修改任务的接口。随后使用 Managed Deep Agents 的软件开发工具包展示三种写法：在文件中定义固定周期任务；通过工具为当前用户创建、查询和删除周期任务；计算未来时间并安排一次性执行。固定周期示例明确指定了时区。
这些示例展示了如何把定时执行接入智能体应用，但没有展示实际执行结果，也没有说明失败重试、避免重复执行或具体权限校验。文章支持的是一种实现思路和平台接口用法，尚不足以验证其长期运行的可靠性；关于普及程度和未来趋势的判断也缺少可核验的统计依据。

---

## 原文整理

Ever hear of the Baader-Meinhof phenomenon? It's when you learn a word for the first time, and suddenly start hearing it everywhere. I think the same principle applies in software engineering: You learn about a simple, but effective, design pattern, and then suddenly start seeing it everywhere.

We're seeing this happen with proactive agents, where we've noticed that nearly all of your favorite emerging proactive agentic systems are powered by a very simple mechanism: Agent runs as cron tasks, leveraging a more than fifty-year old principle of operating systems for a very modern use case.

The future belongs to proactive, always-on, ambient agents. Launches like Muse, Instinct, Dots, Claude Duet, Hone, and more reflect this.

Crons power a few different capabilities that combine to make these agents proactive, personalized, and [ambient](https://www.langchain.com/blog/introducing-ambient-agents) — that is, they can perform tasks on their own, without being manually instructed.

In this blog, we will go through the capabilities that crons unlock, the two core underlying parts you need in order to build agents like this, and then show examples of how you can do this in [Managed Deep Agents](https://app.notion.com/p/Proactive-Agents-406808527b1780d2b7b2e31c5fcf7748?pvs=21).

We’ve seen the impact crons can have. Proactivity is the new default for agent behaviour because of the breadth of new functions it unlocks. **More than 70% of Managed Deepagent runs on LangSmith come from scheduled runs** rather than a human typing a message.

## What do crons unlock?

Crons, despite being a simple concept, unlock a few different behaviors that shape these proactive, always-on agents.

**Memory:** A distinctive part of all these agents is that they remember facts about you over longer time horizons. Implementing this partially involves running a background job to condense previous conversations into a denser form of memory. This is colloquially called “dreaming”. Under the hood, this is just a cron that runs on some cadence (e.g. once a day) and updates long-term memory with any learnings from reflecting on the conversations.

**Proactiveness**: Part of what makes these agents work is that they initiate communication about an idea that they think you may be interested in. Under the hood, the way that this works is there is some “idea” of what this agent should be accomplishing.

For a general purpose agent, it may be as simple as being a good assistant: taking steps to anticipate what you need next, like flagging an upcoming deadline, following up on an open thread, or surfacing news related to your work. For specialized agents, like with Hone, the agent is working towards an explicit goal. The cron then runs on some cadence (once an hour) and prompts the agent to explore new actions to progress on its goal. These actions then show up as proactive messages to the user.

**Smart Triage:** There is so much noise in the world. A big use case for always-on agents is to monitor a source of information (Slack, Email, etc) and ping the user with relevant messages. Under the hood, this can be implemented as a cron that runs on a cadence (once every ten minutes) and is prompting an agent to look at an event stream and notify the user of anything of note.

**Follow ups**: Crons aren’t just used in the background. Oftentimes users may ask to be reminded about something at some point. Crons can be used to set a schedule to wake up and remind the user then.

**Orchestration:** Agent fleets can be managed by a single orchestration agent with the ability to wake itself up in the future. An orchestrator agent can monitor a fleet with crons. It can sleep while waiting for an asynchronous subtask to finish, and wake itself back up again to check in on progress at regular intervals.

## The two key parts to building proactive agents

In order two build these types of agents, we’ve found that there is generally two key parts.

First is a robust cron server. You will need a server that makes it easy for developers to add crons . They should be able to add them with whatever schedule they need, and include a prompt to run as part of that. If the agent needs certain skills or permissions when running, these should be included. If there are a lot of runs at a specific time point, the server should “jitter” them slightly to spread out the load.

Second, you need an agent-friendly interface. For actions like follow ups, the agent needs to be able to interact with these reminders. It needs an easy - but governed - way to set up and modify reminders.

## Build this with MDA

Managed Deep Agents - which are hosted using [LangSmith Deployments](https://www.langchain.com/langsmith/deployment) - provide a built-in cron server that can dynamically trigger an agent run. A scheduled run is deceptively simple: It is just the prompt that was passed in, triggered at a schedule, with the thread and memory based on the principal who created the schedule. This toolkit above provides foundation for tools that accept only a prompt and a cron schedule.

With Managed Deep Agents, schedules can be defined either statically or dynamically by an agent.

![](assets/diagram-20de60b046b2.png)

For example, building an agent with a fixed schedule as part of its core design - say, a daily sales readout - is just a file in `schedules/`:

```
# schedules/daily_digest.py
from managed_deepagents import define_schedule
schedule = define_schedule(
    cron="0 8 * * 1-5",
    timezone="America/Los_Angeles",
    prompt="Inspect the sales report using your tools and send a KPI readout.",
)
```
Managed Deep Agents also offers dynamic schedules, where end users can prompt your agent to schedule recurring tasks that you didn’t anticipate in development. To support this, wrap the Schedules SDK in tools:

```
# tools/scheduling.py
from langchain.tools import tool
from managed_deepagents import ManagedDeepAgentRuntime
@tool
async def set_schedule(prompt: str, cron: str, runtime: ManagedDeepAgentRuntime) -> str:
    """Run a prompt for the current user on a cron schedule."""
    item = await runtime.schedules.create(owner={"type": "user"}, cron=cron, prompt=prompt)
    return f"Created schedule {item['id']}."
@tool
async def list_schedules(runtime: ManagedDeepAgentRuntime) -> str:
    """List the current user's schedules."""
    page = await runtime.schedules.list(owner={"type": "user"})
    return str(page["items"])
@tool
async def delete_schedule(schedule_id: str, runtime: ManagedDeepAgentRuntime) -> str:
    """Delete one of the current user's schedules."""
    await runtime.schedules.delete(schedule_id, owner={"type": "user"})
    return f"Deleted schedule {schedule_id}."
```
``Developers need flexibility, and the Managed Deep Agents tool system is designed with this in mind. Different agent types may need to expose different toolkits for scheduling. In some cases, a basic `remind_me` wrapping one-off run or a `check_in_later` may be sufficient:

```
from datetime import datetime, timedelta, timezone
from langchain.tools import tool
from managed_deepagents import ManagedDeepAgentRuntime
@tool
async def check_in_later(prompt: str, minutes: int, runtime: ManagedDeepAgentRuntime) -> str:
    """Run a prompt once for the current user, a number of minutes from now."""
    at = datetime.now(timezone.utc) + timedelta(minutes=minutes)
    item = await runtime.schedules.create(owner={"type": "user"}, at=at, prompt=prompt)
    return f"Scheduled a check-in {item['id']}."
```
Get started with adding schedules to your Managed Deep Agents from our documentation: [langch.in/mda](http://langch.in/mda). You’re one `mda init` and `mda deploy` away from distributing a proactive agent within your organization.
