# PRD — Jasper's Market WhatsApp Demo

> 旁注（aside note）：本 PRD 是根据现有代码（详见 `code_walk.md`）反推整理的产品需求文档，用于说明这个示例当初要演示/验证的产品能力，而非未来要新增的需求。

## 1. 背景与目的

Jasper's Market 是 Meta 官方提供的 WhatsApp Business Platform 示例，目标受众是**评估或学习 WhatsApp Business Platform 的开发者/集成商**，而不是终端消费者产品。它要回答的核心问题是：

> "一个零售商可以通过 WhatsApp 的哪几类原生消息能力，把一次性的访客互动，转化成可被追踪、有商业价值的购物触点？"

它不是一个完整的电商应用——没有真实的商品目录、购物车或支付，只演示"消息能力"本身。

## 2. 目标用户

- **主要受众**：评估 WhatsApp Cloud API 的开发者/产品经理，用于决定是否在自己的零售/客服场景中采用。
- **次要受众（在场景模拟中）**：通过 WhatsApp 联系 Jasper's Market 的购物者，期望获得促销信息、在线购物入口或菜谱灵感。

## 3. 用户故事

1. 作为一个**新访客**，我给 Jasper's Market 发一条消息（任意内容），期望看到这家店能提供哪些服务的入口。
2. 作为一个**已收到欢迎消息的用户**，我点击其中一个按钮，期望收到与该按钮主题匹配的内容（购物链接 / 当前促销 / 菜谱灵感）。
3. 作为一个**收到了促销/菜谱内容的用户**，在我确实看完（已读）之后，期望商家能再问一句"还有什么可以帮你"，而不是在我还没看到内容时就被连续打扰。

## 4. 功能范围（已实现）

| 功能 | 描述 | 对应触发 |
|---|---|---|
| 欢迎/导流消息 | 三按钮交互消息，引导用户进入三条分支 | 任意首条消息，或未识别的按钮 ID |
| 在线购物入口 | 发送带商品图片的 Utility 模板，模拟"点击去购物" | 按钮 `reply-interactive-with-media` |
| 当前促销 | 发送限时优惠模板（48 小时有效期 + 可复制优惠码 `BERRIES20`） | 按钮 `reply-offer` |
| 菜谱灵感 | 发送媒体卡片轮播模板（多张图片横向滑动） | 按钮 `reply-media-card-carousel` |
| 已读后跟进 | 用户**实际看到**（read）促销/购物/菜谱消息后，自动追问"还有什么可以帮你" | Meta 状态回调 `read`/`delivered` |
| 安全校验 | 校验所有入站 webhook 的 HMAC 签名，拒绝非 Meta 来源的请求 | 所有 `POST /webhook` |

## 5. 非目标（明确不做）

- 不做真实商品目录、库存、价格、购物车或下单流程。
- 不解析自由文本输入（不支持"我想买苹果"这种自然语言下单）。
- 不做用户身份识别或跨会话的个性化记忆——每条消息仅依据消息本身的按钮 ID 路由，没有用户档案。
- 不做多语言（英文 only，`locale: "en_US"` 硬编码在三个模板调用里）。
- 不做消息发送失败后的重试或告警机制（失败仅 `console.error` 后抛出）。

## 6. 关键交互流程

```
用户首条消息
   │
   ▼
欢迎消息（3 个按钮）
   │
   ├─ 点"Shop online"      → 发送购物 Utility 模板 → 标记待跟进(15s)
   ├─ 点"Current promo"    → 发送限时优惠模板      → 标记待跟进(15s)
   └─ 点"Get recipe ideas" → 发送菜谱轮播模板      → 标记待跟进(15s)
                                  │
                                  ▼
                    Meta 回调 delivered/read 状态
                                  │
                          标记仍有效(未过15s)?
                            ├─ 是 → 发送"还有什么可以帮你" + 3 个按钮（回到顶部循环）
                            └─ 否 → 不做任何事
```

## 6.1 调用流程（Call Flow，含具体 API 调用）

以用户点击"Current promo"按钮为例，展示从入站 webhook 到出站 Graph API 调用、再到下一轮状态回调的完整调用链：

