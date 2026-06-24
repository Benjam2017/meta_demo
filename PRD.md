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
