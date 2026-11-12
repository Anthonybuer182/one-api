# tokenhub 完整方案设计（v4）

**一句话**：基于 pi 的客户端 → 钉钉扫码登录 → one-api 自动建用户并绑定该用户专属的 TokenHub API Key → 客户端拿 one-api 令牌对话，one-api 用用户自己的 TokenHub Key 转发上游。

---

## 修订记录

**v4 修订**（驳回两处 v3 论断，补齐限流）：

- P2-9 更正：未知分组倍率 `GetGroupRatio` 查不到时**回退为 1**（不是 0，见 `relay/billing/ratio/group.go` 的 `GetGroupRatio`），内部计费正常；代价是每个请求打一条 `SysError("group ratio not found: ...")`
- P2-9 补充：不要为了消日志把 `user-*` 批量加进「分组倍率」——`controller/group.go` 的 `GetGroups()` 直接返回 `GroupRatio` 的 key，后台渠道编辑页的分组下拉会被上百个内部组名污染（详见第 8 节脚注）
- P1-6 更正：one-api 现有 OAuth 路由全是静态段（`/oauth/github`、`/oauth/lark` 等，在 `router/api.go` 的 `SetApiRouter()` 里逐个注册），**不存在** `:provider` 通配，新增静态路由无 gin 冲突，v3 的兼容兜底作废
- 新增：`/callback` 与 `/exchange` 都要挂 `middleware.CriticalRateLimit()`，与现有 OAuth 路由一致；callback 每次会向钉钉发起 2 次 HTTP 调用，不限制等于对外开放一个可刷的上游调用入口
- 新增：`TOKENHUB_UPSTREAM_URL` 明确填**主机根地址、不带 `/v1`**（one-api 的 OpenAI 渠道自己拼 `/v1/chat/completions`，源码 `relay/adaptor/openai/helper.go` 的 `GetFullRequestURL` 对 OpenAI 类型是 `baseURL + requestURL` 直拼）；它与客户端 `toAuth()` 里带 `/v1` 的 `baseUrl` 是两个不同的值。行号引用改为函数名（`SetApiRouter()`），避免上游改行号后失效

**v3 修订**（补上 v2 中会导致流程跑不通的遗漏，均对照 one-api main 分支源码核验）：

- P0-1 建渠道必须调用 `Channel.Insert()`，它会同步写 `abilities` 表；直接 `DB.Create` 会导致路由永远查不到渠道
- P0-2 建完渠道必须调用 `model.InitChannelCache()`，否则内存渠道缓存（默认开启）最长要等 `SYNC_FREQUENCY` 秒才生效
- P0-3 渠道必须显式设置 `Type = channeltype.OpenAI`（1）与 `Status = ChannelStatusEnabled`，否则 relay 走不进去
- P1-4 占号池 Key 与建渠道不能用 `DB.Transaction` 包（`AddAbilities` 走全局 DB，事务不生效），改为「原子占 Key → 建渠道 → 失败补偿归还」
- P1-5 明确兑换接口如何取回 default 令牌：`Token.Key` 是 `char(48)` 明文且不带 `sk-` 前缀
- P1-6 路由统一到 `/api/oauth/dingtalk/*`，明确方法（v2 中 GET/POST 与根路径 `/oauth/dingtalk` 自相矛盾）
- P2-7 用户名改为 `dt_` + 8 位随机串，避免 `GetMaxUserId()+1` 在并发首登时撞唯一索引
- P2-8 Redis 开启时，设置分组后需删除 `user_group:{id}` 缓存键
- P2-9 补充：未知分组倍率与日志噪音问题（v4 已更正：回退为 1，不是 0）

**v2 已修正且 v3 保持**（源码确认）：

