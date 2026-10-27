# 完全可编程的安全 Web 网关

> **来源**: [https://blog.cloudflare.com/programmable-gateway-dlp](https://blog.cloudflare.com/programmable-gateway-dlp)  
> **作者**: 未知 | **发布时间**: 2026-10-15T17:11:12.510000+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: 安全Web网关, SASE, 边缘计算, Cloudflare Workers, 零信任, DLP, 策略引擎

---

## AI 导读

文章要解决的是传统安全 Web 网关只能在条件匹配后执行有限终止动作，难以处理企业自定义的请求变换、动态策略、异步通知和针对具体数据实例的 DLP 判断。方案是在 Gateway 策略路径接入 Cloudflare Workers：Forward 将匹配的 HTTP 请求交给 Worker 作为内联中间件，可修改请求和响应或触发异步副作用；Custom Functions 在策略评估时接收请求、身份、设备等上下文，返回字符串、数字或布尔值作为策略选择器；DLP custom validation functions 则调用企业自有逻辑验证检测结果。文章以 JWT 角色注入、审批页、文件检查、Slack/PagerDuty 告警、风险评分和 GitHub 密钥有效性验证为例，结论是借助同机边缘调度可扩展 Gateway 行为而无需自建代理。局限包括 Forward 仍为公开测试版、Custom Functions 预计后续提供，Workers 受执行时间和请求量限制且按标准请求计费；文中没有独立性能数据、完整安全威胁模型、资源配额或故障语义，代码示例也不足以证明生产安全性。

---

## 原文整理

# 完全可编程的安全 Web 网关

![](assets/diagram-60305d4b7621.webp)

市面上的安全 Web 网关都遵循同一种工作方式：流量匹配某个条件，然后触发一个终止动作——允许、阻止、隔离或记录日志。近 30 年来，这种模式一直是网络安全的基础，也能很好地处理常见场景。但它有能力上限，环境复杂的客户经常会碰到这个上限。

今年早些时候，我们分享了构建[真正可编程的 SASE 平台](https://blog.cloudflare.com/programmable-sase/)的愿景。我们还谈到，安全行业已经淡化了“可编程性”这个词的含义。大多数厂商宣称自己的产品可编程，是因为它们提供了公共 API、Terraform 提供程序、webhook 和告警功能。Cloudflare 也提供这些能力，但这还不够。真正的可编程性，是在做出安全决策的那一刻，在策略评估流程中直接运行你自己的代码，也就是你为特定环境编写的任意逻辑。

今天，我们很高兴地宣布，Cloudflare 正在将 Cloudflare Gateway 和 DLP（数据丢失防护）与我们完整的边缘计算平台集成，为它们带来真正的可编程性。我们将推出一种新的 Gateway 动作 *Forward*，以及自定义验证函数，面向所有 Zero Trust 套餐提供。这些能力让你可以使用 Cloudflare Worker 转换和处理流量，也可以将自己的数据用于策略评估。

## 每个安全团队都面临的取舍

自防火墙问世以来，企业就一直面临着二选一的局面。你可以维护自己的安全检测引擎，包括自己的防火墙和代理基础设施，以高昂的维护成本和较差的可移植性，换取最大限度的定制能力。或者，你可以接受厂商的限制，把大量维护工作交给厂商，但代价是缺少解决自身环境特有问题的灵活性。

在构建 Cloudflare Gateway 的六年里，我们听过数千家客户讲述他们在这种取舍中的困境。比如，一个安全团队正在迁移某个关键内部应用的访问控制，需要根据身份提供商签发的 JSON Web Token（JWT）中的声明，注入 HTTP 请求头。这些数据就在请求上下文中，但 Gateway 无法读取 JWT 并写入请求头。因此，团队只能专门搭建一组 NGINX 代理来完成这项转换。从此以后，这套基础设施就需要一直打补丁、监控并做好安全防护……没有尽头。

另一个团队希望，员工一注册新的 SaaS 服务，就立即收到 Slack 通知。策略引擎可以检测到注册事件，但只能执行允许或阻止动作。于是，团队构建了一条从 Logpush 到 SIEM 再到 Slack 的处理链路，通知会延迟 5 到 15 分钟，而且无法保证送达。需要维护的基础设施又多了一项。

数十家企业客户希望提供警告页面，要求用户先说明理由，才允许访问敏感目标。其中大多数客户还希望，根据用户身份和所请求访问权限级别等额外信息，定制后续行为。但要让策略引擎提供定制化的动态响应，极其困难。

多年来，我们为客户提供了许多扩展点。但和其他所有防火墙厂商一样，我们不可能以威胁出现的速度，为每一种威胁途径都开发出相应功能。同时，我们也认识到，最好的解决方案应当能够针对你的环境无限扩展。

## 为什么只有 Cloudflare 能解决这个问题

Cloudflare 不只是防火墙、安全 Web 网关、内容分发网络或边缘服务。Cloudflare 是一个可组合的计算平台。Cloudflare Workers 已经允许客户在我们遍布全球的 330 多个节点中的任意节点上，在网络边缘运行复杂逻辑。[Cloudflare Gateway](https://www.cloudflare.com/products/gateway/)也已经在这些相同的节点、相同的数据中心、相同的机器上，对用户流量评估并执行安全策略。

只有 Cloudflare 在同一个全球分布式网络中，同时运行面向客户的分布式计算服务，以及功能完整的安全 Web 网关和防火墙。我们拥有独特的机会和能力，可以将这些功能结合起来，构建业内首个*完全*可编程、可定制的 SASE 平台。

Gateway 和 Workers 从一开始就建立在相同的基础设施上。当 Gateway 将流量转发给 Worker 时，这次调用就是一次本地函数调用。流量不需要绕行，也不会因为跨越网络边界而增加延迟。客户将首次能够完全掌控代理流量的处理方式，无需妥协：只需编写并执行自己的代码，而不必维护任何基础设施。

没有自有计算平台的独立 SASE 提供商，会要求你在另一个云中部署自动化功能、手动配置 webhook，并接受将彼此独立的系统拼接起来所带来的往返延迟和管理开销。这个领域没有其他厂商能让可编程边缘计算平台与安全网关部署在同一处。随着威胁借助 AI 加速演进，这种能力将为安全团队提供迫切需要的灵活性和应变能力。

## Gateway 的两个入口：*Forward* 动作与自定义函数

新的 Gateway 动作类型 *Forward*，以及新的策略资源“自定义函数”（*Custom Function*），是在 Gateway 策略中调用 Worker 的两种不同方式。Forward 动作今天起进入公开测试，自定义函数将在年底前推出。

### *Forward* 动作：流量转换与代理

![](assets/diagram-0a89211210fb.webp)

你可以使用名为 *Forward* 的新[策略动作](https://developers.cloudflare.com/cloudflare-one/traffic-policies/order-of-enforcement/)，自定义 HTTP 流量的处理方式。它会将匹配的 HTTP 请求路由到一个充当代理的 Worker。对于已被 HTTP 策略允许通过的流量，Gateway 会评估带有 *Forward* 动作的策略。第一条匹配的 *Forward* 规则将应用于这段流量，流量随后会被发送到该规则绑定的 Cloudflare Worker。Worker 作为请求处理链路中的中间件，会接收到完整的请求头和请求体。你的逻辑可以修改其中任意一项，也可以执行某种异步操作，然后将请求转发到源站。收到响应后，还可以修改响应，最后将最终响应返回给用户。

```
import { WorkerEntrypoint } from "cloudflare:workers";
import type {
  GatewayMiddlewareContext,
  GatewayMiddlewareNext,
} from "@cloudflare/zt-types";
export class GatewayMiddleware extends WorkerEntrypoint {
  async handle(
    request: Request,
    context: GatewayMiddlewareContext,
    next: GatewayMiddlewareNext,
  ): Promise<Response> {
    // 从 Gateway 上下文中读取一个值
    const clientIP = context.srcIp;
    // 在请求到达源站之前添加请求头
    request.headers.set("X-Gateway-Client-IP", clientIP);
    // 向上游转发请求并获取响应
    const response = await next(request);
    // 在响应到达用户之前修改响应
    response.headers.set("X-Processed-By", "gateway-middleware");
    return response;
  }
}
```

Gateway 负责处理 TCP 连接、TLS 终止和重新加密、证书管理，以及 HTTP 版本协商。Worker 只负责转换逻辑。这样，你现有的安全技术栈就获得了一个可定制、类似代理的扩展点，而无需承担运行传统 HTTP 代理的运维成本。

Worker 也不一定要转换请求。一条规则的唯一用途，可以是将请求代理到某个 Worker，让它获得匹配请求的完整上下文，并异步执行操作：发送 Slack 消息、写入 [R2](https://developers.cloudflare.com/r2/)、向外部处理链路记录日志、调用 webhook，或更新 [KV](https://developers.cloudflare.com/kv/) 存储。流量则可以不受干扰地继续前往目的地。

### 自定义函数：在策略评估过程中直接执行逻辑

![](assets/diagram-3536a80a46ef.webp)

自定义函数会在 Gateway 策略评估过程中直接调用 Worker，并返回一个值。随后，这个函数会在 Gateway 策略构建器中作为选择器提供，系统会根据它返回的值进行条件判断。你可以在任意网络或 HTTP Gateway 规则的条件中使用这个选择器，用法与现有的 `Source IP`（源 IP）或 `Content Category`（内容类别）等内置选择器相同。

Worker 会接收请求的上下文信息，包括源 IP、目的地、用户身份、设备安全状态、HTTP 请求头等，具体取决于所处的策略阶段。Worker 返回一个带有明确类型的值，可以是字符串、数字或布尔值，供 Gateway 在规则条件中使用。接口约定很简单：输入数据，输出一个值。

下面这个自定义函数会解析 JWT `Authorization` 请求头，并返回用户的角色：

```
export class GatewayCustomSelector extends WorkerEntrypoint {
  async compute(ctx: GatewayContext): Promise<string> {
    const authHeader = ctx.request.headers.get("Authorization");
    if (!authHeader) return "anonymous";
    const token = authHeader.replace("Bearer ", "");
    const payload = JSON.parse(atob(token.split(".")[1]));
    return payload.groups?.[0] ?? "unknown";
  }
}
```

部署这个函数后，管理员就可以创建一条 HTTP 策略规则：**自定义函数“JWT Role”的值包含“finance_contractor”**，动作为允许。这个自定义函数会成为规则条件的一部分，与其他所有选择器一起参与评估。

自定义函数并不限于 HTTP 流量。在处理非 HTTP 流量时，它们接收的是身份、设备安全状态和连接元数据，而不是完整的 HTTP 请求，因此你也可以在 Gateway 网络策略规则构建器中，将它们用于任意第 4 层端口和协议的流量。

Worker 本身可以执行 Worker 原本就能执行的任何操作，例如调用外部 API、读取 KV 或 D1，或调用 Workers AI。但在 Gateway 看来，这个函数是确定性的：输入数据，输出一个值。

#### 何时使用自定义函数，何时使用 *Forward* 动作

自定义函数适合对数据执行快速、有明确边界、无状态的评估，并返回符合规定格式的响应。你可以参照 Gateway 现有选择器能够评估的数据来理解它的用途：按照一组条件对数据进行计算，例如“处于这个范围内的端口”，或者“匹配这个正则表达式的 URL 路径”。它的强大之处在于，你可以利用任意数据源或内部数据源，扩展这些结构化评估。下文会给出一些例子。

使用 *Forward* 动作则需要多做一些规划和准备。它可以在用户实际请求的处理过程中进行强大的转换，并在请求发往源站以及响应返回的过程中，对它们进行复杂处理。你应当将它们视为转换函数，而如今极少有企业安全代理甚至会从这个角度构思功能。使用 *Forward* 动作的 Worker 应当用途明确、设计谨慎，以免对最终用户的整体体验造成不利影响。

## 你可以构建什么

### 访问用途说明与中间提示页

Worker 可以拦截发往敏感目标的请求，并返回一个中间提示页，要求最终用户以文字说明访问理由。Worker 调用 [Cloudflare KV](https://developers.cloudflare.com/kv/)，写入一条带有生存时间（TTL）的会话记录，让用户在配置的时间段内只需填写一次。它还会将用户身份和用户填写的理由写入 [Cloudflare D1](https://developers.cloudflare.com/d1/)，形成审计记录。你也可以使用 [Hyperdrive](https://developers.cloudflare.com/hyperdrive/)，将这条记录写入任何你已经在运行的内部数据库。

四年多来，这一直是客户需求最强烈的功能之一。现在，Worker 能够处理和修改 HTTP 流量，按照你自己的标准构建这项功能就很容易了。

### 动态注入 HTTP 请求头

自定义函数从请求的 `Authorization` 请求头中解析 JWT，提取用户的角色或所属组，并将其作为选择器的值返回。Gateway 规则根据这个值进行匹配，然后在发往上游的请求中设置 `X-App-Role` 请求头。无需 NGINX 代理集群，也无需维护额外的基础设施。

### 文件检查与记录

当文件下载符合特定条件时，Worker 可以将文件发送到第三方沙箱服务中，运行文件并进行分析。它也可以将整个文件写入 [Cloudflare R2](https://developers.cloudflare.com/r2/)，作为永久记录，供以后检查，而文件始终不会离开 Cloudflare 网络。

### 策略匹配时实时告警

每当流量匹配某个数据丢失防护（DLP）配置文件时，绑定到 Forward 策略的 Worker 就会收到这段流量。Worker 随后向 Slack 频道或 PagerDuty 端点发送消息，其中包含用户的电子邮件地址、触发的规则和目标 URL。

```
export default {
  async invoke(ctx: GatewayContext, env: Env): Promise<void> {
    await fetch(env.SLACK_WEBHOOK_URL, {
      method: "POST",
      body: JSON.stringify({
        text: `DLP alert: ${ctx.identity.email} triggered "${ctx.rule.name}" accessing ${ctx.request.url}`
      })
    });
  }
}
```

### 复杂的评估逻辑

传统防火墙的表达式构建器，并不是为某些组织在评估时所需的计算而设计的。如果你维护着内部风险评分系统，需要结合多个数据源进行行为分析；或者，你维护着复杂的用户、组及附带条件的成员关系映射，要求多个条件分别成立，那么你可以使用自定义函数来扩展计算能力，评估所有这些逻辑，并向 Gateway 返回一个值。

## 性能与可靠性

由于 Gateway 和 Workers 服务运行在 Cloudflare 网络中的同一批机器上，调用 Worker 不会增加一次网络跳转。性能影响仅限于 Worker 代码本身的执行时间。

当自定义函数执行失败时，无论是抛出异常、超时，还是返回非预期值，你都可以为每个函数配置回退行为。自定义函数的接口约定要求必须提供**默认值**，因此很容易确定失败时的行为。

## 可用性与定价

*Forward* 动作及其与 Workers 的集成从今天起进入公开测试，*自定义函数*预计将在年底前推出。正式发布后，两者都将面向所有套餐类型的全部客户提供。

发送到 Worker 的流量按标准 Worker 请求计费。这项功能适用于 Workers Free、Workers Paid 和 Workers for Enterprise，功能上没有限制。不过，Workers Free 有执行时间限制和每日请求量限制。当你增加发送给 Worker 的 Gateway 流量时，需要考虑这些限制。

## DLP 自定义函数：在检测过程中直接验证结果

[完全可编程的 SASE 平台](https://blog.cloudflare.com/programmable-sase/)并不止于 Gateway。我们的 DLP 服务已经集成到 Gateway、AI Gateway、Email Security 等产品中，如今也通过与计算平台集成，支持自定义处理结果。

大多数数据安全产品通过查找符合已知模式的数据来工作。客户通常可以定义正则表达式、词表或类似规则来执行这些检测，也可以利用关键词或对请求体的上下文分析，提高匹配结果的可信度。但这些方法有一个根本缺陷：这类检测**只能**告诉你，某些内容符合与你的组织有关的数据*模式*，却无法判断*这一份具体数据*是否表明发生了数据外泄或滥用。

随着攻击者越来越先进，智能体也能被用来无情地利用系统漏洞，仅仅提高模式检测的准确性已经不够。要更准确地识别真正重要的数据，你需要知道被检测的数据当前是否确实与你的组织有关。

为了解决这个问题，大多数解决方案采用精确数据匹配，但这种方法也有自己的取舍。客户必须准备、维护并传输参考数据集，然后判断自己在多大程度上愿意让厂商管理来自其最敏感系统的数据。

DLP 自定义验证函数弥补了这一缺口，并从今天起提供。这些自定义函数允许客户直接在 Cloudflare Worker 中，使用自己认定的权威数据源，定义自己的验证逻辑……并将其应用到使用 DLP 保护敏感数据的*所有地方*。

### 凭据与密钥验证

当某个 DLP 检测条目被匹配时，可以调用一个向外发出请求的 Worker，在做出策略执行决定之前运行验证逻辑。如果你配置了用于检测 GitHub 密钥使用情况的配置文件，但只关心仍然有效的密钥，就可以先用 Worker 查询检测到的密钥状态，获取相应 GitHub 端点的响应，再决定该 DLP 配置文件是否应判定这段流量匹配。

```
import {
  DlpValidation,
  type DlpValidationInput,
  type DlpValidationResult,
} from "@cloudflare/zt-types";
export class DlpValidationEntrypoint extends DlpValidation {
  async validate({ excerpt, start, end }: DlpValidationInput): Promise<DlpValidationResult> {
    const bytes = Uint8Array.from(atob(excerpt), (c) => c.charCodeAt(0));
    const token = new TextDecoder().decode(bytes.slice(start, end));
    const res = await fetch("https://api.GitHub.com/user", {
      headers: {
        Authorization: `Bearer ${token}`,
        "User-Agent": "dlp-GitHub-validator",
        Accept: "application/vnd.GitHub+json",
      },
    });
    if (res.ok) return { score: 1.0 }; // 令牌有效且处于启用状态
    if (res.status === 401) return { score: "veto" }; // 令牌无效，舍弃此次检测结果
    return { score: 0 };
  }
}
```

## 接下来

这只是完全可编程 SASE 平台的起点。我们正在积极增加调用自定义功能的入口，将这种能力扩展到整个 Cloudflare One 套件，下一个将是 Email Security。这并不意味着我们不再开发功能：新的架构将使我们能够更快、更灵活地为所有 Cloudflare One 服务构建原生功能，同时改善开发者体验。

使用 Cloudflare One [保护你的 Worker、智能体及其流量](https://developers.cloudflare.com/changelog/post/2026-06-05-gateway-egress/)也在变得越来越容易。如果你是 Enterprise 客户，希望就实施方案进行一次策略讨论、预览即将推出的功能，或在自定义函数进入测试阶段时收到通知，请联系你的客户团队。

现在，Cloudflare Worker 能做到的任何事，你的组织安全系统都能做到。我们迫不及待地想看看你会构建出什么。