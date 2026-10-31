# 网站归因认领

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责与边界（Responsibility and boundary）

拥有同源的浏览器认领协议，把已同意的网站获客上下文转交给已安装的 ThinkRail 宿主。Cloudflare Pages
Functions 暴露该协议，D1 提供短时的原子认领状态。

- **公开接口：** `index.ts` 暴露普通页面的浏览器归因；目标专用的 `claim.ts` 与 `server.ts` 暴露
  不含分析的认领页与 Pages handler。这些次级入口是代码分割边界，防止普通的分析/GTM 模块进入认领页
  或 Worker bundle。服务与 D1 仓储保持内部。
- **允许的依赖：** 用于普通页面记录的网站分析外观层、Web 平台 API、Pages Functions，以及
  `ATTRIBUTION_DB` D1 绑定。
- **禁止：** 导入产品的 server/desktop/CLI 代码；PostHog 管理；账户身份；改动安装器或安装命令；
  持久化 user agent、referrer URL、Cookiebot 值或任意请求体。

## 协议（Protocol）

所有改变状态的路由都要求有效主机名为 `thinkrail.ai`；浏览器绑定还要求完全同源的 `Origin`。origin
守卫在预览与同级宿主上仍会在访问 D1 之前返回 not-found。预览部署没有 D1 绑定；该守卫不是数据
隔离边界。请求体与字符串都有界，未知字段被拒绝。

1. `POST /api/attribution/claims` 接受一个 43 字符的规范 base64url SHA-256 `challenge`，原子消耗
   聚合的 D1 创建配额，存储一个随机的 32 字节 base64url 认领 ID 加创建与过期时间，并返回
   `claim_id`、相对 `claim_url` 以及十分钟后的 `expires_at` 纪元毫秒时间。配额耗尽返回 429。
2. 已安装的宿主打开 `/attribution/claim/?id=<claim>`。加载时，静态路由首先要求 Cookiebot 的第一方
   `CookieConsent` cookie 当前记录 Marketing 授予或 `-1`（无需同意）值；否则它不读取任何存储并
   导航到 `/blog/`。在获得该授予后，它从浏览器存储读取一个经校验、未过期的 journey/获客上下文，
   并做一次绑定尝试；无效的认领 ID 或缺失/无效的上下文绝不绑定。因此，一个精心构造的认领链接可以
   绑定已同意访客的存储上下文；这是被接受的，以便该流程从不打断访客。
3. `POST /api/attribution/claims/<id>/bind` 先写者胜。它暂时存储 journey ID、归一化的首次/末次
   触点字段，以及上下文最新的桌面下载 bridge ID（若存在）。若上下文没有 bridge ID（如 CLI 认领），
   bind 会生成一个随机 bridge ID。
4. 在那次有界的绑定尝试之后，无论结果如何，该路由都把 location 替换为 `/blog/`；当认领 ID 或
   上下文无效或不可用时它也会导航到那里。认领文档不初始化任何 PostHog 或 GTM 加载器，也不发出
   浏览器分析事件；其 no-referrer 策略避免 referrer 泄漏，而
   `Content-Security-Policy: frame-ancestors 'none'` 与 `X-Frame-Options: DENY` 防止点击劫持。
5. `status` 与 `redeem` 接受一个规范的 32 字节 base64url `verifier` 并派生其 SHA-256 challenge。
   经 verifier 鉴权的 status 仅供参照。Redeem 用 `DELETE … RETURNING` 原子消耗一行已绑定、未过期的
   记录，并返回 journey ID、bridge 与归一化的获客上下文。重放、错误的 verifier 与未知 ID 共享同一
   个 not-found 响应；经鉴权但未绑定的认领是 pending，经鉴权但已过期的认领是 gone。

浏览器永远收不到 verifier，服务也从不存储它。宿主只在成功的 redeem 响应中收到 journey ID。没有
任何路由启用 CORS。认领响应为 `no-store`；浏览器路由被排除在所有浏览器分析与搜索索引之外。

## 数据与投递（Data and delivery）

归一化的触点形状包含有界的 UTM `source`、`medium`、`campaign`、`content`、封闭的 referrer 类别、
纪元毫秒时间戳与策略版本 `1`。它不会把落地页传给应用。PostHog 通过把携带 `journey_id` 与
`content_key`/`$pathname` 的网站事件与 `acquisition_linked.journey_id` 关联，推导落地页/末页分析。
产品服务端维护该 schema 的严格镜像，且不导入网站代码；两份副本一起变更。浏览器存储按 journey 保存
首次/末次触点以及最新的桌面下载 bridge；新的 journey 会重置两者。带 UTM 的触点或外部搜索/社交/
推荐触点会推进末次触点并使原有 bridge 失效；无标签的内部/直接导航保留两者，而之后一次已同意的下载
会替换 bridge。一次触点是一个文档的导航：普通页面在每个已同意的 journey 中至多记录一次——在初始化
时，或在较晚授予之后的首次操作之前——因此该页面上后续操作（另一个 CTA、一次复制、一次 GitHub
点击）绝不会再次推进末次触点，也不会使在那里创建的 bridge 失效。一次重新加载、一次会重新加载的
历史导航，或另一个标签页，都是新文档，会记录自己的触点。*原因（2026-10）：* 每次操作都重新记录
同一导航，会在营销活动或外部推荐访客随后点击 GitHub、复制命令或在该页面回答下载后问卷时，丢掉某次
下载的 bridge，于是认领回退为随机 bridge。无 bridge 的事件仍然有效。已知的 Marketing 拒绝或撤回
会清除存储的上下文；未知同意保留它，但认领路由只在当前 Cookiebot cookie 记录 Marketing 授予或
`-1` 时才读取它。普通页面读取要求当前 journey；隔离的认领路由只接受末次触点之后 30 天内的已校验
上下文。不存储 `utm_term`、完整 URL、原始广告点击 ID、账户数据、IP 或 user agent。认领行在十分钟后
过期，并在兑换时或创建过程中的清理中被删除。

D1 条件更新是 bind 与 redeem 的线性化点。一张仅以 UTC 分钟桶为键的配额表原子地允许每个桶最多 1,000
次创建，且不携带任何客户端标识；更旧的桶被顺带删除。Create 在删除所有过期认领行之前消耗配额，然后
用一次条件插入强制 50,000 行的全局活跃上限；容量不足时返回 503 且不插入。不同的认领可以绑定并兑换
同一个 bridge ID，而认领 ID 加 verifier challenge 与原子删除保证认领只能兑换一次。应用代码从不记录
认领 ID、challenge、campaign 值或请求体。部署仍是单个 Cloudflare Pages 项目：固定版本的 Wrangler
`4.124.0` 校验配置并构建静态产物与 `functions/`；其预览环境用 `[]` 覆盖 `d1_databases`，而只有
生产环境绑定 `ATTRIBUTION_DB` 并应用已提交的 D1 迁移。