- 分组绑在 `User.Group`（`varchar(32)`），`Token` 无 Group 字段；路由中间件 `Distribute()` 取 `CacheGetUserGroup(userId)` — `model/user.go:45`、`model/token.go`、`middleware/distributor.go`
- `user.Insert(ctx, inviterId)` 会把 `Quota` 设为 `QuotaForNewUser` 并自动创建 default 令牌（`ExpiredTime:-1, RemainQuota:-1, UnlimitedQuota:true`）— `model/user.go:113`、`model/user.go:133-148`
- 额度必须回填：`PreConsumeTokenQuota` 里有 `if userQuota < quota { return "用户额度不足" }`
- 换头发生在 one-api 内部：`SetupContextForSelectedChannel` 里 `Authorization: Bearer {channel.Key}`
- 302 只回传一次性短码，真令牌经 POST 兑换取得

---

## 1. 需求与约束

| 项 | 内容 |
|---|---|
| 用户 | 你钉钉组织内的成员（企业内部应用，不支持跨组织） |
| 登录方式 | 钉钉扫码，登录即开通，无需手动填 Key |
| 模型上游 | 腾讯 TokenHub（OpenAI 兼容端点） |
| 关键决策 | 每个用户绑定**自己的** TokenHub Key（走 TokenHub 原生独占额度/统计） |
| 已知限制 | TokenHub 无自动创建 Key 的接口 → 号池预生成，定期人工补货 |

---

## 2. 角色与三种凭据

**角色（3 个程序）**：

```
pi 客户端（用户电脑） ←→ one-api 改造版（你的服务器） ←→ TokenHub（腾讯云）
                              ↕
                        钉钉开放平台（扫码认证）
```

**三种 Key，谁持有谁：**

```
Key① one-api 令牌        sk-xxx        签发方: one-api          存放: 客户端 auth.json
Key② TokenHub 真 Key     sk-tp-NNN     申请方: 你在控制台批量创建   存放: one-api 渠道配置（DB）
Key③ 钉钉 AppSecret      secret888     签发方: 钉钉开放平台        存放: one-api 环境变量

用户只见 Key①。Key②③ 永不出服务器。
```

**映射关系的表达（one-api 分组机制，不写映射表）**：

```
张三: users.group = user-3   ↔  渠道「user-3-tokenhub」group = user-3（内填 sk-tp-001）
李四: users.group = user-7   ↔  渠道「user-7-tokenhub」group = user-7（内填 sk-tp-002）
```

约束确认：`User.Group` 与 `Channel.Group` 都是 `varchar(32)`，`user-{int}` 不会超长；`Channel.Group` 支持逗号分隔多分组，但本方案每人只用单值。

---

## 3. 总体架构

```
┌───────────────────────────────────────────────────────┐
│ pi 客户端                                              │
│  ┌───────────────┐        ┌────────────────────────┐  │
│  │ TokenHubOAuth │        │ LLM 请求                │  │
│  │ (OAuthAuth)   │        │ baseUrl=本服务域名/v1    │  │
│  └──────┬────────┘        │ Bearer sk-xxx (Key①)   │  │
│  ┌──────▼────────┐        └───────────┬────────────┐  │
│  │ loopback 回调  │        ┌───────────▼────────────┐  │
│  │ 127.0.0.1:rp  │        │ auth.json              │  │
│  └───────────────┘        │ {type:oauth,access:sk} │  │
└─────────┬─────────────────────────────┬─────────────────┘
          │浏览器                        │ Bearer Key①
          ▼                            ▼
┌───────────────────────────────────────────────────────┐
│ one-api 改造版  https://tokenhub.example.com           │
│                                                       │
│ 【新增】钉钉登录模块          【现成】                   │
│  ├ GET  /api/oauth/dingtalk    ├ /v1/chat/completions │
│  ├ GET  /api/oauth/dingtalk/   ├ 令牌系统              │
│  │        callback             ├ 渠道路由（按 group）   │
│  ├ POST /api/oauth/dingtalk/   ├ 额度/计费/日志         │
│  │        exchange             └ 管理后台              │
│  ├ 换 unionId → 建用户                                 │
│  ├ 号池占 Key → 建渠道(Type=OpenAI) → InitChannelCache │
│  ├ 回填额度 + 设用户分组                                │
│  └ 302 回客户端带一次性短码                             │
│                                                       │
│ 数据库（现成 + 变更）                                    │
│  users / tokens / channels / abilities（现成）          │
│  users 表：新增 dingtalk_id 列（照 lark_id 模式）        │
│  oauth_state   state → redirect_uri / 短码（新增）       │
│  key_pool      号池: tokenhub_key | status（新增）       │
└───────────────────────────┬───────────────────────────┘
                            │ Bearer sk-tp-NNN (Key②)
                            ▼
                    TokenHub（OpenAI 兼容端点）
                            │
                    腾讯云按 Key 统计/独占额度
```

