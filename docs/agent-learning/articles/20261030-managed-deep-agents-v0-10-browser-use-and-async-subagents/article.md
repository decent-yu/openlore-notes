# Managed Deep Agents v0.10: Browser Use and Async Subagents

> **来源**: [https://www.langchain.com/blog/managed-deep-agents-browser-use-and-asynchronous-subagents](https://www.langchain.com/blog/managed-deep-agents-browser-use-and-asynchronous-subagents)  
> **作者**: Nathan Drezner Sydney Runkle | **发布时间**: 2026-10-29T16:54:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: Managed Deep Agents, 浏览器自动化, 异步子智能体, 人机协作, 会话隔离, Slack 集成, 身份映射

---

## AI 导读

文章针对两个问题介绍新版本：智能体无法直接操作只有网页、没有应用程序接口（API）的服务；主智能体等待耗时助手任务时，用户也无法继续交互。第一项做法是给智能体配置浏览器，在隔离运行环境中提供读取页面、点击、截图和人工接管工具。浏览器默认按对话隔离，也可按用户复用登录状态。遇到登录或双重验证时，用户接管远程窗口，完成后交还，智能体检查页面并继续；接管等待不会阻塞其他工作，链接在15分钟后失效。文章称这种方式可避免密码进入模型消息和追踪记录，但未展示验证过程。第二项做法是将子智能体，即使用独立上下文执行分工任务的助手，设为异步运行：它在独立对话线程中执行任务，主智能体可以继续交流，并能检查、调整或取消任务。任务完成后主动通知主智能体；结果何时处理取决于主智能体正在运行、空闲还是等待审批。助手共享主智能体的文件运行环境，相关审批可回到原始 Slack 消息串处理。文章还介绍回复来源记录、停止运行、用户身份映射及升级中的不兼容变更。整体提供了配置入口和预期行为，但没有实测案例、性能对比，也未详述任务失败恢复或共享文件的并发处理，因此更适合作为接入导读。

---

## 原文整理

## Key Takeaways

- **Browser use:** Your agent can book travel or file a claim on sites that have no API. When a site asks you to sign in, the agent hands you the browser window, so your password stays out of the model's messages and traces.
- **Async subagents:** A long job can go to a subagent that runs in the background while your agent keeps talking with you. You can ask the agent to check on the job or change its course, and the result comes back to the conversation when the subagent finishes.
- **Thread origins and stopping runs:** Replies go back to the Slack thread a request started in, including replies from a scheduled run or from a subagent that finished later. A Stop button in the thread cancels its runs.


[Managed Deep Agents](https://docs.langchain.com/langsmith/python/managed-deep-agents) v0.10 is out today. It adds [browser use](https://docs.langchain.com/langsmith/python/managed-deep-agents-browser-use), so your agent can work on websites that have no API, and [async subagents](https://docs.langchain.com/langsmith/python/managed-deep-agents-subagents#async-subagents), which run long tasks in the background while your agent keeps talking to you.

In this post, we cover why agents need each one and how to turn it on, then the rest of the release.

## Give your agent a browser

Booking travel, canceling a subscription, or filing a claim all happen on sites built for people clicking through pages, and there's often no API an agent can call. While agents can reason through how to complete the task, they can’t take action on websites without an API. Fetching the page with `curl` doesn't get far either, it produces HTML before any JavaScript renders the page and the agent can't click anything.

Personal agents like [Instinct](https://instinct.com/) and Meta's [Muse](https://ai.meta.com/muse/) can browse websites to unlock a broader set of use cases, and [DataDome](https://finance.yahoo.com/technology/ai/articles/datadome-report-agentic-browser-traffic-130000558.html)saw agentic browser requests jump 4.7x in September alone. Coding agents need a browser too, to check that a change to an app actually renders and works.

We cover where a browser helps and how it works under the hood in “[Browser use in Managed Deep Agents](https://www.langchain.com/blog/browser-use-in-managed-deep-agents)”. To turn it on, declare a browser at your [project root](https://docs.langchain.com/langsmith/python/managed-deep-agents-project-structure):

```
# browser.py
from managed_deepagents import define_browser
browser = define_browser(scope="thread", idle_ttl_seconds=600)
```
Managed Deep Agents runs the browser in its own [LangSmith sandbox](https://docs.langchain.com/langsmith/sandboxes) and adds the following [browser tools](https://docs.langchain.com/langsmith/python/managed-deep-agents-browser-use#browser-tools) to your agent automatically:

ToolWhat it does`browser_snapshot`Reads the page's accessibility tree and assigns element IDs`browser_run`Navigates and interacts, through JavaScript or batches of actions against snapshot IDs`browser_screenshot`Captures the page, optionally saving the image to the sandbox`browser_handoff`Gives a person control of the window

By default, each conversation gets its own browser, which is reused across turns and starts with no cookies or logins from other threads. With [`scope="user"`](https://docs.langchain.com/langsmith/python/managed-deep-agents-browser-use#choose-a-scope), each [signed-in person](https://docs.langchain.com/langsmith/python/managed-deep-agents-identity) gets one browser whose logins carry across all of their conversations, so the agent only asks them to sign in to a site once.

### Hand the browser to a person

Most browser agents stop at a login wall or a two-factor prompt. With [`browser_handoff`](https://docs.langchain.com/langsmith/python/managed-deep-agents-browser-use#hand-off-the-browser), the agent hands the window to you instead, so your password never ends up in the model's messages or traces.

The agent calls `browser_handoff(reason)` and explains why it needs you. Your agent channels — like Slack and Studio — will show have a link to allow users of your agents to hand off browser control natively, which opens a link to interact with the remote browser and a **Hand back** button . Custom clients can render the handoff artifact themselves. Once you sign in and hand control back, the agent checks the page to see whether the login worked and continues. Handoff returns immediately, so the agent can do other work while it waits. Handoff links expire after 15 minutes.

![](assets/diagram-9bb5ecbab33e.png)

Handoffs let your agent use a webpage for you, without seeing your credentials.

The [browser use docs](https://docs.langchain.com/langsmith/python/managed-deep-agents-browser-use) cover scopes, per-run configuration, and the full handoff flow.

## Run subagents in the background

[Subagents](https://docs.langchain.com/oss/python/deepagents/subagents) let an agent hand a focused task to a helper with its own context window. Previously in Managed Deep Agents, subagents ran inline, so the agent called one and waited for its answer. This works for a quick lookup, and still is a supported pattern. But for a long task backed by a synchronous agent, or where multiple agents need to be fanned out, the person talking to the agent waits too, and additional interactions with the orchestrator agent are blocked.

With Managed Deep Agents v0.10, you can run subagents asynchronously, without blocking the orchestrator agent, by adding `async` to a subagent definition:

```
from managed_deepagents import define_deep_agent
agent = define_deep_agent(
    name="research-assistant",
    model="openai:gpt-5.5",
    subagents=[
        {
            "name": "researcher",
            "description": "Researches a topic in the background.",
            "system_prompt": "Research the topic and report your findings.",
            "async": True,
        },
    ],
)
```
The agent starts an async subagent on its own thread and can keep talking to the user while the subagent(s) run(s). The orchestrator agent starts, checks on, updates, and cancels these tasks with a set of [async task tools](https://docs.langchain.com/langsmith/python/managed-deep-agents-subagents#how-async-subagents-run). Every served agent also gets a background copy of a [general-purpose subagent](https://docs.langchain.com/oss/python/deepagents/subagents#the-general-purpose-subagent), so it can choose at runtime whether to run delegated work in the foreground or the background.

Async subagents [notify the agent](https://docs.langchain.com/langsmith/python/managed-deep-agents-subagents#get-notified-when-a-subagent-finishes) on their own, so the agent never sits in a polling loop. What happens next depends on what the agent is doing when a subagent finishes:

When the subagent finishes, the agent isWhat happensBusyIt checks unfinished tasks before each model call, so the result arrives mid-runIdleThe subagent wakes it with a new runPaused on an approvalThe approval is left alone; the result arrives once the user answers

An async subagent shares the agent's [sandbox](https://docs.langchain.com/langsmith/python/managed-deep-agents-sandboxes), so it picks up the files the agent already wrote. Its run appears under the agent's run in [LangSmith](https://docs.langchain.com/langsmith/observability). In Slack, its [approvals](https://docs.langchain.com/langsmith/python/managed-deep-agents-subagents#answer-approvals-in-channels) post back to the original thread, and answering them there resumes the subagent.

## Also in v0.10

- **`runtime.thread`**: every thread now carries a record of where it started, such as a Slack thread, and who is on it. Tools can read `runtime.thread` on any run, which is how a Studio resume, a scheduled run, and an async subagent's wake-up run all reply in the original Slack thread.The [runtime docs](https://docs.langchain.com/langsmith/python/managed-deep-agents-runtime) cover the managed fields tools can read.
- **Stopping runs**: Slack threads get a native Stop button that cancels the thread's runs, and Studio can cancel runs on Slack threads.
- **Slack channel handlers (experimental)**: `onEvent` decides what each Slack event starts, including mentions, direct messages, other messages, and button clicks. `status` sets what Slack shows while a run works, and `reply: "none"`leaves replying to the agent. See the [Slack channel docs](https://docs.langchain.com/langsmith/python/managed-deep-agents-channels-slack).
- **Bring your own Slack app**: `mda deploy` asks which app a new Slack channel should use. You can pick one that LangSmith creates and manages, create a new one from a manifest, or use one you already have.
- **`resolveUser`**: when someone messages your agent in a channel like Slack, Managed Deep Agents verifies who sent it. `resolveUser` maps that sender to the matching user in your own system, so your tools can look up that person's account and permissions. Write it as a function or point it at an endpoint we call, and set it on the [identity](https://docs.langchain.com/langsmith/python/managed-deep-agents-identity)declaration to cover every channel or on a single channel.
- **Breaking changes**: `connectors.mcp()` and the `connectors/` layout have been removed. MCP servers must be declared in `tools/mcp.ts` or `tools/mcp.py`. The sandbox `scope` option is also removed. For `LANGSMITH_*`variables, shell environment variables now win over a project `.env`.

## Getting started

Managed Deep Agents v0.10 is available in public beta today. Start from the [docs](https://docs.langchain.com/langsmith/python/managed-deep-agents-overview), or create a new project:

```
uvx --from managed-deepagents mda init my-agent
cd my-agent
uv run mda deploy
```
As always, we can't wait for you to tell us what you build.

*ICYMI: v0.9 added agent-created schedules, per-run configuration, and Slack reactions. Read the v0.9 post.*