```
WhatsApp 用户                Meta Cloud API              Jasper's Market 服务器           Redis
     │                            │                              │                          │
     │  点击按钮 "Current promo"   │                              │                          │
     ├───────────────────────────►│                              │                          │
     │                            │  POST /webhook               │                          │
     │                            │  (HMAC: x-hub-signature-256)  │                          │
     │                            ├─────────────────────────────►│                          │
     │                            │  body.entry[].changes[].value.messages[0]               │
     │                            │  { type:"interactive",                                  │
     │                            │    interactive.button_reply.id: "reply-offer" }         │
     │                            │                              │                          │
     │                            │                              │ verifyRequestSignature() │
     │                            │                              │ HMAC 比对，不通过则 throw │
     │                            │                              │                          │
     │                            │                              │ new Message(rawMessage)  │
     │                            │                              │ → message.type = "reply-offer"
     │                            │                              │                          │
     │                            │                              │ Conversation.handleMessage()
     │                            │                              │ switch → case "reply-offer"
     │                            │                              │                          │
     │                            │   POST /{phone_number_id}/messages (标记已读+输入中)      │
     │                            │◄─────────────────────────────┤                          │
     │                            │   { status:"read",                                       │
     │                            │     typing_indicator:{type:"text"} }                     │
     │                            │                              │                          │
     │   已读✓ + "对方正在输入…"    │                              │                          │
     │◄───────────────────────────┤                              │                          │
     │                            │                              │                          │
     │                            │   POST /{phone_number_id}/messages (限时优惠模板)         │
     │                            │◄─────────────────────────────┤                          │
     │                            │   { type:"template",                                     │
     │                            │     template.name:"strawberries_limited_offer",          │
     │                            │     components:[header图片, limited_time_offer,          │
     │                            │                  copy_code按钮(BERRIES20)] }              │
     │                            │                              │                          │
     │                            │   响应 { messages:[{ id: "wamid.XXX" }] }                 │
     │                            ├─────────────────────────────►│                          │
     │                            │                              │ markMessageForFollowUp() │
     │                            │                              │ Cache.insert("wamid.XXX")│
     │                            │                              ├─────────────────────────►│
     │                            │                              │                          │ SET wamid.XXX ""
     │                            │                              │                          │ EXPIRE wamid.XXX 15
     │                            │                              │                          │
     │  收到限时优惠模板消息        │                              │                          │
     │◄───────────────────────────┤                              │                          │
     │                            │                              │                          │
     │  （用户实际打开/看到消息）   │                              │                          │
     ├───────────────────────────►│                              │                          │
     │                            │  POST /webhook (status 回调) │                          │
     │                            │  value.statuses[0] =                                    │
     │                            │  { id:"wamid.XXX", status:"read",                       │
     │                            │    recipient_id:"<用户号码>" }                           │
     │                            ├─────────────────────────────►│                          │
     │                            │                              │ new Status(rawStatus)    │
     │                            │                              │ Conversation.handleStatus()
     │                            │                              │ status==="read" → 继续    │
     │                            │                              │                          │
     │                            │                              │ Cache.remove("wamid.XXX")│
     │                            │                              ├─────────────────────────►│
     │                            │                              │                          │ DEL wamid.XXX
     │                            │                              │◄─────────────────────────┤
     │                            │                              │ 返回 1 (存在且未过期)      │
     │                            │                              │                          │
     │                            │   POST /{phone_number_id}/messages (跟进消息+3按钮)       │
     │                            │◄─────────────────────────────┤                          │
     │                            │   { type:"interactive",                                  │
     │                            │     body.text:"还有什么可以帮你",                         │
     │                            │     action.buttons:[3个按钮] }                           │
     │                            │                              │                          │
     │  收到跟进消息（回到顶部循环）│                              │                          │
     │◄───────────────────────────┤                              │                          │
     │                            │                              │                          │
     │                            │  POST /webhook 总是回 200     │                          │
     │                            │  EVENT_RECEIVED（无论上面     │                          │
     │                            │  业务逻辑是否成功）            │                          │
```

**关键时序约束（附对应代码位置）：**

1. **签名校验在最前面**：`verifyRequestSignature` 跑在 body-parser 解析 JSON 之前的原始 buffer 上，签名不对直接中断，不会进入 `Conversation`。
   → 代码：`app.js:29`（挂载为 `json()` 的 `verify` 回调）、`app.js:90-106`（HMAC 比对与 `throw`）。
2. **入站路由分发**：`POST /webhook` 遍历 payload，把 `value.statuses` 和 `value.messages` 分别分发出去。
   → 代码：`app.js:55-60`（`Conversation.handleStatus`）、`app.js:62-67`（`Conversation.handleMessage`）。