注意架构里的隐藏依赖：**路由数据源是 `abilities` 表 + 进程内 `group2model2channels` 缓存**，不是 `channels` 表本身。所以建渠道必须同步建 Ability 并刷新缓存（见 4.1）。

---

## 4. 登录流程（逐请求）

```
客户端           浏览器           one-api             钉钉
  │①开临时回调服务器 │                │                  │
  │ 127.0.0.1:53721 │                │                  │
  │ path=/oauth/callback/<uuid>      │                  │
  │②notify 打开浏览器│               │                  │
  │───────────────►│ GET /api/oauth/dingtalk           │
  │                │  ?redirect_uri=http://127.0.0.1:53721/oauth/callback/<uuid>
  │                │               │ 校验 redirect_uri 前缀必须是 http://127.0.0.1:
  │                │               │ 生成 state，存 state → redirect_uri（5min）
  │                │◄── 302 ───────│                  │
  │                │   login.dingtalk.com/oauth2/auth │
  │                │   ?client_id=AppKey               │
  │                │   &redirect_uri=本服务域名/api/oauth/dingtalk/callback
  │                │   &response_type=code&scope=openid│
  │                │   &prompt=consent&state=xyz789    │
  │                │──── 扫码确认 ──────────────────────►│
  │                │◄── 302 ?authCode=xxx&state=xyz789 ─│
  │                │  （失败时 ?error=yyy&state=xyz789，回调必须处理 error 分支）
  │                │───►│ /api/oauth/dingtalk/callback  │
  │                │    │③校验 state（存在+未过期，用后即删）
  │                │    │④POST api.dingtalk.com         │
  │                │    │  /v1.0/oauth2/userAccessToken │
  │                │    │  {clientId, clientSecret,     │
  │                │    │   code, grantType:"authorization_code"}
  │                │    │⑤GET /v1.0/contact/users/me    │
  │                │    │  header: x-acs-dingtalk-access-token
  │                │    │  → unionId, nick              │
  │                │    │⑥自动开通（见 4.1）              │
  │                │    │⑦生成一次性短码（32 字符，5min）  │
  │◄── ⑧302 redirect_uri?code=shortCode ─────────────── │
  │⑨临时服务器收下 code  │                             │
  │  POST /api/oauth/dingtalk/exchange {code:shortCode} │
  │  → {token: "sk-xxx"}（短码用后即删）                 │
  │  存 auth.json，关闭临时服务器                        │
  │  「登录成功」     │                │                  │
```

> 短码放在 `code` 参数位，是因为 pi 的 `startOAuthCallbackServer` 的 `complete` 回调签名是 `(code: string) => Promise<T>`，只交出 `code`，不交出完整 URL（`packages/ai/src/auth/oauth/callback-server.ts:28`、`:100-107`）。v1 里“解析完整 URL 拿 token”的写法在现有 API 下拿不到。

### 4.1 步骤⑥「自动开通」

全部在 one-api 进程内直接调 model 层，不走 HTTP 自调用。

