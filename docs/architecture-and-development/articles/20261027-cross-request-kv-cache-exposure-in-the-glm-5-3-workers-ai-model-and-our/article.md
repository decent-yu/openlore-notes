# Cross-request KV cache exposure in the GLM-5.3 Workers AI model, and our remediation with SGLang

> **来源**: [https://blog.cloudflare.com/kv-cache-exposure-remediation](https://blog.cloudflare.com/kv-cache-exposure-remediation)  
> **作者**: 未知 | **发布时间**: 2026-10-26T22:01:23.889000+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: SGLang, KV Cache, Prefill/Decode 分离, 多租户隔离, GPU 内存安全, RDMA, 大模型推理, 安全事故复盘

---

## AI 导读

Cloudflare报告，GLM 5.3服务在2026年8月27日至9月9日期间，有155次响应包含其他用户此前请求的数据，涉及的输入均超过681,280个词元，即模型处理文本的基本单位。调查未发现蓄意利用的证据，但这不等于证明没有发生利用。

问题由配置不匹配和推理引擎SGLang的错误处理缺陷共同触发。服务允许约105万词元的上下文，但负责处理输入的显卡只能缓存681,280个词元。服务把处理输入和逐步生成回答分配给不同节点，两者通过KV缓存交接上下文；KV缓存是模型保存已处理上下文的键值数据。超出缓存容量的请求虽被内部标记为拒绝，却仍以单词元占位输入继续执行。生成节点预留了完整请求所需的内存页，只收到少量新缓存后便认为传输完成，随后读取未被覆盖的旧页面，导致跨用户数据泄露。

修复同时扩大缓存池，并让被拒绝的请求在分配内存和传输数据前就返回错误。作者称，两项改动经过研究者复现脚本和自有标记测试验证后，服务恢复。文章还说明了请求中止时过早释放内存、远程写入稍后到达等相关风险，并介绍延迟释放、限制传输批量和部署前直接核对容量等加固措施。

复盘的关键教训是：请求成功返回和性能正常，不能证明输出正确或用户数据隔离有效。更长期的方案是在每次读取缓存前核对页面是否属于当前请求，但现有原型曾因可靠性问题停用，也受模型架构限制；传输校验尚未覆盖所有模型，形式化验证仍是后续计划。正文提供了补丁线索与验证说明，但未展示原始日志、完整测试数据或影响范围的统计方法。

---

## 原文整理

# Cross-request KV cache exposure in the GLM-5.3 Workers AI model, and our remediation with SGLang

![](assets/diagram-ae76b1faa629.webp)

On September 8, 2026, we were alerted by a security researcher, Gadi Evron from Knostic, to a tenant isolation issue affecting the GLM-5.3 Workers AI model (`@cf/zai-org/glm-5.3`) starting on August 27, 2026. We have remediated the issue, and we restored the model on September 9. We have confirmed that no other model hosted on Workers AI was affected.

Between August 27 and September 9, a small number of customers using the GLM 5.3 Workers AI model received responses that contained data from other customers' requests made to the same model shortly before theirs. In total, 155 responses were affected, all to prompts longer than 681,280 tokens.

After excluding our own testing and the researcher's, the requests we identified are consistent in timing and volume with ordinary long-context use, not with someone repeatedly exploiting the bug. We found no evidence of deliberate exploitation.

Today we are publishing this report alongside a post from SGLang, the open-source inference engine that we use, as a coordinated disclosure. This post describes the bug that caused this issue, our response timeline, and the subsequent hardening we have conducted on our production SGLang deployments. We also detail our continued efforts with the upstream maintainers of SGLang to address this and similar bugs.

## What happened

GLM is a text generation model from [Z.ai](http://z.ai) that is built for agentic coding workflows; it is one of the many [<u>models</u>](https://developers.cloudflare.com/workers-ai/models/) available through Workers AI. Gadi Evron submitted a report on September 8 through our HackerOne program stating that very long prompts sent to `@cf/zai-org/glm-5.3` on Workers AI sometimes returned text that belonged to other users' sessions. Cloudflare reproduced the issue the next morning and took the model offline within 40 minutes of declaring an incident. We then deployed a corrected configuration and a patched inference engine, and restored service the same day.

This specific report of cross-request KV cache exposure was the result of two overlapping issues. The first was a misconfiguration of the open-source inference engine, SGLang, used by Cloudflare. We deployed GLM 5.3 with a 1,048,576-token context window, but the KV cache pool on our prefill GPUs only had room for 681,280 tokens. We were aware of this mismatch before launch. We expected the engine to return an error for any prompt it could not process, and our long-context tests returned successful responses, so we did not believe this to be an issue.

The second was a bug in how SGLang handled the resulting over-length requests in disaggregated prefill/decode mode. A prompt that did not fit the pool was internally marked as rejected and replaced with a one-token placeholder, but instead of aborting the request and returning a client error, prefill generated the KV cache for the placeholder prompt and transferred it to the decode node. The decode node received a "transfer complete" message and assumed that prefill overwrote the KV cache from previous requests with the full prompt for this request, when in fact it left the majority of the previous state intact. The decode node subsequently generated a response while attending over stale cache pages that still held data from earlier requests. As a result, responses to prompts longer than 681,280 tokens could contain data from other users' GLM 5.3 sessions.

## Immediate remediation

After [<u>disabling the model</u>](https://www.cloudflarestatus.com/incidents/jsvvrtk5hvym), we took the following actions to correct this vulnerability before restoring public access to it:

**To fix our configuration, we made the pool hold the full context.** We adjusted our GLM 5.3 configuration to match our GLM 5.2 configuration, which freed enough GPU memory that the prefill KV pool now holds about 2.04 million tokens, roughly twice the 1,048,576-token context length. A prompt that fits the context window now always fits the pool, so the rejection path is never reached for a valid request. This alone prevents the bug from being exercised.

**To fix the engine, we also made the rejection path safe.** Relying on pool size alone is fragile: any change to quantization, tensor parallelism, CUDA graph sizes, or a future model version changes the pool size in a way that's easy to miss. Furthermore, there may be other situations in which the prefill scheduler can reject a request and trigger the same bug. To prevent this, we also patched the engine so that a request carrying a pending rejection is retired at the disaggregation admission queue, with its original error, before any bootstrap, allocation, or KV transfer happens. We proposed this upstream as [<u>#38704</u>](https://github.com/sgl-project/sglang/pull/38704), which also had the decode node verify that the number of tokens transferred matches the number it reserved. The fix was already on SGLang `main` in [<u>#36988</u>](https://github.com/sgl-project/sglang/pull/36988), merged September 6 for a related failure path but not yet released. The maintainers also merged [<u>#38935</u>](https://github.com/sgl-project/sglang/pull/38935), which rejects these requests earlier, following our approach.

We validated both changes against the researcher's reproduction script and our own canary-marker tests before restoring the model.

## Background: how we serve GLM 5.3

Workers AI runs GLM 5.3 with SGLang in a disaggregated configuration: a pool of prefill GPUs processes the prompt and produces the model's KV cache for it, then hands that cache over remote direct memory access (RDMA), via [Mooncake](https://kvcache-ai.github.io/Mooncake/), to a pool of decode GPUs that generate the response. We have written about this architecture before in our posts on [<u>high-performance LLMs</u>](https://blog.cloudflare.com/high-performance-llms/) and [<u>smaller, faster, safer models</u>](https://blog.cloudflare.com/smaller-faster-safer-models/).

The KV cache in GPU memory is a shared pool of fixed-size pages. Each request is assigned pages when it is admitted and releases them when it finishes. Pages are not currently zeroed between requests; the next request that gets a page simply overwrites it. This approach is common among inference engines and is necessary to achieve the best performance, but it means that the inference engine is responsible for tracking ownership of these memory pages. KV cache page ownership and lifetime tracking has to deal with not just shared ownership within a node (KV cache offload and token generation both access the memory with different lifetimes), but also ownership that is technically shared with the remote prefill node (which holds a simultaneous mutable reference to the page).

Furthermore, the size of this pool is not a configuration value. SGLang sizes it at startup from whatever GPU memory remains after loading the model weights. The model's advertised context length is a separate configuration value. Nothing forces the two to agree, and it is our job to check that they do.

## Issue one: misconfiguration

On our GLM 5.3 prefill nodes with our configuration, the pool came to 681,280 tokens while the model's context length is 1,048,576 tokens. The selected KV pool size is printed by SGLang in logs during startup and exposed as a metric. We were aware of this size mismatch; however, during pre-launch testing we observed successful HTTP status codes and a streaming body with no error frames during benchmarking. We assumed that the engine was capable of gracefully handling the difference between the KV cache pool and requested context window via streaming or some other mechanism.

This was one critical gap in our testing: we only validated HTTP response codes and engine performance at long contexts, without validating model output and coherence. We now check KV pool size against context length directly, from the running engine, before any disaggregated deployment change ships, rather than inferring it from response codes.

## Issue two: SGLang bug

In v0.5.21, the SGLang team has now addressed this bug in how it was handling prompts longer than the KV pool size. Here’s what was happening before.

When a prompt arrived that was longer than the KV pool could hold, SGLang's scheduler rejected it. The rejection path (`Req.set_finish_with_abort`) recorded the error to be returned to the client and swapped the prompt for a one-token placeholder so that the request could drain through the normal batch machinery without allocating real memory. On a single-node deployment this worked: the placeholder ran, the batch finished, and the client received an error.

![](assets/diagram-ac2d4514e105.webp)

In disaggregated mode such as we operate, the admission queues on both prefill and decode checked whether a request was *finished*, not whether it was *rejected*. A rejected request reported itself as live. So:

1. **Bootstrap.** The router told the decode node the prompt length (8 pages in the diagram). Decode reserved 8 KV pages for the request.
2. **Prefill admission.** The prefill admission queue admitted any request where `req.finished()` was False. A rejected request stored its rejection in `to_finish`, not `finished_reason`, so `finished()` still returned False. The rejected request was admitted.
3. **Prefill forward pass.** Rejection had already replaced the prompt with a one-token placeholder. Prefill ran a trivial forward pass on that one token.
4. **KV transfer.** Prefill sent one page of KV to decode and signaled that the transfer was complete.
5. **Decode admission.** Decode's admission queue had the same `finished()` check, so it accepted the handoff.
6. **Decode attention.** Decode started generating and attention read all 8 reserved pages. Only page P0 had been written by this request. Pages P1 to P7 still held what earlier requests left in them: other users' prompts and partial generations.
7. **Response.** The client got HTTP 200 instead of HTTP 400. `prompt_tokens` matched the long prompt and the reply was coherent. It arrived faster than an 8-page prefill should take, and it contained content from other users' KV pages.

Every request over the threshold behaved this way. This bug had been reported to SGLang by another user as issue [<u>#30233</u>](https://github.com/sgl-project/sglang/issues/30233) on July 6, 2026, which Cloudflare noticed while investigating this incident. We had been flagging these types of issues with SGLang since v0.5.10 and this security incident sped up work that Cloudflare had already started in collaboration with the SGLang team. The patch for this issue was released in SGLang v0.5.20, with follow-ups in v0.5.21 and v0.5.22, and is available for all other operators dependent on SGLang to use. We highly recommend all operators update SGLang dependencies to v0.5.22 or later. For more details on specific PRs, read on below.

GLM 5.3 was the only model affected by this bug. Every other disaggregated deployment on Workers AI has KV pools larger than its context length on both prefill and decode, so no request could reach the rejection path. We have applied the engine fix and the hardening described below to every model we serve with SGLang, and our deployment checks require them for every new deployment.

## Timeline

All times UTC.

| When | What | 
| Aug 27, 2026 | GLM 5.3 launched on Workers AI with a prefill KV pool smaller than its context window. The exposure was present from launch. | 
| Sep 8, 2026 - 23:11 | Report received via HackerOne. | 
| Sep 9, 2026 - 08:00 | Cloudflare begins verifying the HackerOne report. | 
| Sep 9, 2026 - 13:22 | Security incident declared. Product Security reproduced the leak using two independent accounts within 25 minutes. | 
| Sep 9, 2026 - 14:00 | After confirming the scope of the issue, GLM 5.3 was set to private. Requests return HTTP 400. Status page updated. | 
| Sep 9, 2026 - 14:35 | Root cause identified as the prefill pool being smaller than the context length. Canary prefill node with a larger pool confirmed not susceptible. | 
| Sep 9, 2026 - 14:52 | All prefill nodes rolled to the corrected configuration. | 
| Sep 9, 2026 - 19:04 | Full deployment rebuilt with the corrected configuration plus a Cloudflare-authored engine patch that correctly returns errors for requests rejected by the prefill scheduler. Validated against the researcher's reproduction script. | 
| Sep 9, 2026 - 19:40 | Model restored to public. | 
| Sep 10, 2026 - 23:30 | Meeting with the SGLang maintainers (RadixArk CTO and core engineers) to agree on upstream fixes. | 
| Sep 11-19, 2026 | Additional hardening rolled to all production SGLang deployments (details below). | 
| Sep 16-Oct 12, 2026 | Upstream SGLang merges the admission fix, per-transfer checksums, abort-ACK preservation, and turns deferred KV release on by default. | 
| Oct 2, 2026 | SGLang v0.5.21 released. | 
| Oct 19, 2026 | SGLang v0.5.22 released. | 
| Oct 26, 2026 | Coordinated publication with the researcher and RadixArk. | 

## Ongoing hardening effort

This bug belongs to a class of KV cache memory-safety bugs that Cloudflare has been working to address for several months. In each case, a page is freed, reused, or read while something else still has a claim on it. We see two common causes.

The first is page lifetime within a single node. Several tasks on a decode node can hold the same page at once: token generation, offload to host memory, the KV transfer receiver, and the abort and error paths. If one of them frees the page while another is still using it, a new request can be given that page while it is still being read or written. Our earlier fixes for decode offload ([<u>#24226</u>](https://github.com/sgl-project/sglang/pull/24226), [<u>#30899</u>](https://github.com/sgl-project/sglang/pull/30899)) and double release on grammar errors ([<u>#30937</u>](https://github.com/sgl-project/sglang/pull/30937)) were of this kind.

The second is a race across nodes. When a request is aborted while prefill is still writing its KV cache to decode over RDMA, the writes already sent cannot be recalled. If decode frees those pages right away, those pages can be given to another request, and the late writes land in that request's KV cache. Cloudflare has contributed to the effort to fix this race through proposed fixes ([<u>#31357</u>](https://github.com/sgl-project/sglang/pull/31357), [<u>#38961</u>](https://github.com/sgl-project/sglang/pull/38961)) and a reproduction script ([<u>#32564</u>](https://github.com/sgl-project/sglang/pull/32564)). In both cases the result is corrupted output, or one user's data in another user's response, with no error raised. The bug in this incident was different: a rejected request went through the handoff every time, rather than by a timing accident. But it had the same result: decode reading pages this request never wrote. None of these covered the failure in this incident in which prefill rejected the request, and decode never learned of it.

Since the incident, we have:

- Extended deferred release to hold prefill's source pages as well, and to cover every failure type, not just aborts. We proposed it upstream as [<u>#38961</u>](https://github.com/sgl-project/sglang/pull/38961), which was superseded by [<u>#40645</u>](https://github.com/sgl-project/sglang/pull/40645), [<u>#41402</u>](https://github.com/sgl-project/sglang/pull/41402), and [<u>#41404</u>](https://github.com/sgl-project/sglang/pull/41404).
- Deployed bounded Mooncake transfer batches ([<u>#32758</u>](https://github.com/sgl-project/sglang/pull/32758), from the SGLang community), so no single transfer call can outlast the release timeout.
- Changed our load balancer so a client disconnect no longer aborts a request mid-transfer. It holds the prefill connection until prefill finishes and the decode connection until the first token, then closes both.
- Added a check to every disaggregated deployment change: KV pool size measured against context length, deferred release on both roles, transfer batch bounds inside the release timeout, and all required patches applied.

All of these changes were released to our disaggregated production deployments.

## Working with the SGLang team

We met with the SGLang maintainers at RadixArk on September 10, the day after the incident. We agreed that RadixArk would investigate the race conditions we had reported, upstream the integrity-checking idea, and build reproduction tests for this class of bug.

The following have landed on SGLang `main` to address this incident and related KV cache race conditions, and all but one are included in v0.5.22. Operators running disaggregated SGLang should update to v0.5.22 or backport these patches as needed.

| Category | Pull request | Release | What it does | 
| Incident | [#36988](https://github.com/sgl-project/sglang/pull/36988) | v0.5.20 | Retires aborted prefill requests before their result reaches decode. This stops the exposure in this incident. | 
| Incident | [#38935](https://github.com/sgl-project/sglang/pull/38935) | v0.5.21 | Rejects these requests at admission, before any bootstrap, allocation, or KV transfer. Follows the approach in Cloudflare's draft patch, [#38704](https://github.com/sgl-project/sglang/pull/38704). | 
| Race | [#39461](https://github.com/sgl-project/sglang/pull/39461), [#41612](https://github.com/sgl-project/sglang/pull/41612) | v0.5.21, v0.5.22 | `sgl-router` cancels both halves of a request on client disconnect or prefill failure. | 
| Race | [#37454](https://github.com/sgl-project/sglang/pull/37454), [#41023](https://github.com/sgl-project/sglang/pull/41023) | v0.5.20, v0.5.21 | Enables deferred KV release only on backends that support it, then turns it on by default. | 
| Race | [#40645](https://github.com/sgl-project/sglang/pull/40645), [#41402](https://github.com/sgl-project/sglang/pull/41402) | v0.5.21 | Fixes cases where the drain confirmation was lost, so releases happen on confirmation instead of waiting out the timeout. | 
| Race | [#41404](https://github.com/sgl-project/sglang/pull/41404) | v0.5.21 | Holds pages on every transfer failure, not only decode-initiated aborts. | 
| Race | [#41618](https://github.com/sgl-project/sglang/pull/41618) | v0.5.21 | Fixes a regression in the abort notification path. | 
| Race | [#43719](https://github.com/sgl-project/sglang/pull/43719) | v0.5.22 | A failed Mooncake send no longer reports its writes as drained. | 
| Race | [#37077](https://github.com/sgl-project/sglang/pull/37077) | v0.5.22 | Tags abort messages per request so a reused ID cannot consume a stale confirmation. Adds MORI support. | 
| Race | [#36612](https://github.com/sgl-project/sglang/pull/36612) | v0.5.20 | Shares the prefill-to-decode failure notification across Mooncake, NIXL, and MORI. | 
| Race | [#32758](https://github.com/sgl-project/sglang/pull/32758) | v0.5.20 | Optional cap on Mooncake transfer batch size, so one timed-out call cannot leave an unbounded number of writes in flight. | 
| Lifetime | [#30899](https://github.com/sgl-project/sglang/pull/30899) | v0.5.21 | Decode waits for an in-progress offload to host memory before retracting a request and freeing its pages. Authored by Cloudflare. | 
| Lifetime | [#43562](https://github.com/sgl-project/sglang/pull/43562) | v0.5.22 | Keeps tensor-parallel ranks in agreement when a host-memory cache restore fails. | 
| Observability | [#39500](https://github.com/sgl-project/sglang/pull/39500) | v0.5.21 | Adds `--disaggregation-enable-kv-checksum`: decode verifies a checksum of the transferred KV before generating. Cleanup in [#39729](https://github.com/sgl-project/sglang/pull/39729), merged and awaiting release. | 
| Observability | [#38596](https://github.com/sgl-project/sglang/pull/38596), [#40642](https://github.com/sgl-project/sglang/pull/40642) | v0.5.20, v0.5.21 | Fixes for SGLang's KV canary, which detects and logs potential corruption. | 
| Observability | [#41128](https://github.com/sgl-project/sglang/pull/41128) | v0.5.22 | Metrics for deferred release: count, duration, and whether each hold ended on drain confirmation or timeout. | 

These follow-ups are still works in progress:

- [<u>#39729</u>](https://github.com/sgl-project/sglang/pull/39729) Additional cleanup for `--disaggregation-enable-kv-checksum`. Merged, awaiting release.
- Zero KV pages on release, draft proposal: [<u>#46435</u>](https://github.com/sgl-project/sglang/pull/46435)
- Extend KV checksum validation from the prefill-to-decode handoff to every decode step (see Integrity by construction, below)
- Broader KV Canary and KV Checksum model and feature support

## What we recommend to other operators

If you run SGLang in disaggregated prefill/decode mode, we recommend the following:

1. **Measure your KV pool and compare it to your context length**, on every prefill and decode node, from the engine's startup log, after every change to weights, quantization, parallelism, or memory fraction. If the pool is smaller, lower the context length to fit. Do not rely on a proxy to enforce the limit, and do not assume the engine will reject what does not fit. This is the check we did not enforce.
2. **Upgrade to a build that includes [<u>#36988</u>](https://github.com/sgl-project/sglang/pull/36988) and [<u>#38935</u>](https://github.com/sgl-project/sglang/pull/38935) (v0.5.22 or later)**, so an over-length request is rejected before handoff regardless of pool size. This is the engine-side fix, and it is the backstop for the day your check in step one is wrong.
3. **Run with deferred KV release on both roles** (default after [<u>#41023</u>](https://github.com/sgl-project/sglang/pull/41023)), and bound your Mooncake transfer batches so a single timed-out call cannot leave an unbounded tail of writes.
4. **Turn on corruption detection where your model supports it.** `--disaggregation-enable-kv-checksum` and `--kv-canary` do not yet cover every model architecture. Check your logs to confirm requests are being verified. We are working with the SGLang team to extend both checks to more architectures, including GLM 5.3, GLM 5.3 Flash, and DeepSeek V4.
5. **Ensure your proxy is not generating unnecessary aborts.** Configure your load balancer to allow requests to proceed up to the first token, so that you do not trigger an abort that requires cross-node coordination.

## Longer term

### Integrity by construction

Configuration checks and lifecycle fixes reduce the number of ways a page can be misused. They do not make misuse impossible, and this incident is a reminder that the next bug will have a shape we have not seen. The durable answer is for the engine to detect, at read time, that a request is about to attend over a page it does not own, and fail that one request rather than return someone else's data.

We built a prototype of this over the summer and [<u>described it in August</u>](https://blog.cloudflare.com/smaller-faster-safer-models/). Every physical KV page carries a tag that changes when the page is reallocated, each request records the tags it expects, and the mapping is checked before decode reads the cache. In our tests it cost under 1% in throughput and tail latency. We tested it on GLM 5.2 in mid-August, but disabled it shortly afterwards due to reliability issues.

The prototype was also tied to specific attention backends and model architectures, so it could not be a general fix. We are working with the SGLang team to build a production version into the engine that works across every model, rather than maintaining Cloudflare-specific patches. The per-transfer checksum in [<u>#39500</u>](https://github.com/sgl-project/sglang/pull/39500) is a first step: it validates the handoff from prefill to decode. Page-level tags go further, because they check every read, including reads long after the transfer.

### Integrity by mathematical impossibility

RDMA copies data directly from one node’s memory to another’s. It’s vital for the two machines to agree on where the data goes, how large it is, and when it’s finished; any mismatch can cause the receiver to read the wrong data. This bug is an example of a consistency failure in a distributed system, in which the two machines had inconsistent views of the state of that system.

Elsewhere in Cloudflare, we have put effort into proving the safety of distributed systems using formal verification methods. [<u>Topaz</u>](https://blog.cloudflare.com/topaz-policy-engine-design/) enables conflict-free IP selection for our authoritative DNS servers. Models of [<u>Meerkat’s</u>](https://blog.cloudflare.com/meerkat-introduction/) QuePaxa implementation have demonstrated consistency bugs that were subsequently fixed.

We are now directing our experience in formal methods to KV cache transfer engines. This will start with modeling and proving the safety of an RDMA-based KV cache transfer protocol. The goal is to model our production SGLang’s distributed KV cache system using Mooncake, then track and fix any divergence from the safety properties we want to guarantee. The aim is to prove — as far as possible — that a bug like this cannot occur again. Realistically, we expect gaps in what the provers can tell us, but proving the rest of the system allows us to more closely review and monitor those gaps.

## Coordinated disclosure

We are publishing this report as part of a coordinated disclosure. We want to thank the researcher, Gadi Evron, who reported this through our HackerOne program with a clear, reproducible script and gave us time to fix it and to coordinate with the SGLang maintainers before publishing. We also want to thank the SGLang team at RadixArk, who treated this as their problem as well as ours and landed the upstream fixes with urgency.

Read more at [<u>the post from the SGLang team</u>](https://www.lmsys.org/blog/2026-10-26-cf-security).

If you find a security issue in Workers AI, please report it at [<u>hackerone.com/cloudflare</u>](https://hackerone.com/cloudflare).
