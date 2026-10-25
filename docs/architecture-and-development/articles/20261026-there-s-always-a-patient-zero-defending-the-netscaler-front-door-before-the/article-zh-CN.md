# 总会有人最先中招：在打补丁之前守住 NetScaler 的入口

> **来源**: [https://blog.cloudflare.com/protect-your-netscaler-gateway](https://blog.cloudflare.com/protect-your-netscaler-gateway)  
> **作者**: Kenny Johnson | **发布时间**: 2026-10-19T13:00:00+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: 零信任访问控制, NetScaler Gateway, 虚拟桌面基础设施, Web应用防火墙, 源站访问限制, 身份认证, 纵深防御, 安全运维验证

---

## AI 导读

文章讨论企业认证网关公开暴露时，未认证漏洞如何使登录入口本身成为攻击面，以及补丁部署期间和未来漏洞出现前如何降低风险。其方法是把独立的身份访问决策放在原认证服务之前，由 WAF 检查 HTTP 请求，并通过出站 Tunnel，或专用回源地址、防火墙、双向 TLS、Access JWT 校验等机制约束源站访问，防止攻击者绕过前置控制。原 Citrix 环境继续负责桌面授权和会话票据校验。

文章明确区分网页认证路径与 ICA、DTLS 等其他连接：浏览器认证代理不会自动保护全部桌面流量，后者需要另行设计网络路由和访问策略。它还讨论浏览器远程桌面和替代仅用于访问网页的虚拟桌面场景，并指出消费者登录页及建立初始会话所需的身份提供方端点可能必须保持公开，避免循环认证依赖。

结论是从一个受限服务开始，梳理网页和非网页入口，前置访问控制并封闭源站绕过路径；上线前验证未登录、无权限账号、源站直连及合法用户桌面启动与重连，并结合日志持续复查。补丁与已入侵环境的调查恢复仍然必要。局限是材料主要提供产品架构建议，缺少具体配置、漏洞复现、兼容性与性能测试、防护效果数据；所述漏洞与产品能力未在正文中获得独立验证，不能将方案主张视为实证结果。

---

## 原文整理

# 不止于打补丁：保护你的 NetScaler 网关，应对未来的漏洞

![](assets/diagram-c84c05e942bb.webp)

近期披露的 Citrix NetScaler [重大漏洞](https://www.cisa.gov/news-events/alerts/2026/09/27/critical-zero-day-vulnerabilities-exploited-citrix-netscaler-adc-gateway)暴露出一个普遍的薄弱环节：登录页面本身就是攻击面，攻击者可以利用它入侵并访问应用服务器。事实上，[已披露的漏洞](https://support.citrix.com/support-home/kbsearch/article?articleNumber=CTX697191&articleURL=Citrix_NetScaler_ADC_and_Citrix_NetScaler_Gateway_Security_Bulletin_for_CVE_2026_107406)中，有一个正遭到攻击者利用，允许他们在不登录的情况下，在受影响的网关上执行命令。和每次紧急打补丁一样，管理员都得权衡：是切断或干扰用户访问，还是承担推迟关键安全更新的风险。

如果你是 NetScaler 客户，而且还没有给自己的实例打补丁，请先别往下读，马上去打补丁。这次的漏洞很严重。等你打完补丁，我们再来探讨如何增加安全控制，避免今后再次陷入紧急应对的局面。

这个问题并不限于 NetScaler，同样适用于任何可从互联网访问的内部应用。远程桌面门户、管理控制台以及其他公开可访问的身份认证页面，都值得同样仔细地审视：目标用户是否需要通过不受限制的互联网连接访问它？攻击者能否绕过这道身份认证检查，直接访问应用服务器？

Citrix [发布了修复该漏洞的软件更新](https://support.citrix.com/external/article/CTX697096)，Cloudflare 随后于 10 月 1 日推出了[紧急 Web 应用防火墙（WAF）规则](https://developers.cloudflare.com/changelog/post/2026-10-01-emergency-waf-release/)。但有补丁可用，并不意味着已经安全：每一个实例都必须被找出来、完成更新并经过测试。在生产环境中全面部署更新很麻烦，可能导致停机，也可能影响员工，而他们在整个更新过程中仍然需要使用桌面。客户、患者，以及其他依赖这些远程桌面用户提供服务的人，可不会在意你要打什么安全补丁。

使用 Web 应用防火墙（WAF），可以在补丁全面部署之前争取一些时间，但这种做法属于事后应对，无法顾及下一个被发现的漏洞利用方式。Cloudflare 和其他安全厂商会尽力覆盖攻击变种及类似攻击。但 AI 正在加快攻击者发现全新变种的速度。因此，构建[纵深防御](https://www.cloudflare.com/learning/security/glossary/what-is-defense-in-depth/)至关重要，要确保任何单一软件故障或漏洞利用都不会引发严重的安全问题。

Cloudflare 将访问策略、Web 应用安全和受限的源站连接结合起来，在新漏洞披露之前保护身份认证服务。访问控制限制谁能访问应用服务器；WAF 检查 Web 请求，判断其中是否存在攻击；源站访问限制则防止攻击者通过直连绕过这些防御措施。这些措施能够减少暴露面，无需等待新的漏洞利用特征，也无需替换现有的桌面平台。

## NetScaler 漏洞利用

Citrix 在 [2026 年 9 月 27 日的公告](https://support.citrix.com/external/article/CTX697096)中报告，已观察到针对 CVE-2026-88771 和 CVE-2026-88772 的漏洞利用活动。前者允许攻击者在未经身份认证的情况下，在受影响的设备上执行命令，使用默认配置的设备也包括在内。后者影响启用了数据报传输层安全协议（DTLS）的配置。[10 月 3 日](https://support.citrix.com/external/article/CTX697174)和 [10 月 8 日](https://support.citrix.com/external/article/CTX697191/citrix-netscaler-adc-and-citrix-netscale)发布的后续公告又涉及 SAML 配置中的其他漏洞。利用这些漏洞，攻击者可以在目标机器上执行任意脚本，也可以发起拒绝服务攻击。

[CISA 的警报](https://www.cisa.gov/news-events/alerts/2026/09/27/critical-zero-day-vulnerabilities-exploited-citrix-netscaler-adc-gateway)和 [Mandiant 的技术调查](https://cloud.google.com/blog/topics/threat-intelligence/defending-against-active-exploitation-of-citrix-netscaler-adc-and-gateway-appliances)更详细地介绍了这些漏洞、已观察到的利用活动以及应对指导。如需了解现有 Web 应用防护措施的详情，请参阅 [Cloudflare 针对 CVE-2026-88771 发布的紧急 WAF 更新](https://developers.cloudflare.com/changelog/post/2026-10-01-emergency-waf-release/)。

在典型的 Citrix 部署中，用户先通过 NetScaler Gateway 完成身份认证，网关随后借助安全票据颁发机构接入 StoreFront，由 StoreFront 展示该用户可用的应用和桌面。安装在本机上的 Citrix 应用再利用这些信息，通过 Citrix 的独立计算架构（ICA）协议建立另一条连接。网关会先验证票据，再代理桌面连接。Citrix 文档将这些过程列为[连接流程中不同的阶段](https://support.citrix.com/external/article/227054/netscaler-gateway-storefront-and-xendesk.html)。整个漏洞利用的根源在于，Gateway 必须能从互联网上公开访问，否则用户就无法发起桌面连接。

## 在请求到达身份认证服务器之前决定是否允许访问

在为所有实例打完补丁后，你最应该做的，就是将所有公开可访问的登录页面或应用放到单点登录之后，让请求先经过单点登录，再到达目标服务器。[Cloudflare Access](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/) 可以在请求到达现有网关服务器之前，执行[独立的访问策略](https://blog.cloudflare.com/access-aegis-cni/)。Citrix 环境则继续负责确定用户可以启动哪个桌面，并验证由此生成的会话票据。新增的这道边界确保攻击者无法接触 Gateway 服务器，也就无法尝试利用特定漏洞。他们只能看到 Cloudflare Access 登录页面。

这种访问决策不再依赖于识别漏洞利用特征并据此拦截，而是通过一组零信任策略，判断某个人究竟是否应该获准访问。未来还会出现更多此类攻击，而在整个网络中采用零信任控制，是你接下来最应该做的一件事。

这遵循了 [Cloudflare 作为零号客户在自身架构中采用的原则](https://blog.cloudflare.com/frontier-model-defense/)：减少暴露面，并使用相互独立的防御措施，避免一个漏洞就导致系统被攻破。在这里，切实可行的起点就是用户进入应用之前首先访问的服务。

![](assets/diagram-bde534cf67be.webp)

保护 Web 身份认证路径，相当于在现有 Citrix 控制措施之前增加一道边界。但这道边界不会自动延伸到其他网关服务，包括启用了 DTLS 的监听服务。即使这些接口所支持的桌面会话已经要求身份认证，仍然需要单独审查这些接口的暴露情况，并安装适用的安全更新。

## 源站连接必须落实同样的访问边界

仅修改 DNS 记录并执行 Access 策略还不够。如果攻击者能连接到原始地址或其他主机名，就可以绕过前置的控制措施。Access 针对特定主机名执行策略：如果能通过 IP 地址或其他主机名访问身份认证服务器，那么 Access 策略的执行就毫无意义。务必采取措施，确保所有流量都来自经过 Access 验证的请求。根据内部架构的不同，有几种方式可以实现这一点。

最常用的方式是 [Cloudflare Tunnel](https://developers.cloudflare.com/fundamentals/security/protect-your-origin-server/)。它从应用服务器向外建立隧道，只需出站连接，使你可以关闭所有入站 IP 流量。这意味着服务器不再有任何人都能访问的公网 IP 地址。

如果必须保留源站的公网 IP 地址，或无法运行连接器软件，就需要通过网络限制和源站认证，约束源站接受的连接。例如，[Dedicated CDN Egress IPs](https://developers.cloudflare.com/smart-shield/configuration/dedicated-egress-ips/) 会为账户提供专用源地址，供 Cloudflare 向该账户的源站发送流量。应用服务器的防火墙可以放行这些地址，并拒绝其他来源。这样，[网络限制就能专门针对该账户的流量](https://blog.cloudflare.com/aegis-deep-dive/)，而不必笼统地放行来自 Cloudflare 共享地址段的流量。

[Authenticated Origin Pulls](https://developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/) 可以通过双向 TLS 验证 Cloudflare 到源站的连接，进一步增加防护层次。

支持相关功能的源站还可以[验证 Access JSON Web Token](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/authorization-cookie/validating-json/)，包括其签名、签发者、过期时间以及预期接收该令牌的应用。这是确保请求已经通过 Access 身份认证的最佳方式。

如果你不确定从哪里开始，可以[咨询 Cloudflare 专家](https://www.cloudflare.com/contact/sase/)，审查你的架构，确定哪些控制措施适合你的环境。

## 保护面向 Web 的身份认证服务

Cloudflare Access 决定谁可以访问 Web 应用；Cloudflare 的 [Web 应用防火墙（WAF）](https://developers.cloudflare.com/waf/)则检查发往同一个受保护服务的 HTTP 请求。WAF 在任何 Access 身份认证检查之前运行，因此你可以彻底拦截已知恶意 IP、特定国家、自治系统编号（ASN），或具有其他你明确认为不应访问应用的属性的请求。防火墙会持续检查所有请求，也能识别出那些已经通过身份认证、却试图利用已知 CVE 特征发动攻击的用户。

除了托管的 CVE 特征规则，[WAF Attack Score](https://developers.cloudflare.com/waf/detections/attack-score/) 还利用机器学习，识别那些与已知规则不完全匹配、但可能具有恶意的请求。管理员可以在规则中使用这一评分，拦截请求或要求请求方通过验证。WAF 攻击评分以 Cloudflare 全球网络流量中的观测数据为基础，Cloudforce One 的调查也会为 [Cloudflare 共享威胁情报平台](https://blog.cloudflare.com/cloudflare-threat-intelligence-platform/)提供情报。

订阅了 Cloudforce One 威胁情报服务的客户，还可以[在 WAF 策略中使用实时威胁指标](https://blog.cloudflare.com/realtime-threat-intel-waf-rules/)。这样，在检查请求本身之外，还能结合关于已知恶意来源的背景信息。

减少暴露面，并不能清除已经在系统中站稳脚跟的攻击者。说真的：如果你还没打补丁，现在就去做，并遵循 Citrix 的[疑似遭入侵时的处理指导](https://support.citrix.com/external/article/CTX694799/steps-to-take-if-netscaler-adc-is-suspec.html)。其中说明了如何保全证据、吊销凭据和证书、调查相连系统，以及重建系统或从已知安全的状态恢复。

## 将零信任扩展到桌面访问，或消除对桌面的依赖

保护 Web 身份认证可以是第一步，但不必成为最终架构。Cloudflare 的[虚拟桌面零信任架构](https://developers.cloudflare.com/reference-architecture/diagrams/sase/zero-trust-and-virtual-desktop-infrastructure/)还支持对桌面基础设施本身进行访问控制。借助 Cloudflare One Client 和私有网络连接，团队可以限制哪些人、哪些设备能够访问承载桌面流量的端点，包括现有独立计算架构（ICA）路径上的端点。这需要单独设计路由和策略；仅靠浏览器身份认证代理，并不能提供这项能力。

另一些工作流程则可以消除对 ICA 的依赖。对于受支持的 Windows 工作负载，Cloudflare 的[基于浏览器的远程桌面访问](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/use-cases/rdp/rdp-browser/)通过 Cloudflare Tunnel 将用户连接到 Windows 主机，[无需本地远程桌面客户端](https://blog.cloudflare.com/browser-based-rdp/)。如果用户只是为了打开 Web 应用才需要桌面，那么 Access 和 Browser Isolation 就可以[满足这一 Web 访问需求](https://blog.cloudflare.com/decommissioning-virtual-desktop/)，无需完整的虚拟桌面。

先保护身份认证服务和现有桌面连接。然后，替换那些仅用于 Web 访问的虚拟桌面，并将受支持的 Windows 工作流程迁移到基于浏览器的远程桌面。减少暴露面，不必等到整个桌面迁移完成。

## 确定哪些身份认证服务需要公开访问

同样的评估也适用于虚拟桌面以外的场景。员工门户或管理控制台可能需要让获准用户从任何地方访问，但这并不意味着它需要接受来自所有人的应用请求。

并非每个身份认证页面都能要求访问者事先拥有员工身份。面向消费者的登录页面，以及建立初始会话所需的身份提供方端点，可能必须保持公开可访问。应当有意识地保护这些入口，并避免引入循环的身份认证依赖。关键在于，让服务是否可被访问成为经过设计的选择，而不是沿袭下来的默认状态。

从一个需要限制访问的服务开始。梳理通往它的 Web 和非 Web 路径，然后在请求到达其身份认证入口之前作出访问决策。在扩大部署范围之前，先在自己的环境中验证这道边界：

1. 在未登录的情况下请求受保护的主机名。确认请求止步于 Cloudflare，没有到达源站。
2. 使用不属于获准群组的测试账户登录。确认该账户仍然无法访问受保护的应用。
3. 测试源站直连地址和其他主机名。确认它们都无法绕过访问策略。
4. 以获准用户的身份登录。确认身份认证、桌面启动和会话重连仍然正常。

检查相关策略日志、防火墙日志和源站日志，不要只凭浏览器显示的结果作判断。当路由、身份或故障切换设置发生变化时，重复这些检查，避免下一次配置更新悄悄重新打开你已经关闭的通路。

## 将各项控制措施结合起来

完整架构会将访问决策、Web 检查和可用性控制放到 NetScaler 之前，同时限制到源站的连接。公网源站路径和 Tunnel 路径是两种可选方案。原生客户端的桌面连接仍然独立，也可以选择在这部分流量外围增加一层零信任网络访问控制。

![](assets/diagram-32658370de9a.webp)

## 开始使用

[咨询 Cloudflare 专家](https://www.cloudflare.com/contact/sase/)，为你的环境规划合适的控制措施。[下载解决方案简介](https://cf-assets.www.cloudflare.com/dzlvafdwdttg/1kcuN1xf5hTA5i4BxQRhix/450f5f5cdf526b661417291406df7d76/Defense-in-Depth-for-Internet-Facing-Employee-Systems_Cloudflare.pdf)，并[报名参加即将举行的网络研讨会](https://cloudflare.registration.goldcast.io/webinar/06911088-c2f1-41d0-93a9-18030d09dc4c)。

*Andrei Migia、Nathan Sackett 和 Doug Shepherd 为本文作出了贡献。*