```
1. FillUserByDingtalkId(unionId) 命中 → 跳到第 7 步
2. 建用户
     Username    = "dt_" + random.GetRandomString(8)   // 3+8=11 ≤ validate:"max=12"
     DisplayName = 钉钉 nick
     DingtalkId  = unionId
     Role        = RoleCommonUser
     Status      = UserStatusEnabled
     user.Insert(ctx, 0)
       → 内部已完成：Quota = config.QuotaForNewUser、AccessToken/AffCode 生成、
         default 令牌创建（UnlimitedQuota=true, ExpiredTime=-1）
       → 此处 user.Id 可用
3. 回填额度与分组
     user.Quota = TOKENHUB_USER_QUOTA     // 默认 100000000 ≈ $200（QuotaPerUnit=500000）
     user.Group = fmt.Sprintf("user-%d", user.Id)
     user.Update(false)
     // Redis 开启时还需 DEL "user_group:{id}"，避免旧分组缓存（本流程通常无缓存，仍需防御）
4. 号池原子占 Key（单条 UPDATE 自带原子性，不需要事务）
     UPDATE key_pool SET status='已分配', union_id=?
       WHERE id = (SELECT id FROM (SELECT id FROM key_pool
                   WHERE status='未分配' ORDER BY id LIMIT 1) t)
         AND status='未分配'
     影响行数 0 → 号池空：登录失败，返回「额度号池耗尽，请联系管理员」，并发告警（webhook 可选）
5. 建渠道
     channel := model.Channel{
       Type:    channeltype.OpenAI,          // 必填，否则 relay 不认
       Status:  model.ChannelStatusEnabled,
       Name:    fmt.Sprintf("user-%d-tokenhub", user.Id),
       BaseURL: &upstreamURL,                // 主机根地址，不带 /v1（one-api 自己拼 /v1/…）
       Key:     poolKey,                     // sk-tp-NNN
       Models:  TOKENHUB_MODELS,             // 逗号分隔，必须与 TokenHub 模型名一致
       Group:   user.Group,
     }
     channel.Insert()                        // 关键：DB.Create + AddAbilities()
     失败 → 补偿：把 key_pool 该行改回 status='未分配'、清空 union_id，返回 500
    // 不要用 DB.Transaction 包这一步：AddAbilities 内部走全局 DB，事务不会回滚 Ability
6. 刷新进程内渠道缓存
     model.InitChannelCache()
     // 不做这步，MemoryCacheEnabled=true 时最长等 SYNC_FREQUENCY 秒才路由得到
7. 取回 default 令牌（新老用户都适用）
     查 DB: user_id=? AND name='default'
       → 取 token.Key（char(48)，明文，无 sk- 前缀）
     若不存在（用户删过）→ 新建 Token{UserId, Name:"default", ExpiredTime:-1,
       RemainQuota:-1, UnlimitedQuota:true, Key: random.GenerateKey()} 后 Insert()
8. 写短码：oauth_state 表新增一行（code=32 位随机串, tool='short_code',
   user_id=?, expire=now+5min）→ 302 redirect_uri?code=<shortCode>
```

### 4.2 兑换接口

```
POST /api/oauth/dingtalk/exchange
body: { "code": "<shortCode>" }

① 短码查库：存在、未过期、未使用（用后立即删）
② 取出 user_id → 取 default 令牌
③ 返回 { "token": "sk-" + token.Key }     // 拼前缀，DB 里不带 sk-
④ 挂 CriticalRateLimit()，并保证短码熵 ≥ 32 字符
⑤ /callback 同样挂 CriticalRateLimit()：它每次会向钉钉发起 2 次 HTTP 调用
   （userAccessToken + users/me），不限制等于开放一个可被刷的上游调用入口
```

---

## 5. 对话流程（逐请求）

```
客户端                                one-api                TokenHub
  │ POST /v1/chat/completions           │                      │
  │ Bearer sk-xxx (Key①)                │                      │
  │ {"model":"deepseek-v3",...}         │                      │
  │────────────────────────────────────►│                      │
  │                    ①TokenAuth: 去 sk- 前缀 → ValidateUserToken
  │                        （Token.Key 是纯字母数字，不会被 '-' 截断）
  │                    ②Distribute: CacheGetUserGroup(张三)=user-3
  │                        → CacheGetRandomSatisfiedChannel(user-3, deepseek-v3)
  │                        → 命中渠道 user-3-tokenhub
  │                    ③SetupContextForSelectedChannel:
  │                        Authorization 换成 Bearer sk-tp-001
  │                        BaseURL 换成 TokenHub 端点
  │                                     │─────────────────────►│
  │                                     │      腾讯云认 Key②    │
  │◄──────────── SSE 流式透传 ──────────────────────────────────│
  │                    ④PreConsume/PostConsume：扣内部额度并记账
```

