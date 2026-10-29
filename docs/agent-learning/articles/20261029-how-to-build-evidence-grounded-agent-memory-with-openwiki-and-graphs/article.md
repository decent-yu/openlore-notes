# How to build evidence-grounded agent memory with OpenWiki and Graphs

> **来源**: [https://www.langchain.com/blog/building-agent-memory-openwiki-graphs](https://www.langchain.com/blog/building-agent-memory-openwiki-graphs)  
> **作者**: 未知 | **发布时间**: 2026-10-28T17:24:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: OpenWiki, 智能体记忆, 跨文档问答, GraphRAG, 证据溯源, 知识更新, 检索评估

---

## AI 导读

文章讨论智能体在多份文件中寻找适用规则时，如何避免每次重新梳理引用、修订和覆盖关系。作者先用 OpenWiki 将原始资料整理成互相链接、带原文引用的知识页面，供智能体反复使用。页面中的主张绑定来源行及其哈希值，也就是用于识别内容变化的摘要；来源变化后，仅将受影响的主张交给模型复核。

实验使用111份合成保险文档和972条事实记录，设置80道简单查询与50道跨文档问题，每题运行三次。在模型、指令和问题相同的条件下，比较关键词搜索与增加知识页面后的表现。跨文档问题的准确度从0.10升至0.52，每个正确答案的成本从约0.55美元降至0.19美元；简单查询的准确度基本不变，耗时从9秒增至12秒。

知识页面仍存在引用不稳定的问题。作者进一步构建证据图，即把主张、页面和原文段落用注明关系类型的连接组织起来，让智能体直接沿引用或替代关系寻找依据。工具先定位起点，再沿关系展开；遇到分支仍由智能体选择。相较仅使用知识页面，作者报告准确度相对提高27%、引用一致性翻倍、中位回答时间下降40%、每个正确答案的成本下降58%。引用一致性衡量多次回答是否引用相同证据，并不等于答案正确。

这些结果支持将该方法用于反复发生的跨文档推理。回答成本未计入知识页面的初次构建和更新，图中的引用也可能继承模型错误。实验基于合成资料，缺少真实业务验证及完整复现信息，不能据此推定其他领域或更强检索基线下也有相同收益。

---

## 原文整理

## Key Takeaways

- **A compiled wiki gives agents domain context that retrieval alone must reconstruct.** On questions spanning multiple sources, an agent using OpenWiki as a memory layer was 5x more accurate than one using grep tools alone.
- **Adding an evidence graph helps agents follow the wiki back to supporting evidence.** We turned citations and document references into explicit graph edges agents could follow. Compared with the wiki alone, this doubled how consistently the agent cited the same evidence, improved accuracy by 27%, and cut cost per correct answer by 58%.
- **Save compiled memory for repeated cross-source reasoning.** The wiki captures domain knowledge and complex relationships. Adding an evidence graph improves accuracy and retrieval consistency. For simple lookups, however, the agent without memory was just as accurate and faster.

Agents working across an organization’s policies, manuals, wikis, tickets, and code must navigate knowledge scattered across systems and teams. Finding a relevant passage is only the start. They also need to determine which rules apply, what has changed, and how one source affects another.

A persistent memory layer preserves knowledge and relationships for agents to reuse across tasks. Without it, agents often have to reconstruct that understanding on every question, making their work expensive and error-prone. This is especially challenging in banking and insurance, where policies and guidelines frequently amend or supersede one another.

We tested [OpenWiki](https://docs.langchain.com/oss/openwiki/overview), our open-source wiki compiler, as this memory layer. On questions requiring connections across sources, the wiki made the agent roughly 5x more accurate than search alone. Adding an evidence graph that connects wiki statements to supporting passages and links related sources improved accuracy a further 27% and doubled citation consistency.

## Why a compiled wiki?

RAG pipelines often use semantic or hybrid search to retrieve isolated chunks, leaving agents to piece together how those chunks relate. We’ve explored improving retrieval with [agentic RAG](https://www.langchain.com/blog/agentic-rag-with-langgraph) and [graph-based metadata filtering](https://www.langchain.com/blog/graph-based-metadata-filtering-for-improving-vector-search-in-rag-applications), but finding relevant passages is only part of the work. Cross-source questions require tracing references, resolving conflicting rules, and understanding dependencies. Memory preserves that understanding so agents don’t have to rebuild it on every question.

A compiled wiki provides that memory layer by synthesizing source material into linked pages that explain key concepts and their relationships. The wiki is updated as sources change, giving agents context they can reuse.

[OpenWiki](https://docs.langchain.com/oss/openwiki/overview), our open-source CLI built on [Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview), creates these wikis with citations back to the sources. We originally built it so coding agents wouldn’t have to rediscover architecture and code behavior on every task.

![](assets/diagram-f8f4dcd8afc0.png)

Every OpenWiki statement rests on a **claim**, an assertion linked to supporting source lines and their hash. When those lines change, the hash changes, allowing OpenWiki to flag affected claims without a model call, saving on token costs and making staleness detection more deterministic. Only those claims go to a model for confirmation, revision, or retraction, allowing the memory layer to self-correct as its sources change. We explain this in [Building self-correcting memory in OpenWiki](https://www.langchain.com/blog/self-correcting-memory-openwiki).

## Building the test document library

We built a synthetic insurance document library of **111 long-form documents**. We chose insurance because policies and endorsements frequently **reference, amend, and override** one another; this is exactly the kind of cross-document dependency that makes retrieval hard.

To score the benchmark, we created a **ledger of 972 facts** and controlled where each fact should appear, how documents referenced each other, and what changed between versions. We generated the surrounding prose with an LLM and ran automated checks to ensure the required facts were present.

Finally, we made the corpus intentionally messy, mixing in **outdated versions, contradictions, indirect reference chains, varied wording, and very long documents** to better match what agents face in real enterprise libraries.

## How we configured our experiment

We evaluated the same retrieval agent under two conditions: **Search** (keyword search over the original documents) and **Wiki** (the same agent with access to an OpenWiki-compiled, source-linked wiki). The underlying model, instructions, documents, and questions were held constant — only the agent’s access to compiled memory changed.

We tested **80 simple lookup questions** and **50 connected questions** that require following dependencies across documents (e.g., identifying what depends on a provision or is affected when it changes). We asked each question **three times** to measure run-to-run variation.

**Metrics (what “better” means):**

- **Accuracy (0–1):** a grader checked answers and quoted evidence against a checklist from the 972-fact ledger (rewarding required facts and penalizing contradictions).
- **Consistency (0–1):** overlap of cited evidence across the three runs (average pairwise Jaccard similarity of citation sets). This measures whether the agent reliably grounds its answers in the *same* sources, not whether the answers are correct.
- **Cost + latency:** model spend (USD) and wall-clock time per question.

## Where the wiki helped, and where it struggled

![](assets/diagram-5d94d9ea6d63.png)

OpenWiki compiled the 111-document library into a wiki roughly one-tenth its original size for $4.15. Adding 17 documents later cost about $0.09 each. **The analysis below excludes compilation and update costs, reporting only retrieval and answering costs.**

We used simple lookups as a control. These single-passage questions require little cross-document insurance context, so we didn’t expect the wiki to help much. Results were similar for search and wiki, respectively (0.84 vs 0.83 accuracy; 0.72 vs 0.74 citation overlap; $0.020 vs $0.021 per correct answer), while latency increased from 9 to 12 seconds.

Connected questions required reasoning across sources using domain-specific relationships. Here, the wiki improved accuracy roughly 5x, from 0.10 with search alone to 0.52. Its compiled context helped the agent apply domain knowledge and find relevant relationships with fewer tool calls, cutting cost per correct answer from about $0.55 to $0.19.

Despite these accuracy and cost gains, the wiki agent’s evidence retrieval remained inconsistent. Citation overlap dropped from 0.74 on simple lookups to 0.25 on connected questions. Citation stability doesn’t measure correctness, but changing evidence can change an answer, especially when sources conflict. For critical business functions, teams need answers they can reproduce and audit, not just better average accuracy.

The wiki helped the agent apply domain knowledge across sources. The remaining challenge was making the path to supporting evidence more repeatable.

## Adding an evidence graph

To make retrieval more consistent, we added a lightweight **evidence graph: a map that turns wiki citations into explicit, typed edges between claims, wiki pages, paragraphs, and topics.** OpenWiki citations are text references in Markdown files. The graph makes them durable by materializing them as typed edges. That gives the agent a stable path to follow instead of rediscovering the same links with search.

![](assets/diagram-96713f8abe70.png)

We built this graph in two steps:

1. **Store every paragraph as a new primitive.** A deterministic process splits each document into paragraphs and assigns each a unique identifier. These paragraphs become atomic units of information, linking claims to exact supporting passages rather than entire documents.
2. **Connect the passages with typed edges.** We linked paragraphs, claims, and wiki pages with typed edges: claims `cite` paragraphs, newer documents `supersede` older ones, and sections `refers_to` other documents. Each edge identifies the relationship, not just the connection. We built edges two ways. Some are extracted deterministically from document references. Because wiki claims already carry text citations, we can materialize typed edges like `refers_to` and `supersedes` without another model call.

The graph took under a second to build and resolved every explicit cross-reference in the test library, so the agent could follow links instead of rediscovering connections on each question.

The graph’s job is evidence navigation, not modeling everything the organization knows. The wiki still provides the synthesis. Unlike an extraction-heavy knowledge graph pipeline, this layer reuses existing references and citations rather than extracting and reconciling entities with a model. Construction requires no additional model calls, though edges derived from the wiki still depend on the accuracy of its model-generated citations.

![](assets/diagram-0e905ecacdc9.png)

Two tools let the agent navigate the map: `locate` finds a starting point that matches the question, and `expand` follows its connections to supporting passages and related documents. Where there’s only one path, `expand` follows it automatically; where the path branches, the agent chooses where to go next. This pattern of locating relevant nodes and gathering connected context also appears in GraphRAG systems such as [Microsoft’s GraphRAG](https://microsoft.github.io/graphrag/).

```
@tool(parse_docstring=True)
async def locate(terms: str, within: str | None = None) -> dict:
    """Find where to start: match the question's words against document titles,
    section headings, wiki pages, and claim statements."""
    ...
@tool(parse_docstring=True)
async def expand(node: str, edges: list[str] | None = None, hops: int = 1) -> dict:
    """Open one node and list its typed edges (cites, refers_to, supersedes,
    amends, attaches_to, and their inverses). With hops > 1 and one edge type,
    walk that edge while there is a single path, and stop at a branch."""
    ...
```
## Better accuracy, lower cost, faster answers

![](assets/diagram-89f2c95cd73d.png)

Adding explicit evidence paths improved every measure over the wiki alone. Accuracy rose 27%, citation consistency doubled, median answer time fell 40% , and cost per correct answer fell 58%. The evidence graph complemented the wiki’s context with more directed retrieval, improving both accuracy and consistency without making the agent fully deterministic.

## When is adding memory worth the cost?

The strongest case for a compiled wiki is repeated cross-source reasoning. In our experiment, the wiki helped the agent apply domain relationships rather than reconstruct them through search. But the memory layer alone didn’t make evidence retrieval consistent. Adding explicit graph edges improved citation consistency and accuracy while reducing retrieval work.

The wiki provided a strong foundation for a lightweight evidence graph, derived deterministically from its existing citations and document references. This let us add explicit evidence paths without additional model calls or the need to model everything the organization knows.

Whether a wiki is worth the investment depends on reuse. Our reported answer costs exclude compilation, and adding documents cost about $0.09 each. Ingestion and ongoing maintenance may outweigh retrieval savings for simple lookups, where search alone was just as accurate and faster, especially at enterprise scale.

The tradeoff changes when multiple agents repeatedly answer questions spanning the same disconnected sources. A shared wiki preserves the relationships between those sources, while graph edges guide agents to supporting evidence. In our tests, that combination produced more accurate answers at a lower cost per correct answer. With enough reuse, savings from avoiding repeated context reconstruction can offset the cost of building and maintaining memory.

## What's next

Next, we want to build wikis more dynamically, compiling and retaining only the context agents need for cross-source questions rather than entire repositories.

We also want to explore claims and typed edges as a foundation for other kinds of agent memory. Linking knowledge to evidence and detecting when it changes could help ground semantic memory (facts), episodic memory (what happened), and procedural memory (how to do things).

Want to add wikis into your agents? Check out our OpenWiki resources:

## Acknowledgments

Thank you Colin Francis and Nick Hollon for their thoughts around OpenWiki and evaluations!