3. **每次发送都先标记已读**：`GraphApi.#makeApiCall` 只要拿到 `messageId`，就先发一次"已读+输入中"调用，再发真正的业务消息——两次独立的 Graph API 调用，不是一次。
   → 代码：`services/graph-api.js:16-48`（`#makeApiCall` 内先 `typingBody` 再 `requestBody`），四个模板方法 `services/graph-api.js:72`、`103`、`161`、`200` 都经过这同一入口。
4. **`Cache.insert` 发生在拿到 Graph API 响应之后**：用的是**出站消息**的 `wamid`（即 `response.messages[0].id`），不是入站消息的 ID——这是为了在下一轮 `statuses` 回调里能对上号。
   → 代码：`services/conversation.js:84-85`（`markMessageForFollowUp` → `Cache.insert`），调用点在 `services/conversation.js:104`、`112`、`120`（三个模板分支各自调用一次）。
5. **`Cache.remove` 兼具"判断+清除"双重作用**：`DEL` 返回值 >0 表示这个 key 之前存在且没过期，据此决定是否要发跟进消息；无论结果如何，key 都已被清掉，避免后续 `delivered`/`read` 重复回调触发第二次跟进。
   → 代码：`services/conversation.js:133-150`（`handleStatus`，第 137 行过滤非 `delivered`/`read`，第 143 行 `Cache.remove`），底层实现在 `services/redis.js:40-51`（`DEL` + 返回 `resp > 0`）。
6. **15 秒 TTL 的来源**：`Cache.insert` 写入 key 后立刻 `EXPIRE 15`，这是整条跟进逻辑能成立的时间窗口假设。
   → 代码：`services/redis.js:27-38`（`insert` 方法，注释里写明"假设大多数 delivered/read 回调会在 15 秒内到达"）。
7. **Webhook 响应与业务逻辑解耦**：`POST /webhook` 对 Meta 的应答永远是 `200 EVENT_RECEIVED`，且是在所有 `forEach` 内部异步逻辑触发之后立即同步返回的（`handleMessage`/`handleStatus` 是 fire-and-forget，没有 `await`）——这意味着 Meta 看到的"处理成功"只代表"收到了"，不代表后续 Graph API 调用一定成功。
   → 代码：`app.js:45-74`（整个 `POST /webhook` handler，第 73 行 `res.status(200).send('EVENT_RECEIVED')` 与第 49-70 行的 `forEach` 之间没有 `await`/`Promise.all` 关联）。

## 6.2 入站消息详细示例（WhatsApp → 服务器）

下面是 Meta 实际发到 `POST /webhook` 的完整 payload 结构示例，对应代码里 `app.js` 解构出的字段（`value.metadata.phone_number_id`、`value.messages`、`value.statuses`）。每个外层都是同一个 envelope：

```json
{
  "object": "whatsapp_business_account",
  "entry": [
    {
      "id": "<WABA_ID>",
      "changes": [
        {
          "field": "messages",
          "value": { /* 见下面三种具体场景 */ }
        }
      ]
    }
  ]
}
```

`app.js:48-71` 就是在解这层 `entry[].changes[].value`。

### 场景 A：用户发的第一条消息（自由文本，未识别）

```json
{
  "messaging_product": "whatsapp",
  "metadata": {
    "display_phone_number": "15550001111",
    "phone_number_id": "123456789012345"
  },
  "contacts": [
    { "profile": { "name": "Jane Doe" }, "wa_id": "15551234567" }
  ],
  "messages": [
    {
      "from": "15551234567",
      "id": "wamid.HBgLMTU1NTEyMzQ1NjcVAgARGBI4...",
      "timestamp": "1750000000",
      "type": "text",
      "text": { "body": "Tap send to get started" }
    }
  ]
}
```

- `value.metadata.phone_number_id` → `app.js:53` 取出，作为后续回复时调用 Graph API 的路径参数。
- `value.messages[0]` 整个对象传给 `new Message(rawMessage)`（`services/message.js:11-22`）。因为 `type === "text"` 不等于 `"interactive"`，`message.type` 被置为 `"unknown"`，落入 `conversation.js` 的 `switch` 默认分支（`services/conversation.js:122-129`），回欢迎消息。

### 场景 B：用户点击按钮回复（路由进三个分支之一）