理解要点：换头在 one-api 内部完成；客户端两跳都是 Key①，Key② 只出现在 one-api→TokenHub 这一跳，所以腾讯云控制台按 Key 看到的就是各人独立用量。

---

## 6. 开发清单

### 6.1 one-api 侧（Go）

| 文件 | 内容 | 规模 |
|---|---|---|
| `model/user.go`（改） | `User` 加 `DingtalkId string gorm:"column:dingtalk_id;index"`（照 `LarkId` 模式）+ `FillUserByDingtalkId()` + `IsDingtalkIdAlreadyTaken()` | ~30 行 |
| `model/key_pool.go`（新） | 号池表 + 原子占用 / 归还 / 补货 | ~60 行 |
| `model/oauth_state.go`（新） | state 映射 + 一次性短码（同一张表，`type` 字段区分） | ~50 行 |
| `controller/auth/dingtalk.go`（新） | 登录入口 + 回调 + 自动开通 + 兑换 | ~280 行 |
| `router/api.go`（改） | 3 条路由（见下），均挂 `middleware.CriticalRateLimit()`，与现有 OAuth 路由同级 | 3 行 |
| `main.go`（改） | 读钉钉 / TokenHub 环境变量 | ~5 行 |

路由（统一放在现有 `apiRouter` 下，即 `/api` 前缀）：

```
GET  /api/oauth/dingtalk              登录入口（挂 CriticalRateLimit）
GET  /api/oauth/dingtalk/callback     钉钉回调（挂 CriticalRateLimit）
POST /api/oauth/dingtalk/exchange     短码换令牌（挂 CriticalRateLimit）
```

> 与现有 OAuth 路由平级添加即可：one-api 的 `/oauth/github`、`/oauth/oidc`、`/oauth/lark`、`/oauth/wechat` 都是静态路径（在 `router/api.go` 的 `SetApiRouter()` 里逐个注册），没有 `:provider` 通配，不存在 gin 路由冲突。

环境变量：

```
DINGTALK_CLIENT_ID / DINGTALK_CLIENT_SECRET
TOKENHUB_PUBLIC_URL=https://tokenhub.example.com     // 对外域名，用于拼 callback / 兑换地址
TOKENHUB_UPSTREAM_URL=https://tokenhub.example.com   // 渠道 BaseURL：主机根地址，不要带 /v1，末尾也不要带 /
                                                     // one-api 自己会拼 /v1/chat/completions
TOKENHUB_MODELS=deepseek-v3,kimi-k2,...              // 必须与 TokenHub 模型名一致
TOKENHUB_USER_QUOTA=100000000                        // 新用户内部额度
```

> `TOKENHUB_UPSTREAM_URL` 与客户端 `toAuth()` 里的 `baseUrl` 不是同一个值：渠道 BaseURL 填根地址（不带 `/v1`），填成 `.../v1` 会请求到 `/v1/v1/chat/completions`，末尾多一个 `/` 会拼出 `//v1/...`（多数网关不容忍）；客户端 `baseUrl` 用 `${TOKENHUB}/v1`。
> Go 侧构造渠道时 `BaseURL: &upstreamURL` 必须取**变量地址**（`&os.Getenv(...)` 编译不过），所以在 `main.go` 读完环境变量后存成包级变量。

### 6.2 客户端侧（TypeScript，1 个新文件）

`tokenhub-oauth.ts`，照抄 `packages/ai/src/auth/oauth/openrouter.ts`：

