# Bringing post-quantum cryptography to geographically fenced key management

> **来源**: [https://blog.cloudflare.com/post-quantum-geo-key-manager](https://blog.cloudflare.com/post-quantum-geo-key-manager)  
> **作者**: 未知 | **发布时间**: 2026-10-23T18:06:56.089000+00:00  
> **深度调研**: [查看专题调研报告](research.md)  
> **标签**: 后量子密码学, GeoKey Manager, HPKE, ML-KEM-768, X25519, 密钥管理, 数据主权

---

## AI 导读

客户需要让加密数据在全球网络中保持可用，又要限制哪些地区的服务器能使用私钥；对长期敏感数据，还需防范攻击者先收集密文、等量子计算成熟后再破解。文章介绍的新方案把客户私钥加密后全球分发，将解密所需的区域根密钥通过独立服务发给获准服务器。授权依据包括可信平台模块（TPM）提供的硬件身份、服务器位置记录和区域配置，因此地域限制依赖这些控制环节的正确性。

每个区域根密钥按服务和密钥代次派生应用密钥对，再用公钥加密客户私钥。当前方案采用混合公钥加密（HPKE）框架，结合抗量子算法 ML-KEM-768 与传统密钥协商算法 X25519；文章称，只要两者至少一个仍安全，就能维持机密性。同一服务、区域和代次共享应用密钥对，因此应用密钥泄露可能影响多个客户，区域根密钥泄露的影响更广。

系统在缺少密钥或检查失败时拒绝请求。平滑轮换允许旧代次短暂共存，强制轮换则重新加密相关客户私钥，以移除旧代次。区域根密钥保存在内核密钥环而非磁盘，重启后需重新授权获取。不过，边缘服务会在进程内缓存密钥，直到重启或缓存过期，正文没有给出撤销权限的最长生效时间。

这套机制控制私钥在哪里解密和使用；请求路由、隧道终止和流量检查仍由其他机制负责。产品处于封闭测试阶段，内容分发网络（CDN）集成仍在开发。正文提供了架构和运行取舍，但没有性能数据、测试结果或足以独立验证安全保证的实现细节。

---

## 原文整理

# Bringing post-quantum cryptography to geographically fenced key management

![](assets/diagram-166b36433014.webp)

Today, Cloudflare is launching a post-quantum geographically restricted key manager in support of our Data Localization Suite (DLS) in closed beta. This new PQ GeoKey Manager is a cornerstone of Cloudflare’s data sovereignty solutions, making sure that a customer's decryption keys are only accessible in their chosen geography. GeoKey Manager implements this by wrapping each decryption key such that it can only be decrypted by Cloudflare servers that are located in the allowed geography.

While general availability (GA) is planned for Q1 2027, today’s launch is an important milestone on our journey to support our customers’ data sovereignty goals while also migrating all Cloudflare products to [<u>post-quantum (PQ) cryptography</u>](https://blog.cloudflare.com/post-quantum-roadmap/).

Governments, financial institutions, healthcare providers, and defense organizations process highly sensitive data that can remain sensitive for years or decades after it is produced. Encryption is often required to protect this data. When classical, public-key encryption is used, adversaries do not immediately need a quantum computer today to put that data at risk: they can store the ciphertext and retain them until quantum computers can break the public key cryptography protecting it — an attack strategy known as harvest now, decrypt later.

For organizations with geographic requirements, resisting that threat is only half the challenge. At Cloudflare's scale, high availability requires replicating customer data across many servers across our global network. However, many data sovereignty regulations require that certain types of data be processed only within a given geographical region. We must therefore ensure the data is end-to-end encrypted such that only servers within the region have access to the plaintext. In this post, we describe the system that makes this work.

## The evolution of GeoKey Manager

Cloudflare pioneered the concept of geo-restricted transport layer security ([<u>TLS</u>](https://www.cloudflare.com/learning/ssl/transport-layer-security-tls/)) keys back in 2017 with [<u>GeoKey Manager</u>](https://developers.cloudflare.com/ssl/edge-certificates/geokey-manager/). This gave customers control over the locations where their private TLS keys could be unwrapped on Cloudflare’s anycast network to provide stronger technical guarantees of which region their requests are decrypted into. For instance, a customer could choose to only unwrap their private keys in India. So instead of allowing decryption across Cloudflare’s entire global network, GeoKey Manager makes sure TLS signing for each customer request happens in our India network.

GeoKey Manager evolved again in 2022 with [<u>ciphertext policy attribute based encryption</u>](https://blog.cloudflare.com/inside-geo-key-manager-v2/) (ABE). Instead of tying encrypted keys to a fixed set of data centers, ABE expressed geographic requirements using attributes such as country or region, like “region = European Union” or “region = Canada + United States + Mexico”. Newly authorized servers could receive matching decryption capabilities without rewrapping every customer key.

That flexibility created a challenge and opportunity for the post-quantum transition. No production-ready post-quantum ABE scheme is available today, and customers do not need a large number of regions. A simpler design meets the requirement and is easier to maintain and operate. Cloudflare therefore uses hybrid [<u>ML-KEM-768</u>](https://en.wikipedia.org/wiki/ML-KEM) and [<u>X25519</u>](https://en.wikipedia.org/wiki/Curve25519) protection with two kinds of keys, out of which ML-KEM-768 is a [<u>NIST</u>](https://www.nist.gov/) standard that also opens doors to highly sensitive customers. A regional root key is a symmetric secret tied to one region, and an application-specific key pair is derived from it for a single service, such as [<u>Gateway</u>](https://www.cloudflare.com/products/gateway/) or [<u>Tunnel</u>](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/). This protects private-key material against harvest now, decrypt later while preserving regional key-use boundaries.

## Global ciphertext, regional key unwrapping

In its new form, GeoKey Manager is the key foundation for Cloudflare’s data sovereignty solutions. It controls which region protected decryption keys can be used from and when. Separate workstreams determine where tunnels terminate, where traffic is inspected, how it is routed, and where plaintext processing occurs. But all of them depend on the region where a customer’s private key can be unwrapped, because TLS traffic can only be decrypted where that key is available.

During initialization, the edge key service (the component on each Cloudflare edge server that unwraps customer private keys and uses them to serve requests) verifies that it has loaded every regional root key required by its configuration. If a configured key is missing or malformed, the service does not start. After initialization, an individual request also fails closed (Cloudflare rejects it rather than risk unwrapping a key out of region) if its metadata identifies a regional root key that is not locally available. If a request for Germany was somehow trying to decrypt in Spain or France it would fail, since neither Spain nor France have the regional root key for Germany. This guarantees that a regional root key never leaves its region, no matter what.

Cloudflare needs to maintain high performance while keeping our customers’ regionalized data secure at scale. This is where the next version of GeoKey Manager comes in. We are enabling fast performance while logically separating our network by using post-quantum cryptography to restrict where customers’ private keys are unwrapped. Most of our regions will be available for customers to regionalize their workflow, and there may be product specific caveats. Customers select a region and Cloudflare manages everything else needed to unwrap their private keys within the region’s boundary.

## How GeoKey Manager ensures that keys are only available in a region

GeoKey Manager controls where wrapped private keys can be unwrapped and used on the data plane. It does not control where requests are forwarded or routed (that’s [<u>Regional Services</u>](https://developers.cloudflare.com/data-localization/regional-services/)), where tunnels terminate, or where traffic inspection happens (also Regional Services).

GeoKey Manager separates globally distributed wrapped private keys from the regional root key required to unwrap it. [<u>Quicksilver</u>](https://blog.cloudflare.com/quicksilver-v2-evolution-of-a-globally-distributed-key-value-store-part-1/), our global configuration distribution system, distributes wrapped private keys and its metadata across our network. It does not distribute regional root keys; that happens through a separate provisioning service that stores regional root keys and delivers them only to edge servers authorized for each region. Regional root keys follow a separate, controlled path. On the data plane, a node’s regional configuration, which region a node belongs to, determines which nodes can receive which regional root keys.

### Defining key hierarchy and application-specific keys

Every wrapped customer private key carries metadata identifying its service context, regional boundary, and epoch.

The service context is the Cloudflare service that uses the key, such as Tunnel or Gateway. The regional boundary is the approved set of Cloudflare data centers where the key may be unwrapped and used, such as data centers in Germany or the European Union. The epoch identifies which generation of the regional root key the key belongs to; each rotation creates a new epoch.

For example, Acme Co. might have one key for Tunnel in Germany (acme0001) and another for Gateway in Germany (acme0002). Road Runner Inc. might have one key for Tunnel in Germany (runner0001) and another for Gateway in the European Union (runner0002).

![](assets/diagram-c83270ad2b47.webp)

Keys with the same service context, regional boundary, and epoch use the same application-specific key pair. Therefore, compromising the Tunnel-Germany application-specific private key could expose both acme0001 and runner0001, but not the Gateway keys. Compromising the Germany regional root key could affect every application-specific key derived from that root for the same epoch, including acme0001, acme0002, and runner0001.

In the current architecture, the application-specific key pair is asymmetric. The public component wraps the customer's private key material, while the private component is required to unwrap it. The resulting wrapped private key is distributed with its service context ([<u>Access</u>](https://www.cloudflare.com/products/access/), Gateway, Tunnel, [<u>Cloudflare One client</u>](https://developers.cloudflare.com/cloudflare-one/team-and-resources/devices/cloudflare-one-client/)), region, and epoch.

### Provisioning regional root keys

![](assets/diagram-cb3f0294796e.webp)

When Cloudflare/Admin configures a region for a CF1 use case, key management service (which also derives application-specific key pairs and wraps customer private keys) creates a regional root key and associates it with a regional boundary and epoch and sends it to provisioning service for storage and distribution which under the hood uses Vault, which is managed by us on-premises.

### Authorizing edge servers for a region

![](assets/diagram-ff1d580d57e2.webp)

Before an edge server can receive a regional root key, provisioning service evaluates three inputs:

1. [<u>Trusted Platform Module</u>](https://en.wikipedia.org/wiki/Trusted_Platform_Module) (TPM) proves that the server possesses a registered hardware identity
2. [<u>Zinc</u>](https://blog.cloudflare.com/redefining-fleet-management-at-cloudflare/), Cloudflare's server inventory system, maps that identity to a data center and location
3. Provisioning service determines which regional root keys the server may access

Zinc updates its membership records for each edge server. This asynchronous process adds new servers as the network grows and removes servers when they are decommissioned. As Cloudflare performs maintenance on data centers in Germany, the definition of which data centers belong in Germany changes in Zinc. That definition is swiftly propagated through our network with Quicksilver so that our edge is up to date. The reverse also works. As we add capacity to our network, the definition of Germany, or any other region, is updated in Zinc and that is also swiftly propagated through our network with Quicksilver.

Zinc also associates a registered hardware identity with a server and physical location, which supports [<u>FIPS 140-3</u>](https://en.wikipedia.org/wiki/FIPS_140-3) compliance. TPM-backed identity, server inventory data, and regional configurations determine which region an edge server belongs to and what regional root keys it can access.

Each edge server calls the control plane’s provisioning service, and provides its registered hardware identity. After verifying the server’s hardware identity and regional membership, the provisioning service issues a token that permits the server to retrieve only the regional root keys authorized for its location. Each edge server receives its token and uses it to fetch the regional root keys.

Authorized edge servers load regional root keys into the [<u>kernel keyring</u>](https://kernel-internals.org/crypto/keyring/) rather than persisting them on disk. Process level access controls limit which services can use them. Because this memory is volatile, a machine restart removes the keys and the server must authenticate and provision them again. This adds operational complexity in exchange for reducing persistent key material which could be an attack vector for adversaries.

### Wrapping private keys for global distribution

When the key-management service receives a customer's private key material with a service context and regional boundary, it will use the request’s regional boundary to get the regional root key and its latest configured epoch. It will then derive the corresponding asymmetric application key pair and use its public component to wrap the private key. The result is a wrapped private key and its metadata.

Quicksilver will then distribute the wrapped private key and metadata across our network. Since the wrapped key requires the specified region to decrypt, global wrapped key distribution does not grant edge servers global access to the private key because unwrapping still requires the corresponding regional root key and service context.

![](assets/diagram-b615aff0e07a.webp)

### Unwrapping using application-specific keys on the data plane

![](assets/diagram-23f26d5ce33d.webp)

## Why we chose this cryptographic design

### Symmetric roots with asymmetric key wrapping

Regional root keys are 256-bit symmetric secrets. GeoKey Manager derives application-specific keys from each regional root key. For the current Cloudflare One path, the control plane’s key management service derives an application-specific [<u>Hybrid Public Key Encryption</u>](https://blog.cloudflare.com/hybrid-public-key-encryption/) (HPKE) key pair.

### HPKE with hybrid post-quantum KEM

The Cloudflare One path uses HPKE, as defined in [<u>RFC 9180</u>](https://www.rfc-editor.org/info/rfc9180). Its key-encapsulation mechanism (KEM) combines [<u>ML-KEM-768</u>](https://en.wikipedia.org/wiki/ML-KEM) with [<u>X25519</u>](https://cryptography.io/en/latest/hazmat/primitives/asymmetric/x25519/), while [<u>HKDF</u>](https://datatracker.ietf.org/doc/html/rfc5869)-SHA-256 provides key derivation and AES-256-GCM provides authenticated encryption. Combining post-quantum and classical components preserves confidentiality as long as at least one component remains secure.

### Designing for rotation

The validated design supports two types of key rotation: soft rotation and hard rotation.

A soft rotation creates a new epoch for the regional root key while retaining controlled access to a limited number of previous regional root-key epochs — let’s say one previous generation, as an example. The customer’s new private-key material is wrapped using the latest epoch, while older wrapped private keys are still usable during the transition. This is a similar concept to when a credit card company allows you to use a previous version of your credit card for recurring payments from an authorized retailer. In this case, the number of epochs we support determine how many generations of previously wrapped private keys — credit cards — are usable. This allows customers to preserve their service uptime while they transition.

A hard rotation instead rewraps affected customers’ wrapped private keys under a replacement epoch so access to the previous epoch can be removed. In this case, it would be like the credit card company not allowing you to use the previous credit card for a transaction under any circumstances.

### Security and availability tradeoffs

Failing closed is a very intentional design choice. Failing open would mean unwrapping a customer’s private key outside its allowed region rather than rejecting the request. We never do that: if any required check fails (service context, region, epoch, etc.), the request is rejected. At startup, each edge’s key service loads every configured regional root key from its Linux kernel keyring. Missing or malformed root keys prevent the edge key service from initializing. Once running, the edge key service uses in-memory copies so that it can’t be tampered with, preventing out of region or cross-region unwrapping. In practice, an edge server serves requests for a region only once its configuration and key provisioning are complete.

The trust boundary (all services that can access the regional root key) includes the key-management service, the provisioning service, and authorized edge key services because each can access regional root keys. The key-management service uses them to derive application-specific keys, the provisioning service stores and distributes them, and edge key services use locally authorized copies of them. Quicksilver remains outside this boundary because it receives only wrapped private keys and associated metadata.

## Operating GeoKey Manager

Cloudflare’s network is constantly changing. As servers enter or leave service, Cloudflare’s fleet inventory updates regional membership. This means the list of data centers that make up a region like Spain is not static. GeoKey Manager’s provisioning layer refreshes that membership several times each day, granting or removing access to regional root keys as the network changes. This process is separate from key rotation and requires no customer action.

When regional membership changes, the provisioning service immediately synchronizes the server’s Linux kernel keyring with the latest authorized regional root keys. The edge key service caches these keys in process memory until it restarts or the cache expires. It then reloads the current authorized set from the kernel keyring.

## What’s next: Cloudflare One integration and CDN

GeoKey Manager is the basis for an improved Regional Services offering for Cloudflare One; these new capabilities will be announced in a few months.

Older versions of GeoKey Manager already provide geographic controls for private keys associated with CDN certificates. We are developing a separate integration that will bring this new post-quantum GeoKey Manager described here to that existing CDN capability. We will share testing and transition details when that work is ready.

## Getting started

If you are a current DLS customer who is regionalizing CDN traffic, you can register intent to test this feature before it’s generally available for CDN traffic [<u>here</u>](https://forms.gle/GwBH2JKgF65QBpoAA).

Thank you to everyone who made this product a reality. It was a huge effort across the Data Localization Suite, Cloudflare One, and Cloudflare Research teams.
