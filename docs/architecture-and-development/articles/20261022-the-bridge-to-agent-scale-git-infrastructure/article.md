# The bridge to agent-scale Git infrastructure

> **来源**: [https://github.blog/engineering/architecture-optimization/the-bridge-to-agent-scale-git-infrastructure](https://github.blog/engineering/architecture-optimization/the-bridge-to-agent-scale-git-infrastructure)  
> **作者**: Eleanor White; Victoria Dye | **发布时间**: 2026-10-21T16:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: Git, Azure Blob Storage, 分布式存储, 副本一致性, 并发控制, 读写分离, 云迁移

---

## AI 导读

GitHub 迁移 Git 仓库时遇到一个问题：旧系统在每次写入期间锁住整个仓库，并等待所有健康副本响应或超时，因此跨地域约 25 毫秒的一次网络往返会拖慢整个仓库的写入队列，繁忙仓库尤其难以承受。文章提出将串行执行范围缩小到单个 Git 引用，即分支、标签等指向提交的名称；修改同一引用的写入仍需排队，修改不同引用的写入则可并行。作者报告，在模拟真实流量的内部测试中，不同引用的写入吞吐量提高了 3—4 倍，延迟也有所下降。这种收益依赖写入分散到多个引用，每次写入本身仍需承担跨地域延迟。

接着，系统通过一个不保留持久本地状态的“虚拟写副本”将数据写入 Azure Blob Storage 对象存储，并把对象存储持久化纳入写入确认条件。它还能合并不同引用的并发写入，摊薄网络往返开销。在此基础上，虚拟读副本可从对象存储加载缓存，无需加入协调写入的副本组，从而按读取需求独立扩容，并为读写使用不同的资源配置。

文章展示了逐步迁移和扩容的路径，但这些能力处于不同推进阶段，不能把内部测试结果视为全面上线后的效果。读取测试先触及压测工具上限，未给出可量化的容量结论；测试配置、故障处理和读副本一致性机制也未充分说明。写入仍需等待保存完整仓库的副本，更进一步的写入扩展方案留待后续介绍。

---

## 原文整理

### [Eleanor White](https://github.blog/author/nellwhite/)

Senior Software Engineer

We moved a copy of 99.99999% of GitHub’s repositories to Azure. Here’s how per-reference quorums will bring over the rest, and how object storage lets reads scale independently.

Over the past year, while Git traffic grew exponentially, 99.99999% of the Git repositories on GitHub began living in two places at once: our own data centers and Azure. The rest are among the busiest on the platform, and they’re held back by just one thing: a 25ms round trip.

In an [earlier blog post](https://github.blog/engineering/architecture-optimization/building-git-infrastructure-for-agent-scale-development/), we laid out the two principles behind our next-generation storage architecture:

In this post, we’ll look at the immediate bridge between our old design and our new one. We’ll talk about how a change to the way our quorum serializes writes gets the busiest repositories into Azure, and how that same step opens the door to replicas that keep no persistent state of their own.

The timing and shape of the move were driven by the agentic shift: rapidly growing traffic, a hard ceiling on the power capacity of our data centers and the surrounding geographical region, and physical supply chain constraints. At the beginning of our migration, the remaining capacity with all possible expansion gave us enough time to migrate safely, but not enough to do it slowly.

While we had prototyped new architectural approaches for scaling Git on cloud infrastructure, changing the storage architecture and the hosting platform at the same time, under that growth, carried too much risk. Given our scale and how critical the service is to the community, we needed to sequence the work carefully: move the data first, then evolve the architecture once it arrived.

Git operations are a Tier 0 service: they’re a core function of GitHub, and nearly everything in GitHub depends on the repository data. That meant the data, services, and every dependency had to move first. The first copy of a repository landed in Azure’s Central US region in August 2025. Today, most repositories have two of their five replicas in Azure, and Azure serves about half of Git traffic. For most of that time, Azure absorbed our growth while our own data centers held steady.

During the migration, traffic follows the data. Most repositories have five replicas to balance durability, availability, and scale. To minimize read latency, Spokes uses preferential routing to direct traffic to a replica in the same region and availability zone as the request. As replicas move to Azure, the share of the traffic that can be served from Azure grows with each one, until the last replica arrives and the old region stops receiving requests entirely. Every repository goes through this process:

So, what about the remaining 0.00001% of repositories?

In our current architecture, a repository hosted in more than one region has a lower write throughput ceiling. Spokes’ quorum only needs a majority to commit a write, but it waits for every healthy replica to respond (up to a timeout) before moving on, so slow replicas delay every write rather than falling behind. In practice, that means the slowest healthy replica sets the latency of every write, and because each write holds a repository-wide lock until it commits, it also sets the throughput ceiling. Once a replica lives in the target Azure region, it’s about 1,000 miles from our data centers, making each network round trip around 25ms, and a write requires multiple round trips. A repository already near its throughput ceiling can’t absorb that on every serialized write; it would fall further behind with every push.

This is where our first principle, minimizing coordination, comes into play. Specifically for this case: don’t serialize what doesn’t need to be. Git’s consistency requirements apply to each reference individually. Once the server accepts an update to a reference, no client should see an older value afterward, only that value or a newer one. That holds whether the update is a fast-forward, a force push, or a delete. But in most cases, updates to different references don’t depend on each other and can be applied independently.

So, we’re changing the scope of the quorum from the whole repository to individual references.

Holding a repository-wide lock for the entire reference update, round trips included, turns the quorum into a single, unordered request queue with one write in flight at a time. Each write in that queue takes as long as the slowest healthy replica. As the write rate grows, so does the queue, and each write waits longer than the one before it.

With per-reference serialization, only writes that contend on a shared reference must wait for each other. In practice, most writes to a repository touch different references—for example, agents and developers pushing to their own branches—so they no longer wait on each other. Each write still experiences the cross-region latency but writes to different references can be in flight at the same time. That removes the single bottleneck on the repository. We can visualize how queueing and timeouts build up by comparing the two approaches side by side under similar load:

These changes raise a repository’s write throughput ceiling roughly in proportion to how many references are being written at once, and get the last remaining repositories into Azure. In internal testing with realistic traffic, writes to different references consistently saw 3–4x higher throughput at lower latency.

Most repositories are heavily read-skewed, with somewhere between 100 and 1,000 reads for every write. Today, reads and writes hit the same replicas, so they compete for the same resources and can’t be scaled independently.

Decoupling storage from compute changes that. Once repository data is durably stored in Azure Blob Storage and updated with each incoming write, we can begin to decouple read traffic from the write quorum, and independently scale the compute needed for each. For example, reads would benefit from far more in-memory cache than we give them today, but that memory is also needed for compaction. Once they’re separated, each gets its own configuration.

The first step is making object storage a member of the quorum, but a special one that we call a virtual replica. The compute that represents it is our first fully ephemeral node. It serves as a bridge to object storage, streaming incoming writes and serving no reads. Because it’s a member of the quorum, a write isn’t acknowledged until it’s durable in object storage.

That adds a little latency to each write, but just like the cross-region round trip, per-reference serialization keeps it from becoming a throughput bottleneck. The virtual write replica also coalesces transactions on disjoint references into a single set of network I/Os to object storage, so a busy repository pays for one round trip across many concurrent writes instead of one per write. For now, the virtual write replica also handles compaction, and read replicas inherit the compacted state.

With the virtual write replica in place, we can scale out as many virtual read replicas as a repository needs, on demand. Each read replica warms its cache from Azure Blob Storage and starts taking read traffic without joining the quorum, using object storage as the source of truth and reconciling its local state against it in the background.

This step unlocks significant read scale. In internal testing, our distributed load testing harness routinely found ceilings in the current architecture. Against virtual read replicas, the harness hit its own limits first. It also comes with a few challenges that are solved in subsequent steps:

How can we push write throughput further?

The GitHub of today was built on the Spokes quorum, and over the years Spokes has become much more than a quorum coordinator. It’s also an intelligent routing and load balancing layer, a unified API that internal clients use to access and transact repository data at well over 100 billion operations per day, and a collection of background systems we call “sweepers” that handle repository placement, repair, and rebalancing between storage nodes.

The vast majority of what lives under the Spokes service umbrella applies just as well to the new architecture as it does to the existing one. But we also recognize that the commit protocol that has taken GitHub to a billion repositories won’t take it to the next billion and beyond.

We’re not done yet. With these steps in place, the next limit is write throughput. Every write still waits on replicas that each hold a full copy of the repository, yet very little of a write actually needs to be ordered. Git objects are immutable and named by their contents, so they can be stored without anyone agreeing on them. What’s left is small, and the hard part is ordering it as fast as possible without moving the bottleneck somewhere else. Designing for that on a whiteboard is one thing. Doing it underneath a billion repositories that developers and agents are writing to around the clock is another. Getting there forced us to rethink the second-order effects too, like where durable reference state lives and how compaction and garbage collection scale.

Agent traffic isn’t slowing down, and each of these steps gives repositories more headroom as it lands, without waiting for the end state.

**Coming next week:** At GitHub Universe and in the final post in this series, we’ll take a tour through that end state, what it does for write throughput, the second-order problems we had to solve to get there, and why there’s still a (small) consensus group somewhere in the stack.

*Thanks to Brian Celenza for their contributions to this post.*

Put native GitHub features to work to review outside collaborator access and archive stale repositories.

Our new repo star history API now provides trends without returning individual identities or requiring unbounded database work. These are the engineering tradeoffs we made.

We’re rebuilding GitHub’s Git infrastructure while GitHub keeps running, creating a foundation for agent-scale software development.