```json
{
  "messaging_product": "whatsapp",
  "metadata": {
    "display_phone_number": "15550001111",
    "phone_number_id": "123456789012345"
  },
  "contacts": [
    { "profile": { "name": "Jane Doe" }, "wa_id": "15551234567" }
  ],
  "messages": [
    {
      "from": "15551234567",
      "id": "wamid.HBgLMTU1NTEyMzQ1NjcVAgASGBI4...",
      "timestamp": "1750000005",
      "type": "interactive",
      "context": {
        "from": "15550001111",
        "id": "wamid.HBgLMTU1NTAwMDExMTEVAgARGBI4..."
      },
      "interactive": {
        "type": "button_reply",
        "button_reply": {
          "id": "reply-offer",
          "title": "Current promo"
        }
      }
    }
  ]
}
```

- `rawMessage.type === "interactive"` → `services/message.js:15-16` 取 `rawMessage.interactive.button_reply.id`，即 `"reply-offer"`，赋给 `message.type`。
- `conversation.js:114-121` 的 `case constants.REPLY_OFFER_ID` 命中，触发 `sendLimitedTimeOfferMessage`。
- `rawMessage.id`（这条入站消息自己的 `wamid`）会作为 `messageId` 一路传进 `GraphApi`，用于"标记已读+输入中"那次调用（`services/graph-api.js:18-34`）——**注意这不是被写入 Redis 的那个 ID**，写入 Redis 的是后面发出去的模板消息的 `wamid`。

### 场景 C：消息状态回调（delivered / read）

这是 Meta 在你发出模板消息**之后**，单独再 POST 过来的一条 `statuses` 事件，跟场景 A/B 的 `messages` 是互斥字段：

```json
{
  "messaging_product": "whatsapp",
  "metadata": {
    "display_phone_number": "15550001111",
    "phone_number_id": "123456789012345"
  },
  "statuses": [
    {
      "id": "wamid.HBgLMTU1NTEyMzQ1NjcVAgATGBI4...",
      "status": "read",
      "timestamp": "1750000012",
      "recipient_id": "15551234567",
      "conversation": {
        "id": "abc123...",
        "origin": { "type": "business_initiated" }
      },
      "pricing": {
        "billable": true,
        "pricing_model": "CBP",
        "category": "utility"
      }
    }
  ]
}
```

- `app.js:55-60`：因为这次是 `value.statuses` 而不是 `value.messages`，走 `Conversation.handleStatus`。
- `services/status.js:11-20`：取出 `id`（即上一步模板消息的 `wamid`）、`status`（`"read"`）、`recipient_id`。
- `services/conversation.js:137`：`status.status === 'read'` 通过过滤；`conversation.js:143` 用这个 `id` 去 `Cache.remove`——如果它正好是 `markMessageForFollowUp` 之前写进 Redis 的那个 key，就触发跟进消息。
- 如果 `status` 是 `"sent"` 或 `"failed"`，第 137 行的判断直接 `return`，不会做任何事——`"failed"` 状态下还会带 `errors` 数组，本 demo 目前完全没有处理失败状态的逻辑（见 PRD 第 8 节"已知限制"）。

## 6.3 Meta API 与外部服务器的集成方式（通用原理）

这个 demo 只是 Meta WhatsApp Business Platform 集成模式的一个具体实例。理解整体集成方式，有三个互相独立的接口面：

### ① Webhook（Meta → 你的服务器，推送）

在 Meta App 后台（"WhatsApp" > Configuration）注册一个 HTTPS URL，并订阅特定字段（这里是 `messages`）。之后 Meta 会主动推事件过来：

- 你的端点必须是**公网可达、带有效证书的 HTTPS**——本地开发常用 ngrok 之类工具做隧道（`CLAUDE.md` 里也提到了这点）。
- 验证握手（`GET /webhook`）只在你保存这个 URL 时发生一次。
- 之后每个真实事件都是一次 `POST /webhook`，结构就是 PRD 6.2 节展示的那种 envelope。**如果你不能很快返回 200，Meta 会带退避策略重试**——这正是 `app.js` 立刻同步回 `200 EVENT_RECEIVED`、业务逻辑异步处理（没有 `await` 阻塞响应）的原因。
- 安全性完全由你自己负责：HMAC 签名校验（用 `APP_SECRET`）是唯一挡在你的端点和"任何发现这个 URL 的人"之间的防线。

### ② Graph API（你的服务器 → Meta，同步调用）

同步的 REST 调用，目标是 `https://graph.facebook.com/{version}/{phone_number_id}/messages`，用长期有效的 `ACCESS_TOKEN`（放在 Authorization header 里的 Bearer token，这里被 `facebook-nodejs-business-sdk` 的 `FacebookAdsApi` 包装掉了）做鉴权。**所有"发送"动作——回复、模板消息、已读回执、输入中指示器——都走这条路**。每次调用同步返回结果，包含新生成的 `wamid`（出站消息 ID），这正是本 demo 用来对上后续状态回调的关键钩子（见 6.1 节第 4 点）。