```
login()
  startOAuthCallbackServer({ providerName, path: `/oauth/callback/${uuid}`, complete })
  complete(code) → POST {TOKENHUB}/api/oauth/dingtalk/exchange {code} → 拿到 sk-xxx
  interaction.notify({ type: "auth_url", url: `${TOKENHUB}/api/oauth/dingtalk?redirect_uri=${cb.redirectUri}` })
  → { type:"oauth", access:"sk-xxx", refresh:"", expires: MAX_SAFE_INTEGER }

refresh(credential) → 原样返回（令牌长效）
toAuth(credential)  → { apiKey: credential.access, baseUrl: `${TOKENHUB}/v1` }
```

### 6.3 明确不用写的

OpenAI 兼容网关、渠道路由、额度计费、管理后台（one-api 现成）；对话流式（pi 现成）；建令牌（`user.Insert` 已建 default）；TokenHub 侧零开发。

---

## 7. 配置清单（人工操作）

### 7.1 钉钉开放平台（一次性）

```
① open-dev.dingtalk.com 登录（企业管理员 / 子管理员）
② 应用开发 → 企业内部应用 → 钉钉应用 → 创建应用
③ 基础信息 → 凭证：拿 Client ID(AppKey) / Client Secret
④ 开发配置 → 权限管理：申请 Contact.User.Read
⑤ 开发配置 → 安全设置 → 重定向 URL：
     https://tokenhub.example.com/api/oauth/dingtalk/callback
     （本地调试另加 http://localhost:xxxx）
⑥ 应用发布 → 版本管理与发布 → 创建新版本 → 发布（未发布扫码不生效）
```

### 7.2 腾讯 TokenHub（周期性）

```
① 控制台开通服务，记录 OpenAI 兼容端点（TokenHub 侧没有公开的“自动创建 Key”接口）
② API Key 管理 → 批量创建（每次 1~10 个）→ 导入 key_pool 表（status='未分配'）
③ 如需要按 Key 独占额度/独立 TPM，开通 Token Plan
④ 号池余量低于阈值时重复 ②（唯一的人工环节）
```

### 7.3 部署（一次性）

```
2C4G 起，Docker 跑改造版 one-api + Caddy/Nginx 做 HTTPS
首次启动：改 root 密码 → 导入号池 → 冒烟（阶段 1 的 curl 验证）
建议生产用 MySQL：号池原子 UPDATE 的派生表写法在 SQLite 也能跑，但并发语义 MySQL 更稳
```

---

## 8. 安全设计

| 风险 | 防护 | 实现位置 |
|---|---|---|
| 伪造 redirect_uri 窃取令牌 | 只允许 `http://127.0.0.1:` 前缀 | dingtalk.go 入口校验 |
| state 伪造 / 重放 | 随机 16 字节、5 分钟过期、用后即删 | oauth_state 表 |
| authCode 重放 | 钉钉原生一次性 + 5 分钟 | 钉钉侧保证 |
| 短码重放 / 泄露 | 32 字符随机、一次性、5 分钟过期；兑换接口 `CriticalRateLimit()` | oauth_state + exchange |
| callback 被刷（放大打钉钉 API） | `/callback` 挂 `CriticalRateLimit()` | router/api.go |
| 真令牌出现在 URL | 已消除：302 只带短码，令牌走 POST 返回 | 回调 / 兑换 |
| 令牌含 `-` 被截断 | `Token.Key` 用 `random.GenerateKey()`（纯字母数字） | 建令牌时 |
| Key① 泄露 | 后台一键吊销；`TOKENHUB_USER_QUOTA` 兜底 | 现成功能 |
| Key② 泄露 | 只存服务端 DB；腾讯控制台重置后同步 key_pool | 运维流程 |
| AppSecret 泄露 | 仅环境变量，不进代码 / git | 部署规范 |
| 号池被刷空 | 新用户额度封顶 + 号池余量告警 | key_pool + 监控 |
| 渠道残留导致脏路由 | 回收必须走 `channel.Delete()`（会删 Ability） | 运维流程 |
| 传输窃听 | 全链路 HTTPS（对外域名 + loopback 本机） | Caddy |

