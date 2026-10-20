# From 2,500 requests to 24: Rebuilding star history without the stargazers

> **来源**: [https://github.blog/engineering/architecture-optimization/from-2500-requests-to-24-rebuilding-star-history-without-the-stargazers](https://github.blog/engineering/architecture-optimization/from-2500-requests-to-24-rebuilding-star-history-without-the-stargazers)  
> **作者**: Taylor Cuilty; Sarita Iyer | **发布时间**: 2026-10-14T19:11:33+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: API设计, 数据库查询优化, 时间序列聚合, 分页策略, 隐私保护, 缓存, 性能分析

---

## AI 导读

文章解决利用 stargazer 列表重建仓库星标曲线时的身份暴露、调用次数过多和数据库重复处理问题。旧接口每页最多返回100人，且分页实现存在每次请求都统计全部星标的缺陷；正文称限制访问前一周约有3.7万次慢计数查询，平均接近3秒，逾千次被数据库保护机制终止。作者先考虑仅返回时间戳，但这仍需遍历大量记录；随后说明先全量分桶、排序再执行 LIMIT/OFFSET，只会减少返回结果，不能减少此前的计算。最终方案按日历周分页，将页码与页大小转换为有索引的创建时间范围，先过滤再按日聚合，每次最多覆盖30周。以约25.1万星标的 react/react 为例，完整读取从约2511次请求降至24次，相比每页重新全量聚合的方案，总扫描量从约600万行降至约25.1万行。排查还促使网页改用缓存星标总数，作者称相关慢查询降至零。结论是应围绕趋势需求设计接口，并在聚合前缩小输入集合。局限包括：时间窗口不约束窗口内记录量，星标异常密集的周仍可能昂贵；底层只保留当前星标，取消星标会删除记录，因此曲线不能视为完整的历史事件或历史净星标总量记录。材料未给出完整查询、执行计划、新接口延迟测试或可见图表，无法进一步复核性能收益。

---

## 原文整理

[Get repository star history](https://docs.github.com/rest/activity/starring#get-repository-star-history), then share feedback in the [GitHub Community discussion](https://github.com/orgs/community/discussions/206104).

### [Taylor Cuilty](https://github.blog/author/taylorcuilty/)

Senior Software Engineer

Our new repo star history API now provides trends without returning individual identities or requiring unbounded database work. These are the engineering tradeoffs we made.

Maintainers look at star history to answer a practical question: did a launch, release, or event change a project’s trajectory? To reconstruct that curve from the GitHub API, tools had to request pages of stargazer records (each person who currently stars the repository and when they starred it), keep the timestamps, and throw the identities away.

Over time, that design created problems at both ends. Identities were available for bulk collection, so bad actors started scraping public stargazer lists for spam and phishing campaigns. For highly starred repositories, the endpoint could also make the database process many thousands of entries to return a single page.

The fix seemed clear: return counts instead of people. But returning less data didn’t actually make the query any cheaper.

Now maintainers have a direct way to get the curve: we built an endpoint that returns up to 30 weeks of daily star counts in a single request, with no identities attached. Keeping that cheap to serve meant reducing how much data the database processed, not just how much it returned, and the endpoint does that by **filtering stars by time before it counts anything**.

We first addressed the immediate risks to developers and to our database by [restricting stargazer and watcher lists to repository administrators and collaborators](https://github.blog/changelog/2026-06-30-upcoming-access-restrictions-to-public-api-endpoints-and-ui-views/). After the access change, callers without the required permission could receive an empty response or a `403 Forbidden` response.

That change also cut off tools that had used the old response to calculate star history, often without displaying or otherwise using the usernames it returned. So we talked with maintainers and tool builders to understand what they still needed. They used star history as an approximate signal of interest: to compare activity before and after a release, or to see whether an announcement reached new developers. What they cared about was the curve.

The old API response had made identity look like part of that contract because every timestamp arrived attached to a user. In practice, tools such as Star History needed counts and dates, not usernames or other user data. Once identity was no longer part of the requirement, we could design an aggregate endpoint for the actual job: retrieving changes in star counts over time at a useful resolution.

When the API change disrupted Star History, users rallied around the project. GitHub’s Community team listened and connected us with product engineering, who dug into our use case and worked alongside us to design a purpose-built endpoint. A difficult situation became a constructive collaboration between GitHub, Star History, and the broader community—ultimately leading to a better solution.

Tianzhou Chen, star-history.com maintainer and Bytebase co-founder

Once we knew what maintainers needed, the next step was understanding why the original design couldn’t provide it cheaply. The original `GET /repos/{owner}/{repo}/stargazers` endpoint was built to answer a simple question: who starred this repository? It returned a paginated list of people, and if the request specified a custom media type, each entry also included when the star was created.

The star model was built for the same job. It records the current set of people who have starred a repository, which keeps showing a star count or a list of stargazers simple. It was not designed as a historical event log: when someone removes a star, that record is removed. When tools began using the list as a time series, reconstructing a growth curve meant retrieving individual identities and processing every current star. Because each page held at most 100 people, the work grew with the repository’s popularity: building the full curve for `react/react`, with about 251,000 stars, took about 2,511 requests, so tools often ran into rate limits or sampled pages instead of reading them all.

Serving a page required the database to query the repository’s current stars. Due to a bug in the pagination implementation, every request also counted all of those stars to work out how many pages existed, even though the caller received only one page of up to 100 people. On a highly starred repository, that meant counting many thousands of entries on every request. In the week before we restricted access, nearly 37,000 of these count queries ran slowly enough to be logged, averaging nearly 3 seconds each, and database safeguards killed more than 1,000 of them, which contributed to request timeouts.

The chart below shows what all that counting cost. After we restricted access, the database served about 60% fewer stargazer requests, and the total time it spent on the count query fell by about 80%.

Removing identities handled the privacy requirement. The harder part was keeping each request from scanning the repository’s entire current star set.

Our first spike kept one record per star and returned the `starred_at` value without the stargazer’s identity for callers who lacked collaborator access. At first, this looked like the quickest path forward, because it directly served the request we had heard from tool builders. It also closed the identity-harvesting path with a relatively small change to the existing serialization and authorization flow.

This approach addressed identity exposure and would have allowed existing tools to keep working. But API consumers would still have to paginate through and process potentially many thousands of individual records to build an aggregate history. It also tied the new use case to an endpoint and pagination model designed for listing people rather than retrieving trends.

Aggregation seemed like the obvious next step. Instead of returning one row per star, the query could group stars into daily or weekly buckets, sort those buckets, and paginate the much smaller result.

The problem was where pagination happened. In a straightforward SQL query, the database first finds all stars for the repository, groups them into buckets, and sorts those buckets. Only then does it apply `LIMIT`, which controls how many buckets are returned, and `OFFSET`, which determines where the page begins:

`all matching stars -> bucket every row by day or week -> sort every bucket -> OFFSET/LIMIT -> response`
For a popular repository, an anonymous request could still scan and aggregate many thousands of index entries to return a handful of buckets. Returning fewer buckets would shrink the JSON response, but it would not reduce the expensive grouping and sorting the database had already performed. For `react/react`, reading the full history that way would have meant examining roughly 6 million rows, because every one of the 24 requests would have reprocessed all of its stars. To reduce the query cost, we had to narrow the rows before aggregation began.

The design we shipped paginates star history by calendar week. For each request, it translates `page` and `per_page` into an indexed `created_at` range, filters the stars to those weeks, and then groups the matching records into daily counts. A response includes up to 30 weeks, and increasing `page` moves backward through the repository’s history.

`indexed date range -> group rows in that range -> daily buckets -> response`
With the date range applied first, each request processes the weeks being returned instead of every current star in the repository. For tools, that means one request now covers up to 30 weeks of history, no matter how many stars the repository has. For `react/react`, that’s about 24 requests instead of about 2,511, and the database examines each star about once (roughly 251,000 rows) across the full history. This fits the primary use case because maintainers usually inspect growth across weeks or around a known release, launch, or event. Week-based ranges also align with other repository activity APIs: the response follows the same shape as the [weekly commit activity endpoint](https://docs.github.com/rest/metrics/statistics#get-the-weekly-commit-activity), `GET /repos/{owner}/{repo}/stats/commit_activity`.

The time-range design limits each request to a defined period, but it does not limit the number of stars within that range. A week with an unusually large number of stars can therefore still produce a slow or expensive query. The normal request path no longer queries every current star, but exceptionally dense weeks remain a scaling limitation.

Comparing the performance of the old and new API paths also surfaced other places where GitHub was counting stars on demand. We found the same pattern on the repository stargazers page on GitHub.com and updated it to use the repository’s cached star count. Slow queries from that page, which had numbered in the hundreds each week, dropped to zero. We recently shipped a similar fix to other stargazer API endpoints.

The chart below shows slow and killed stargazer count queries each week, by where they came from.

`LIMIT` and `OFFSET` reduced the number of results returned, but they did not reduce the grouping and sorting the database had already done. We had to look at the order in which the work happened.
Restricting the original endpoint addressed the immediate problem. Listening to maintainers gave us something **better** than what we started with: star history that’s easier to get, cheaper to serve, and safer for everyone who stars a project.

Put native GitHub features to work to review outside collaborator access and archive stale repositories.

We’re rebuilding GitHub’s Git infrastructure while GitHub keeps running, creating a foundation for agent-scale software development.

How we fully migrated github.com away from CSS-in-JS.