### ③ WhatsApp Manager / Business Manager（管理面，不在代码里）

这不是你服务器运行时调用的 API，而是你在网页后台完成的配置动作：

- 注册并验证手机号码。
- 提交**消息模板**审核（本仓库的 `template.sh` 自动化了"提交"这一步，但审核通过与否仍由 Meta 决定，提交≠可用）。
- 管理你服务器用来取 `ACCESS_TOKEN` 的系统用户（System User）/ App 权限。

### 一句话总结

你的服务器只是一个轻量中转层——**发送/接收的可靠性、重试机制、模板审核、对话窗口与计费规则全部由 Meta 掌控**；你要做的只是：尽快验证签名并响应 webhook，然后在各种窗口规则允许的范围内（自由文本受 24 小时客服窗口限制，已审核模板任何时候都能发）通过 Graph API 回调过去。

## 7. 成功标准（演示场景下的）

- 用户从首条消息到收到欢迎按钮的延迟可感知为"即时"。
- 三个按钮分支都能正确渲染对应模板（图片可加载、优惠码可复制、轮播可左右滑动）。
- 用户点开内容后的跟进消息能在用户实际"已读"之后出现，而不是消息刚发出就追问。
- 伪造的 webhook 请求（签名不匹配）被拒绝。

## 8. 已知限制 / 技术约束（影响产品行为）

- **24 小时客服窗口**：欢迎消息和按钮回复消息只能在用户最近一次主动联系后的 24 小时内发送；超出窗口只能依赖已审核模板（产品上意味着"跟进消息"理论上也受此约束，但因为是对刚被读的消息的直接跟进，通常仍在窗口内）。
- **15 秒跟进窗口是硬编码的近似值**：`redis.js` 注释里写明"假设大多数 delivered/read 回调会在 15 秒内到达"，如果 Meta 端回调延迟超过 15 秒，用户将不会收到跟进消息。这是当前实现的已知行为，不是 bug。
- **模板需要预先人工审核**：促销/购物/菜谱三类内容如果要更换图片或文案，必须先在 WhatsApp Manager 走模板审核流程，无法即时上线。

## 9. 未来可考虑的扩展方向（仅作记录，非当前计划）

- 接入真实商品目录与购物车（脱离演示性质，成为真正的电商集成）。
- 支持自由文本理解（需要引入 NLU，目前的精确匹配按钮路由不支持）。
- 把 15 秒 TTL 改为可配置项，应对回调延迟的真实分布。
- 增加多语言模板（`locale` 当前硬编码为 `en_US`）。

## 附录 A：Meta WhatsApp Business API 入门指南（零基础）

这一节假设你完全没接触过 Meta 的开发者生态，从最基础的几个概念讲起，逐步对应到代码里实际用到的字段。

### A.1 先搞清楚"谁是谁"——账号体系

Meta 这一整套东西有好几层账号/对象，刚接触时最容易搞混。从大到小：

```
Meta 开发者账号（你的个人/公司登录账号）
   │
   └─ Meta App（在 developers.facebook.com 创建的"应用"，有自己的 App ID / App Secret）
         │
         └─ WABA = WhatsApp Business Account（你公司在 WhatsApp 商业平台上的"账户"，有 WABA ID）
               │
               └─ Phone Number（挂在这个 WABA 下的具体手机号，有 phone_number_id）
                     │
                     └─ 这个号码具体能发的"模板"（Templates），归属于 WABA，不归属于某个号码
```

类比一下：**Meta App** 像是你的"开发者工号"，**WABA** 像是"公司的官方客服账号"，**phone number** 像是"客服账号下的某一条专属热线"。一个 WABA 下可以挂多个手机号（比如不同国家用不同号码），但模板是整个 WABA 共享的。

- 本 demo 代码里只关心**最后两层**：`phone_number_id`（每次收发消息都要用）和模板名（如 `strawberries_limited_offer`，属于 WABA 层）。
- App ID / App Secret / WABA ID 主要是在**配置阶段**（Meta 后台、`template.sh` 脚本）用到，运行时代码（`app.js`/`graph-api.js`）基本不直接用 App ID 或 WABA ID。

### A.2 两种"密钥"，作用完全不同

