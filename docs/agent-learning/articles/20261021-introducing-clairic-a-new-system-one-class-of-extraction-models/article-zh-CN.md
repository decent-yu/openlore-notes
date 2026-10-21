# 介绍 Clairic：全新的 System One 抽取模型

> **来源**: [https://blog.cloudflare.com/clairic-extraction-model](https://blog.cloudflare.com/clairic-extraction-model)  
> **作者**: 未知 | **发布时间**: 2026-10-19T17:48:26.614000+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: 信息抽取, 结构化输出, 多模态模型, 不确定性估计, 时间序列预测, 模型级联, Workers AI

---

## AI 导读

文章介绍 Clairic，一种面向 System One extraction 的单次前向模型，试图解决传统 LLM 在重复结构化抽取中的速度、成本、输出格式和可追溯性问题。用户以 state 和 typed fields 定义要读取的字段；模型返回文本跨度、数字、日期、时长、列表、估计分布或未来序列，并附带 confidence、p_null、区间和字符偏移。文中用客服邮件演示订单号、日期、总额、延迟天数、损坏类型和维修费用的提取或计算，用收据图像展示视觉读取，用房产可比数据展示带区间估计，用 TSA 每日客流展示预测。文章还把 Clairic、Clef 和 GLM-5.3 组成提取—决策—生成级联，在其测试中端到端延迟为 19.3 秒、较全 LLM 流程少 69%，成本少 49%。局限包括长链算术和多行聚合较弱、估计依赖输入证据、区间校准并非保证、预测能力基础且不替代专用时间序列模型；当前仍是早期预览。文中未提供足以独立复现实验的完整数据、评测协议或外部验证。

---

## 原文整理

# 介绍 Clairic：全新的 System One 抽取模型

![](assets/diagram-2f088decc36c.webp)

今天，我们宣布推出一类全新的 System One 风格模型的预览版：抽取模型。我们的首款模型 Clairic 已在 Workers AI 上提供托管服务，模型标识为 `@cf/cloudflare/clairic.`我们计划很快开放模型权重。Clairic 能够从非结构化输入中抽取、计算、估计和预测数值及其他类型的值，相比传统大语言模型（LLM），准确率显著更高、延迟更低、成本也更低。

## 背景

一个月前，TypeSafe 宣布推出 System One 模型，在业界引起轰动。正如 [<u>TypeSafe 在其博客中所述</u>](https://typesafe.ai/blog/introducing-system-one-models-and-jev)，按照 Daniel Kahneman 在《*思考，快与慢*》中的定义，System One 模型应该快速而精准，与能够进行审慎推理的 System Two 模型（例如 LLM）形成对照。基于这一理念，TypeSafe 推出了 Jev：一种专门针对特定用途设计的模型，由于服务的领域较窄，定价也较为亲民。它在 LLM 表现不佳的地方表现出色，通过概率和类型明确的输出帮助用户做决策。Jev 的推出促使人们开始思考：除了 LLM，还能有哪些新类型的模型，以及如何将它们融入自己的工作流。

System One 模型的理念启发了我们，让我们开始思考还有哪些任务是 LLM 做起来效率不高的。如今，发往前沿 LLM 的大量结构化输出请求，都在反复处理形式相同的问题：*发票总额是多少？什么时候送达的？晚了多少天？维修要花多少钱？未来 24 小时的流量会怎样？*这些问题的答案都无法事先一一列出，而决策模型需要预先列出答案，才能为各个答案给出概率。这让我们想到了今天要介绍的这类模型：System One 抽取模型。

## 介绍 System One 抽取模型

到目前为止，所有 System One 模型都是决策模型：Jev 和 [<u>Clef</u>](https://blog.cloudflare.com/clef-decision-models/) 接收一个输入状态和一组问题，针对输入及列出的问题给出概率。[<u>Clairic</u>](http://developers.cloudflare.com/workers-ai/models/clairic/) 则是一个新类别中的首款模型：System One 抽取模型。它依然快速、类型明确、成本低，但回答的是**具体值**：这些值可以直接从输入中抽取，也可以根据输入计算，或预测输入尚未包含的情况。

大多数模型任务要求的输出分为三类：一个数值、从固定选项中做出的选择，或开放式文本。Clairic 处理第一类，返回数值及其 80% 区间，也能从输入中提取姓名、日期和列表等带有明确类型的值。Clef 处理第二类：在选项中做选择，并返回每个选项的概率。LLM 处理第三类。我们将 Clairic 和 Clef 称为 System One 模型，是因为它们只需一次处理就能给出答案，无需推理，也不生成 token。它们只对输入收费，通常不到一秒就能响应，而 LLM 通过推理完成同一步骤则需要几秒钟。

抽取模型可以返回：

- **抽取值：**从输入中复制的文本，例如姓名、订单号或描述。每个值都会以文本片段的形式返回，并附带字符偏移量和置信度分数。
- 其中还包括**计算值：**从输入中读取或计算得出的数值、日期、时间、日期时间、时长和列表。它们遵循各自类型的规则（例如，精确到美分的金额保留两位小数），并附带置信度分数。
- **估计值：**输入没有直接给出、但提供了相关依据的值。每个值都会以概率分布的形式返回，分布位于你声明的边界内，采用你指定的网格，并附带中位数和 80% 区间。
- **预测值：**序列未来的值，每个未来步都有一个估计值，并各自附带 80% 区间。

![](assets/diagram-7ad50982e156.webp)

由于它不生成 token，因此不会输出类型错误的答案，也不会以 LLM 同样的方式产生“幻觉”（当然，它仍可能犯错）。输出要么是你要求的类型的值，要么是在模型无法从输入中找到答案时返回的 null。它还会为输出附上置信度分数，帮助你判断是否可以信任模型，或是否需要再检查一次。

下面展示了抽取模型在 System One 和 System Two 模型体系中的位置：

|  | System One：抽取 | System One：决策 | System Two | 
| 模型示例 | Clairic | Jev、Clef | GPT、Claude、Gemini、DeepSeek | 
| 采样方式 | 单次并行处理 | 单次并行处理 | 每次生成一个 token | 
| 用途 | 预测值 | 做决策 | 写作和推理 | 
| 机器学习任务类型 | 回归与抽取 | 分类 | 生成 | 
| 思考模式 | 系统 1：快速，单次处理 | 系统 1：快速，单次处理 | 系统 2：缓慢，审慎思考 | 
| 输出 | 类型明确的值：文本片段、数值、日期、时间、列表、分布，并附带经过校准的概率 | 类型明确的决策：选项及经过校准的概率 | 自由形式的字符串 | 
| 输出空间 | 连续，但受类型和你定义的模式边界约束 | 你列出的有限选项集 | 任何内容 | 
| 回答的问题 | 金额是多少？什么时候发生？将会是多少？ | 属于哪一类？是否合法？我们应该采取什么行动？ | 为什么会发生？我们应该如何处理？你能修复它吗？ | 
| 输出是否始终有效？ | 是：每个答案都是其声明类型的值（或 null），且位于你设定的边界内 | 是：答案始终是你提供的选项之一 | 否：字符串必须经过解析和检查 | 

这些模型可以协同工作，帮助你更好地解析现实世界的数据、根据输入做决策，再采取行动。抽取模型将原始输入转换为类型明确的值，决策模型将这些值转换为决策，LLM 再将决策转换为文字。

## 请求如何运作

Clairic 的设计与 LLM 不同。它与决策模型类似，只通过一次前向传播处理输入 token，因此不收取输出 token 的费用。对于某些用途，它比 LLM 快得多、便宜得多，也准确得多。

一个请求包含 `state`（要读取的文本，以及最多四张可选图片）和一组以映射形式组织、类型明确的 `fields`。你为每个字段命名，指定类型，并可选择用自然语言提供 `instructions`，随后就能通过一次并行处理，为每个字段得到一个类型明确的答案。整个过程没有推理，也没有输出 token，所有计算都在一次前向传播中完成，因此既快又便宜。

| 类型 | 返回内容 | 
| `text` | 输入中的某个具体文本片段，附带偏移量 | 
| `number` | 读取或计算得出的数值，遵循 `integer` 和 `decimals` 设置 | 
| `date`, `time`, `datetime` | 以 ISO 格式表示的日历日期和有效时间 | 
| `duration` | 以你选择的单位表示的一段时长 | 
| `list` | 按输入顺序排列的匹配文本、数值或日期 | 
| `estimate` | 位于你指定的 `[min, max]` 范围内、采用 `step` 网格的分布，可使用线性或对数尺度。 | 
| `forecast` | 序列每个未来步对应的一个估计值 | 

每个标量答案都附带 `p_null`（输入中不存在该值的概率）和 `confidence`。每个 `estimate` 答案都附带中位数和 80% 区间（`p10` 到 `p90`），也可以按需返回全部 99 个百分位数。字段数量没有限制：如果一次处理无法同时容纳输入和所有字段，服务器就会将字段拆分成多次并行处理，每次都能看到完整输入，最后合并答案。

下面通过一些示例说明模型的功能。当然，要在工作流中使用这些功能，需要提供适当的输入。

## 示例：抽取值

抽取可能是如今交给 LLM 最常见的任务：“以 JSON 格式返回发票号码、供应商和到期日。”大多数时候它都能完成，但出错的情况很难发现。输出可能不是有效的 JSON，生成过程中可能把某个数字读错，而返回一段摘录文本后，也无法追溯它在原文中的位置。大多数人会构建抽取流水线来应对这些问题，加入多层解析、模式校验、重试和抽查。此外，每份文档的每个字段都需要支付推理和输出 token 的费用。

Clairic 的工作方式不同，它不生成文本答案。你得到的是：

- 从输入中抽取的**值**，因此答案有据可依。
- **计算值**，因此它可以解析非结构化数据，无需预先清洗。
- **文本片段的字符偏移量**，因此你可以定位数据在输入中的位置。
- **置信度标签**，因此你可以判断哪些值需要再检查。
- **强类型输出**，因此数值和日期都按各自类型的规则形成（例如，精确到美分的金额字段保留两位小数）。
- **对空值的处理**：缺失的字段会返回 `null`（同时附带空值概率的估计值和一个猜测值）。
- **更便宜、更快的答案**，因为所有字段都在同一次并行处理中得到回答。因此，查询三十个字段与查询三个字段的成本大致相同。

下面通过一个示例，说明 Clairic 如何从非结构化数据中抽取值并进行估计。以这封客服邮件为例：

*你好，订单 A-88412 于 2026 年 9 月 28 日送达，屏幕有裂痕，一角也凹了。我支付了 $649.00，另加 $52.10 的税。现在是 2026 年 10 月 9 日，我希望能维修或退款。谢谢，Dana*

你可以从 Worker 向 Clairic 发出请求，处理这些数据：

```
const { answers } = await env.AI.run("@cf/cloudflare/clairic", {
  state: email,
  fields: {
    order_id:            { type: "text", instructions: "Order number" },
    delivered_on:        { type: "date", instructions: "Delivery date" },
    amount_paid:         { type: "number", decimals: 2, unit: "USD", instructions: "Total paid including tax" },
    days_since_delivery: { type: "duration", unit: "days", integer: true,
                           instructions: "Days from delivery to the date the customer wrote" },
    damage:              { type: "list", items: "text", instructions: "Each kind of damage described" },
    tracking_number:     { type: "text" },
  },
});
```
Clairic 会返回抽取出的值、在输入中对应的文本片段、为空的概率（p\_null）以及置信度分数：

```
{
  "order_id": {
    "type": "text",
    "p_null": 3.4e-6,
    "span": [10, 17],
    "guess": "A-88412",
    "confidence": 0.999957,
    "value": "A-88412"
  },
  "delivered_on": {
    "type": "date",
    "p_null": 6.5e-8,
    "guess": "2026-09-28",
    "confidence": 0.999999,
    "value": "2026-09-28"
  },
  "amount_paid": {
    "type": "number",
    "p_null": 1.6e-5,
    "guess": 701.1,
    "confidence": 0.999944,
    "value": 701.1
  },
  "days_since_delivery": {
    "type": "duration",
    "p_null": 6.1e-9,
    "unit": "days",
    "guess": 11,
    "confidence": 0.999999,
    "value": 11
  },
  "damage": {
    "type": "list",
    "value": ["cracked screen", "dented corner"],
    "spans": [[50, 64], [71, 84]]
  },
  "tracking_number": {
    "type": "text",
    "p_null": 1,
    "span": [106, 112],
    "guess": "$52.10",
    "confidence": 0.416016,
    "value": null
  }
}
```
有几点值得注意：

- 邮件中没有直接写出 `amount_paid`。在这个例子中，Clairic 在同一次处理中将价格和税额相加，并返回总额。
- `days_since_delivery` 也是根据文本中的两个日期计算得出的，输入中并没有直接给出。
- `damage` 以原文中的准确文本片段列出了两种损坏情况，因此你可以在原文中将它们高亮显示。
- 邮件中没有 `tracking_number`，所以它返回 `null`，而不是一个看似合理的猜测。`guess` 展示了模型在不返回空值的情况下原本会选中的内容。

### Clairic 也支持图片

下面是一张实际拍摄的收据照片，文字不够清晰，拍摄角度也不利于辨认。我们将这张图片传给 Clairic，看看它能抽取出哪些内容：

![](assets/diagram-e6856ef9c697.webp)

```
{
  "model": "clairic",
  "answers": {
   "order_number": {
    "type": "number",
    "p_null": 0.0040206750854849815,
    "guess": 77245,
    "confidence": 0.776249383996532,
    "value": 77245
   },
   "purchased_at": {
    "type": "datetime",
    "p_null": 4.136094133855295e-08,
    "guess": "2026-07-01T13:34:00",
    "confidence": 0.831464227002992,
    "value": "2026-07-01T13:34:00"
   },
   "quantity": {
    "type": "number",
    "p_null": 5.2313655629632194e-08,
    "guess": 2,
    "confidence": 0.9997007500403496,
    "value": 2
   },
   "unit_price": {
    "type": "number",
    "p_null": 0.001599092734977603,
    "guess": 7.99,
    "confidence": 0.9991279019989127,
    "value": 7.99
   },
   "subtotal": {
    "type": "number",
    "p_null": 9.69956488461321e-08,
    "guess": 15.98,
    "confidence": 0.9999881983893096,
    "value": 15.98
   },
   "sales_tax": {
    "type": "number",
    "p_null": 2.5511653802823275e-06,
    "guess": 0,
    "confidence": 0.9999800924450729,
    "value": 0
   },
   "tip": {
    "type": "number",
    "p_null": 5.953857908025384e-05,
    "guess": 2.39,
    "confidence": 0.9335768781214697,
    "value": 2.39
   },
   "discount": {
    "type": "number",
    "p_null": 0.9994148015975952,
    "guess": 0,
    "confidence": 0.91852733929319,
    "value": null
   }
  }
}
```
可以看到，它从图片中抽取出了所有值，并附带置信度分数。它还正确指出收据中没有显示折扣，将该值返回为 null。

## 示例：估计值

有些值根本不在输入中，但输入提供了相关依据。例如，某个值可能没有直接以文本写出，却能从周围的值中判断出来。如果让 LLM 做估计，你通常只能得到一个数值，无法了解模型有多大把握。它可能给出保守的猜测，可能产生幻觉，也可能经过一番推理却仍绕着答案打转。即使最终得到了估计值，下游处理也无法区分模型很有把握的 $180 和没有把握的 $180，因此无法帮助你判断在当前情况下哪些操作适合自动完成。此外，用 LLM 进行推理也慢得多、贵得多。

决策模型可以通过在多个区间（“低于 $100”“$100–500”等）中做选择，近似给出估计，但这些区间较为粗略，每增加一个区间都会增加输入 token，而每增加一个类别，也会让模型在决策过程中多一些混淆的可能。

Clairic 会在你声明的范围内返回一个**分布**：包括点估计值、中位数、80% 区间，还可以按需返回每一个百分位数。估计值在设计上就会落在你设定的边界内，因此无需事后将其限制在范围内或取整。区间宽度反映了数据带来的把握程度：范围窄意味着确定性高，范围宽则意味着确定性低，你可以据此决定是否安排进一步检查。

下面是另一个示例，Clairic 根据历史房价和可比房产的成交数据给出估计：

```
const { answers } = await env.AI.run("@cf/cloudflare/clairic", {
  state: `Comparable sales:
sqft,beds,year_built,neighborhood,sale_price
1450,3,1998,Elm Park,412000
2100,4,2006,Elm Park,585000
980,2,1975,Riverside,265000
1600,3,2003,Elm Park,449000
1300,3,1990,Riverside,318000
Subject property:
sqft,beds,year_built,neighborhood
1620,3,2001,Elm Park`,
  fields: {
    sale_price:     { type: "estimate", unit: "USD", min: 50000, max: 5000000, scale: "log", step: 1000 }
});
```
Clairic 会返回一个估计值：

```
{
  "sale_price":     { "type": "estimate", "value": 418000, "median": 415000, "p10": 370000, "p90": 470000, "p_below": 0.0, "p_above": 0.0 }
}
```
值始终落在你设定的边界内，区间宽度则表明输入能为预测提供多少依据：价格区间较窄（$370,000-$470,000），挂牌天数区间较宽（24-148）。由于字段数量不限，而且会并行回答，你可以在一次调用中评估许多对象（每个候选对象、潜在客户或房源各做一次估计），再通过代码排序。

## 示例：预测值

对通用模型来说，预测时间序列中的值相当困难。通常，你要么为每个指标搭建专门的流水线（解析数据、拟合模型、调参、部署），要么将 36 个月的数据粘贴到 LLM 中，让它预测接下来的三个月。第一种方式准确，但每遇到一个新序列都要重新搭建，既昂贵又耗时。第二种方式容易上手，但得到的只是没有区间的点预测，预测会随提示词的措辞而变化，答案的可信度也有限。

Clairic 介于两者之间。它并不声称能替代专用的时间序列模型，但比常用的 LLM 稍快、稍便宜。`forecast` 字段接收以纯文本形式放在 `state` 中的历史数据，并为每个未来步返回一个值和一个 80% 区间，都位于你声明的边界内。它能直接读取你已有形式的历史数据，无论是导出的 CSV、粘贴的仪表板内容，还是夹杂着备注的数据行，往往无需事先清洗数据就能工作。

为了看看它在真实数据上的表现，我们使用了一个公开序列：美国运输安全管理局（TSA）每天在美国机场安检点检查的人数。我们连续 21 天，每天早晨都向 Clairic 提供截至当时的实际人数，请它预测第二天的人数：

```
const { answers } = await env.AI.run("@cf/cloudflare/clairic", {
  state: `Daily number of people screened at TSA checkpoints at US airports.
date,weekday,travelers
2026-07-16,Thu,2850229
...
2026-10-13,Tue,2353356
Forecast date: 2026-10-14 (Wed)`,
  fields: {
    travelers: { type: "forecast", horizon: 1, interval: "day",
                 min: 500000, max: 4000000, scale: "log",
                 instructions: "Number of people screened tomorrow" },
  },
});
```
返回结果为：

```
{
  "travelers": { "type": "forecast", "value": [2416117], "median": [2419283], "p10": [2270774], "p90": [2556769] }
}
```
实际上，TSA 当天检查了 2,419,474 人，与 Clairic 预测的中位数 2,419,283 非常接近。

![](assets/diagram-cbf13770a92b.webp)

*数据：[<u>TSA 安检点旅客人数</u>](http://tsa.gov/travel/passenger-volumes)，访问日期为 2026 年 10 月 15 日。*

在这 21 天里，Clairic 的平均预测误差为 3.3%，21 个实际人数中有 17 个落在它给出的 80% 区间内，符合 80% 区间应有的覆盖表现。它仅凭数字就识别出了每周的节奏：周二和周六人数较少，周四、周五和周日人数较多。

预测失准的情况也很有意思：落在区间外的四天中，有三天处于哥伦布日的周末假期，乘机人数高于此前几周的数据所暗示的水平。Clairic 根据序列中的规律进行预测，但不会考虑序列没有体现的事件，例如节假日。

## 快思考与慢思考

这些模型旨在融入你现有的智能体工作流。抽取模型预测值，决策模型决定如何处理这些值，LLM 再据此采取行动。这是 System One 与 System Two 模型的完美结合。它弥合了现有方案的缺口：此前，你要么手动为 LLM 抽取数据，要么让 LLM 解析杂乱的数据（成本高、耗时长，还可能产生幻觉）。借助 System One 抽取模型，你可以更快、更高效、更精确地处理现实世界的数据。

下面继续使用前面的商品损坏索赔示例，说明这些模型如何协同工作。我们将它做成了一个[<u>可以在这里试用的在线演示</u>](https://clairic-pipeline-demo.workers-ai-mle.workers.dev/)。

```
// 1. Clairic turns the email into values
const facts = (await env.AI.run("@cf/cloudflare/clairic", {
  state: email,
  fields: {
    customer_name:       { type: "text", instructions: "Customer's first name" },
    order_id:            { type: "text", instructions: "Order number" },
    amount_paid:         { type: "number", decimals: 2, unit: "USD", instructions: "Total paid including tax" },
    days_since_delivery: { type: "duration", unit: "days", integer: true,
                           instructions: "Days from delivery to the date the customer wrote" },
    damage:              { type: "list", items: "text", instructions: "Each kind of damage described" },
    repair_cost:         { type: "estimate", unit: "USD", min: 10, max: 5000, scale: "log", step: 1,
                           instructions: "Cost to repair the damage described" },
  },
})).answers;
const { customer_name, order_id, amount_paid, days_since_delivery, damage, repair_cost } = facts;
// 2. Clef decides from Clairic's values and the policy, not from the raw email
const options = {
  refund:  "Full refund of the amount paid",
  repair:  "Free repair, with a prepaid shipping label",
};
const { choice } = (await env.AI.run("@cf/cloudflare/clef", {
  state: `Damage claim for order ${order_id.value}.
Paid: $${amount_paid.value.toFixed(2)}. Delivered ${days_since_delivery.value} days ago.
Damage reported: ${damage.value.join(", ")}.
Estimated repair cost: $${repair_cost.median} (80% range $${repair_cost.p10}-$${repair_cost.p90}).
Policy: refunds within 30 days of delivery. Prefer repair when it costs less than 60% of the price and the damage doesn't stop the product from working.`,
  questions: {
    resolution: { type: "choice", instructions: "What should we offer?", criteria: options },
  },
})).answers.resolution;
// 3. An LLM writes the reply, from the decision and the facts it needs
const { response } = await env.AI.run("@cf/zai-org/glm-5.3", {
  messages: [{
    role: "user",
    content: `Write a short, warm reply to ${customer_name.value}. Use only these facts: order ${order_id.value}; ` +
             `damage: ${damage.value.join(", ")}; what happens next: ${options[choice]}. Under 60 words.`,
  }],
});
```
每一步拿到的输入都比上一步更规整。Clef 根据五行由类型明确的值组成的摘要做决策，而不直接处理自由形式的邮件；这样的邮件还可能遭受提示注入。LLM 完全看不到邮件，只能看到它所需的决策和预测值，因此可能被它误用的数字更少。每一步都有明确类型且可供审计，只有一步会生成文本。在全部使用 LLM 的版本中，这三个步骤都是生成式调用，回复中的每个值都必须人工核查。

我们将这条流水线与全部使用 LLM 的版本进行对比，两者都由 GLM-5.3 撰写回复。在全部使用 LLM 的版本中，GLM-5.3 还会根据相同的邮件、政策和提示词完成抽取与决策：

|  | Clairic → Clef → GLM-5.3 | 每一步均使用 GLM-5.3 | 
| 端到端延迟 | 19.3 秒 | 61.7 秒 | 
| 1. 抽取 | Clairic：0.46 秒 | GLM-5.3：16.1 秒 | 
| 2. 决策 | Clef：0.22 秒 | GLM-5.3：21.9 秒 | 
| 3. 回复 | GLM-5.3：18.6 秒 | GLM-5.3：16.4 秒 | 
| 输入 token | 701（每百万次索赔 $184） | 390（每百万次索赔 $546） | 
| 输出 token | 1,329（每百万次索赔 $5,848） | 2,544（每百万次索赔 $11,194） | 
| 每百万次索赔的成本 | $6,032 | $11,740 | 

级联流程用时 19.3 秒，而另一种流程用时 61.7 秒：**耗时减少 69%，速度约为原来的 3.2 倍**。Clairic 和 Clef 完成抽取与决策的总用时不到一秒，而 GLM-5.3 完成相同的两个步骤需要 38 秒。与全部使用 LLM 的流水线相比，采用级联流水线还可使**成本降低 49%**。使用 GLM 时，两条流水线在每次运行中都得到了正确的事实和决策，但 Clairic 每次给出的维修费用估计完全一致，GLM-5.3 的估计则每次都有变化（分别为 $250、$250 和 $220，且每次的区间也不同）。

值得注意的是，在全部使用 LLM 的流水线中，Gemma 4 和 Qwen 都未能完成这项任务，因为它们无法根据输入推断出 repair\_cost 这样的值，这表明 LLM 并不擅长数据抽取任务。

![](assets/diagram-70a34eea079d.webp)

## 预览版发布

今天，我们在 Workers AI 上发布了 Clairic 的托管端点。我们计划很快开放模型权重。在此之前，我们希望先花些时间介绍 System One 抽取模型的概念，让你开始思考这类模型适合融入哪些工作流。

当前端点使用的是早期版本，我们预计它会继续改进。以下是我们目前的初步观察：

- **读取能力很强，但多步算术还不够好。**一步和两步计算（求和、求差、换算、日期运算）似乎较为可靠。对于任何没有草稿区的单次处理模型来说，较长的、步骤相互依赖的计算链都很困难。对表格中的多行数据进行聚合也是如此。至少目前，最好让 Clairic 预测值，再让代码完成数学运算。
- **估计需要依据。**当输入包含可比对象、历史数据或描述性文本时，Clairic 的估计效果最好。如果要求它估计一个没有任何依据的值，答案能具体到什么程度可能会受到限制。
- **校准表现需要测量，并无保证。**这些区间在训练时以校准为目标，在大多数留出数据集上也确实经过了良好校准，但遇到不熟悉的数据时，区间可能偏窄。
- **预测能力还很基础**。模型的设计目的并不是准确替代用特定数据训练的时间序列模型，也不是准确预测这些模型的结果。你应当将它用于适当的场景，例如非关键用途中的一般规律匹配，而非细致的长期预测。
- **利用置信度。**模型架构能确保每个答案的格式正确，但不一定内容正确。应利用 `confidence`、`p_null` 和区间，在适当的工作流背景下判断哪些操作可以自动完成，哪些需要送交进一步审核。

我们会在每个后续版本中持续改进模型的算术、估计和预测能力。再完善一段时间后，我们也计划开放模型权重，供所有人探索和试用。

## 试用

Clairic 已在 Workers AI 上提供预览版，模型标识为 `@cf/cloudflare/clairic`。你可以在 Worker 中通过 `env.AI.run()` 调用，也可以从任何地方通过 REST API 调用。我们在[<u>这里</u>](https://developers.cloudflare.com/workers-ai/models/clairic/)提供完整的模型文档，也新增了一篇[<u>文档说明</u>](https://developers.cloudflare.com/workers-ai/concepts/extraction-models/)，介绍抽取模型 API 的形式。

附言：既然大家喜欢了解 [<u>*Clef* 这个名字的词源</u>](https://blog.cloudflare.com/clef-decision-models#what-is-a-decision-model)，这里也说说我们是如何确定 *Clairic.* 这个名字的。最初，我们希望模型的名字在主题和发音上都与 Clef 相近。我们想到了 Clair，它们都有“cl”这个音，也都与音乐有渊源（“Clair de Lune”）。更重要的是，我们希望这个名字能体现模型的特点。Clairic 模型应该能帮你完成*文书*工作，例如从一组输入中*清晰地*抽取数据，并在记账等任务中保持准确。模型的预测能力则呼应了“*Clair*”这个词根：让你能够估计和预测未来的值，类似于*预知能力*。最后，我们也喜欢把它看作 \[Cl\]oudflare 的 \[AI R\]egression（AI 回归）模型。于是，Clairic 就这样诞生了。希望你喜欢它。