> 关于「分组倍率」的脚注：`user-*` 不在 `GroupRatio` 里，`GetGroupRatio` 回退为 1（计费不受影响），并每请求打一条 `SysError("group ratio not found: ...")`。不要为了消日志把 `user-*` 批量加进倍率配置——`controller/group.go` 的 `GetGroups()` 直接返回 `GroupRatio` 的 key，后台渠道编辑页的分组下拉会被上百个内部组名污染。要么接受日志噪音，要么把 `relay/billing/ratio/group.go` 里这个分支降为 debug 级别。

---

## 9. 实施计划

| 阶段 | 内容 | 验收标准 | 工时 |
|---|---|---|---|
| 0 | 钉钉应用 + TokenHub 开通 | 拿到 AppKey/Secret、端点、1 个测试 Key | 0.5 天 |
| 1 | 原版 one-api 部署 + 手动配渠道（Type=OpenAI，group 手工对齐测试用户） | curl 带测试令牌调 `/v1/chat/completions` 通；把用户 group 改成渠道 group 后路由正确 | 0.5 天 |
| 2 | 客户端手填 apiKey（ApiKeyAuth）先通对话 | 客户端正常对话 | 0.5 天 |
| 3 | 钉钉登录模块 + 号池 + users 加列 | 扫码后后台出现：用户(group=user-N, 额度已回填)、渠道(Type=OpenAI, Ability 已生成)、default 令牌；**登录后立即对话成功，无需等待**；两终端并发首登拿到不同 Key；号池空时提示明确 | 2 天 |
| 4 | 客户端 tokenhub-oauth.ts | 扫码 → 短码换令牌 → auth.json 出现 sk-xxx → 对话成功 | 0.5 天 |
| 5 | 边界与安全测试 | 吊销令牌 401；伪造 redirect_uri 被拒；短码二次使用被拒；Redis 场景分组即时生效；TokenHub 控制台按 Key 见各人用量 | 1 天 |

阶段 1-2 完成即有一条可用链路（手动发令牌兜底），3-4 只增强体验。

---

## 10. 运维与演进

**日常运维**：

```
- 号池监控：status='未分配' 数量 < 5 时告警（登录时号池空 → 明确报错 + webhook）
- 上游欠费：从渠道错误日志识别持续 402/欠费 → 禁用渠道并通知本人；Key 可回收
- 离职处理：停用用户 → channel.Delete()（删渠道与 Ability）→ InitChannelCache()
            → key_pool 该行改回未分配、清 union_id
- 新模型上线：改 TOKENHUB_MODELS → 批量更新所有渠道 Models → InitChannelCache()
- 事故排查顺序：abilities 表是否有该 group+model 行 → 进程缓存是否已刷新 → 渠道 status
```

**后续演进（本期不做）**：

```
- 外部用户 → 钉钉 ISV 第三方应用（服务端代码基本复用，主要是审核流程）
- 短效令牌 + refresh（安全要求提高时，refresh() 改为调 TokenHub 换新令牌）
- 多上游故障切换 → 同 group 配多条渠道（one-api 原生支持，按 priority 挑）
- 部门级共享 Key → group 按部门命名，一个部门共用一个 TokenHub Key（省号池）
```

---

## 11. 全链路复述（自检）

```
登录（一次）：
  扫码 → 钉钉证明身份 → one-api 建用户（回填额度、group=user-N）
  → 原子占号池 Key → 建渠道(Type=OpenAI) → Channel.Insert 生成 Ability
  → InitChannelCache 让路由立刻可见
  → 取 default 令牌 → 一次性短码回 loopback → 客户端兑换得到 sk-xxx 存盘

对话（每次）：
  客户端带 sk-xxx → 验票 → 按 User.Group 命中专属渠道 → 换头成 Key②
  → 转发 TokenHub → 透传回复 → 内部记账
```

v4 定稿。可落地的最小改造集：`model/user.go` 加列、`key_pool`、`oauth_state`、`dingtalk.go`、`router/api.go` 3 条路由（均带 `CriticalRateLimit`）、`tokenhub-oauth.ts`。