| 名称 | 拿来做什么 | 谁验证它 | 在代码哪里 |
|---|---|---|---|
| `APP_SECRET` | 证明"这条 webhook 确实是 Meta 发给我的，不是别人伪造的" | **你的服务器**验证 Meta | `app.js:90-106`（HMAC 签名比对） |
| `ACCESS_TOKEN` | 证明"我的服务器有权限调用 Graph API 发消息" | **Meta 的服务器**验证你 | `graph-api.js:13`（传给 SDK，每次调用自动带上） |

简单记忆：`APP_SECRET` 是"收信验真"用的，`ACCESS_TOKEN` 是"发信认证"用的，两个方向相反，千万别混。

`ACCESS_TOKEN` 本身也分两种：
- **临时 Token**（在 Meta 后台"快速开始"页面直接复制的那种）：通常 24 小时就过期，只适合本地调试。
- **系统用户长期 Token**（System User Token）：在 Business Manager 里专门建一个"系统用户"身份生成，不过期（除非手动撤销），生产环境必须用这种。

### A.3 什么是 Graph API——一句话理解

Graph API 不是 WhatsApp 专属的，它是 Meta 所有产品（Facebook、Instagram、WhatsApp 广告投放等）共用的统一 HTTP 接口风格：`https://graph.facebook.com/{版本号}/{对象ID}/{动作}`。

放到 WhatsApp 场景下：
```
POST https://graph.facebook.com/v18.0/123456789012345/messages
Authorization: Bearer <ACCESS_TOKEN>
Content-Type: application/json

{ "messaging_product": "whatsapp", "to": "15551234567", "type": "text", "text": { "body": "Hi!" } }
```
- `123456789012345` 就是 `phone_number_id`——你要用"这个号码"的身份发消息，就把它放在 URL 路径里。
- 这条 HTTP 请求本身是**同步**的：你发出去，Meta 立刻返回一个 JSON，里面带新生成的消息 ID（`wamid...`），但这只代表"Meta 已经接受了这条消息去投递"，不代表用户已经收到。**用户是否收到/已读，要靠后面单独的 webhook 状态回调来告诉你**——这正是本 demo `handleStatus` 存在的原因。

代码里没有手写这条 HTTP 请求，是因为 `graph-api.js` 用官方 SDK `facebook-nodejs-business-sdk` 的 `FacebookAdsApi.call()` 把"拼 URL、加 Bearer header、序列化 JSON"这些都封装掉了——本质上等价于上面这条 curl 请求。

### A.4 什么是 Webhook——为什么不能"主动去问"

如果没有 webhook，你的服务器要知道"有没有新消息"，只能不停地去问 Meta（轮询），效率很差。Webhook 是反过来的模式：**你先告诉 Meta 一个 URL，以后有事 Meta 主动敲你的门**。

具体到配置步骤（都在 Meta 后台 App 的 "WhatsApp → Configuration" 页面完成，不是代码）：
1. 填入你的服务器 URL（必须是 HTTPS，本地开发用 ngrok 这类工具临时生成一个公网地址）。
2. 填入一个你自己随便起的 `VERIFY_TOKEN` 字符串。
3. 点击保存——Meta 此时会发一次 `GET /webhook?hub.verify_token=...&hub.challenge=...` 过来，你的服务器要把 `hub.challenge` 原样返回，证明"这个 URL 真的是我控制的"（这就是 `app.js:32-42`）。
4. 选择"订阅哪些字段"——本 demo 订阅的是 `messages`（包含用户发的消息和消息状态变化两类事件，尽管字段名只叫 messages）。

配置完之后，**任何时候**有用户发消息、或者你发出去的消息状态变化（送达/已读/失败），Meta 都会主动 `POST` 到这个 URL，内容就是 PRD 6.2 节展示的那种 JSON。

#### A.4.1 验证握手的完整细节（`GET /webhook`）

这是配置阶段唯一一次的 `GET` 请求，三个 query 参数缺一不可：

```
GET /webhook?hub.mode=subscribe&hub.verify_token=my_secret_token&hub.challenge=1158201444
```

| 参数 | 含义 |
|---|---|
| `hub.mode` | 固定值 `"subscribe"`，标识这是一次订阅校验 |
| `hub.verify_token` | 你在 Meta 后台填的那个字符串，原路返回让你比对 |
| `hub.challenge` | 一个随机数字，你必须**原样**、以**纯文本**（不是 JSON）返回，HTTP 状态码 200 |

对应代码：

```js
// app.js:32-42
app.get("/webhook", function (req, res) {
  if (
    req.query["hub.mode"] != "subscribe" ||
    req.query["hub.verify_token"] != config.verifyToken
  ) {
    res.sendStatus(403);   // token 不对，直接拒绝
    return;
  }
  res.send(req.query["hub.challenge"]);  // 校验通过，原样回显
});
```

**常见踩坑**：如果这里返回了 JSON（比如 `res.json({challenge: ...})`）而不是纯文本，Meta 后台会显示"验证失败"，因为它要的是裸字符串，不是包了一层的对象。

#### A.4.2 真实事件的请求长什么样（HTTP 层面）

不只是 body，连 headers 也要注意：

```
POST /webhook HTTP/1.1
Host: yourdomain.com
Content-Type: application/json
X-Hub-Signature-256: sha256=7f3c1e2a...（64位十六进制）
User-Agent: facebookexternalhit/1.1

{ "object": "whatsapp_business_account", "entry": [ ... ] }
```

- `X-Hub-Signature-256` 是**唯一**用来证明来源的凭证——没有 IP 白名单机制，Meta 的出口 IP 段会变化，所以**永远不要**用"判断来源 IP"代替签名校验。
- Content-Type 总是 `application/json`，所以 `app.js:29` 用 `json({ verify: ... })` 而不是 `urlencoded` 来处理这条路径上的 body（`urlencoded` 中间件第 22-26 行是给别的场景留的，webhook 走的是 json 中间件）。

#### A.4.3 签名校验到底在比对什么（逐步拆解）

```js
// app.js:90-106
function verifyRequestSignature(req, res, buf) {
  let signature = req.headers["x-hub-signature-256"];      // 1. 取出 header，形如 "sha256=7f3c1e2a..."
  let elements = signature.split("=");
  let signatureHash = elements[1];                          // 2. 拿到等号后面真正的哈希值
  let expectedHash = crypto
    .createHmac("sha256", config.appSecret)                 // 3. 用你的 APP_SECRET 做密钥
    .update(buf)                                            // 4. 对"原始请求体字节"（不是解析后的 JS 对象！）做 HMAC
    .digest("hex");
  if (signatureHash != expectedHash) {
    throw new Error("Couldn't validate the request signature.");  // 5. 不一致就拒绝整个请求
  }
}
```

**为什么必须用原始字节、不能用 `JSON.stringify(req.body)` 重新算**：JSON 序列化在不同语言/库之间可能有空格、字段顺序的细微差异，只要差一个字节，HMAC 就完全不同。这就是为什么这个 `verify` 回调要挂在 `body-parser` **解析之前**的钩子上（`app.js:29` 的 `json({ verify: verifyRequestSignature })`），第三个参数 `buf` 就是还没被解析、原封不动的 `Buffer`。

#### A.4.4 webhook 字段（field）不止 `messages` 一种

Meta 后台订阅页面会列出一串可勾选的字段，本 demo 只勾了 `messages`，但平台还提供（仅列常见的，本 demo 均未使用）：

| Field | 触发时机 |
|---|---|
| `messages` | **本 demo 使用**——用户发消息、以及你发出消息后的状态变化（delivered/read/sent/failed），都从这个字段推过来 |
| `message_template_status_update` | 你提交的模板审核结果变化（PENDING → APPROVED/REJECTED） |
| `message_template_quality_update` | 已上线模板的"质量评分"被 Meta 重新评估 |
| `phone_number_quality_update` | 你的号码被 Meta 标记了限速/质量等级变化（消息发太多被用户举报会触发） |
| `account_update` | WABA 账号本身的状态变化（如被封禁） |

**容易误解的点**：字段名是 `messages`，但它的 payload 里其实同时可能出现 `value.messages`（用户发的消息）**或** `value.statuses`（状态回调）——这两者是同一个订阅字段下的两种不同事件形状，`app.js:55` 和 `app.js:62` 分别用 `if (value.statuses)` / `if (value.messages)` 来区分，而不是看 `field` 的值。

#### A.4.5 Meta 的重试与超时行为

- 你的服务器必须在**几秒内**返回 HTTP 200，否则 Meta 视为投递失败。
- 失败后 Meta 会**按退避策略重试**（间隔逐渐变长），重试一段时间后放弃（具体窗口由 Meta 控制，不是你能配置的）。
- 这意味着**同一个事件可能被投递多次**（网络抖动、你这边短暂超时都可能导致重复）——本 demo 目前**没有做任何去重**：`handleMessage`/`handleStatus` 如果被同一个事件触发两次，会真的发送两次回复。这是当前实现的一个已知缺口，可以补在 PRD 第 8 节"已知限制"里。
- 正因为重试机制存在，`app.js:73` 选择"先回 200，业务逻辑异步处理"而不是"处理完再回 200"——避免业务逻辑（调用 Graph API）的延迟拖慢到让 Meta 误判超时、触发不必要的重试。

#### A.4.6 不保证顺序

如果用户连续快速发了三条消息，理论上你的 `/webhook` 可能不是严格按发送顺序收到这三个 `POST`（网络路径、Meta 内部队列都可能导致乱序）。本 demo 因为每次只依据**单条消息自身的按钮 ID**路由、不依赖"上一条消息是什么"的上下文，天然不受这个问题影响——这也是为什么 CLAUDE.md 里强调"每条消息无状态路由"是有意为之的设计，不只是偷懒。

### A.5 消息到底分几类，怎么决定能不能发

这是最容易踩坑的一点。WhatsApp 不允许商家随意给用户发广告骚扰消息，所以有一套"窗口"规则：

- **用户主动找你说话**，会打开一个 **24 小时客服窗口**：在这 24 小时内，你可以自由发任何格式的消息（文本、按钮、图片……），不需要审核，这就是本 demo 里"欢迎消息"和"跟进消息"能直接发的原因（`messageWithInteractiveReply`，见 `graph-api.js:50-73`）。
- **超过 24 小时**，或者你想**主动**找用户说话（用户根本没先联系你），就只能发**模板消息（Template Message）**——内容结构必须提前在 WhatsApp Manager 里提交、经过 Meta 审核通过，运行时只能填充里面预留的"变量"（图片链接、优惠码等），不能自由编写文案。这就是 `messageWithUtilityTemplate` / `messageWithLimitedTimeOfferTemplate` / `messageWithMediaCardCarousel` 存在的原因。

模板还分三个**类别（Category）**，审核标准不同：
| 类别 | 用途举例 | 本 demo 用到的 |
|---|---|---|
| `UTILITY`（事务性） | 订单确认、配送通知、账单提醒 | `grocery_delivery_utility` |
| `MARKETING`（营销） | 促销、新品推广 | `strawberries_limited_offer`、`recipe_media_carousel` 实际上更偏营销性质，但 demo 里用的是较宽松的审核类别做演示 |
| `AUTHENTICATION`（身份验证） | 一次性验证码（OTP） | 本 demo 未使用 |

模板提交后会有三种审核状态：`PENDING`（审核中）→ `APPROVED`（通过，可以用）或 `REJECTED`（被拒，需要改了重新提交）。**审核可能需要几分钟到几十分钟，不是即时的**——这也是为什么"改促销图片"在 PRD 第 8 节被列为"已知限制"。

### A.6 计费模型简单提一下（虽然本 demo 代码不涉及计费逻辑）

Meta 对 WhatsApp 消息按"对话（Conversation）"收费，不是按条数：一个 24 小时窗口内，无论你来回发多少条消息，都算**一个对话**，按对话的发起方（用户发起 user-initiated，还是商家发起 business-initiated）和类别计费一次。这解释了 PRD 6.2 节场景 C 的状态回调 JSON 里为什么会带一个 `pricing` 字段（`category: "utility"`、`billable: true`）——那就是 Meta 在告诉你"这次对话按什么类别计费了"。

### A.7 用一张总览图串起来

```
                 【一次性后台配置，不是代码】
Meta 开发者账号 → 创建 App（App ID/Secret）→ 关联 WABA（WABA ID）→ 注册手机号（phone_number_id）
                                                      │
                                            提交模板 → 等审核通过
                                                      │
              ┌───────────────────────────────────────┘
              │
   【运行时，两条独立通道，对应本 demo 的 app.js / graph-api.js】
              │
   用户发消息 ──► Webhook POST /webhook（你被动接收，HMAC 验证来源）
                      │
                      ▼
              你的业务逻辑决定要回什么
                      │
                      ▼
   你发消息 ──► Graph API POST /{phone_number_id}/messages（你主动调用，Bearer Token 认证）
                      │
                      ▼
            （24h 窗口内自由发 or 窗口外只能发已审核模板）
                      │
                      ▼
   送达/已读/失败 ──► Webhook POST /webhook（再一次被动接收，告诉你结果）
```

把这张图和 PRD 6.1 节的具体调用流程图对照看，就能把"通用平台机制"和"这份代码具体怎么实现"对上号了。
