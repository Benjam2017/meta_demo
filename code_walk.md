# Jasper's Market Demo — 代码走读

Meta 官方的 "Jasper's Market" 示例应用（`fbsamples/whatsapp-business-jaspers-market`）。单一 Express 服务器，演示 WhatsApp Business Platform 的功能：交互式按钮回复、Utility 模板、限时优惠模板、媒体卡片轮播模板。除了一个 15 秒 TTL 的 Redis 标记外，整体无状态。

## 整体架构

```
WhatsApp 用户 ──► Meta Cloud API ──► POST /webhook (app.js)
                                         │  HMAC 校验 (x-hub-signature-256)
                                         ▼
                              Conversation.handleMessage / handleStatus
                                         │
                          ┌──────────────┴──────────────┐
                          ▼                              ▼
                    GraphApi（发送消息）           Cache/Redis（15秒 TTL 标记）
```

## 入口：`app.js`

- `app.use(json({ verify: verifyRequestSignature }))` — 在 body-parser 解析 JSON 之前，先用原始 buffer 计算 HMAC-SHA256，跟请求头 `x-hub-signature-256` 比对。签名不匹配会在 verify 回调里 `throw`，由 Express 默认错误处理转成 500（没有本地 catch——失败是故意的）。
- `GET /webhook` — Meta 的一次性订阅校验握手：核对 `hub.verify_token`，原样回显 `hub.challenge`。
- `POST /webhook` — 遍历 `entry[].changes[].value`；把 `value.statuses` 分发给 `Conversation.handleStatus`，`value.messages` 分发给 `Conversation.handleMessage`。无论内部处理成功与否，**始终**回 `200 EVENT_RECEIVED`——这是 Meta 的要求，避免业务异常触发 webhook 重试风暴。
- 启动时调用 `config.checkEnvVariables()`；缺失的环境变量只是 `console.warn`，不会导致启动失败（demo 级别的宽松处理）。

## 消息建模：`services/message.js` / `services/status.js`

两个极薄的包装类，把原始 webhook payload 整形成内部好用的对象：

- `Message` — 只认 `type === 'interactive'`，取出 `interactive.button_reply.id` 作为 `this.type`（这个值就是后续的路由键）；其它类型一律标记为 `'unknown'`。这个 demo 只响应按钮点击，不解析自由文本。
- `Status` — 提取 `id`（消息 ID）、`status`（sent/delivered/read/failed）、`recipient_id`。

## 核心逻辑：`services/conversation.js`

`Conversation` 是一组静态方法构成的路由器，以 `message.type`（按钮回复 ID）为键：

| 按钮 ID | 行为 |
|---|---|
| `reply-interactive-with-media` | 发送 `grocery_delivery_utility` Utility 模板 |
| `reply-media-card-carousel` | 发送 `recipe_media_carousel` 轮播模板 |
| `reply-offer` | 发送 `strawberries_limited_offer` 模板（优惠码 `BERRIES20`） |
| 其它/首次消息 | 发送三按钮的 "Try out the demo" 交互消息（`APP_DEFAULT_MESSAGE`） |

发送上述三个模板之后，都会调用 `markMessageForFollowUp(response.messages[0].id)`，把**刚发出的那条消息**的 ID 写入 Redis，TTL 15 秒。

`handleStatus` 的逻辑：
1. 只关心 `delivered` 或 `read` 状态，其它直接忽略。
2. 调用 `Cache.remove(status.messageId)`——如果这个 ID 之前被标记过（说明这是刚发的模板消息，现在用户确实看到了），就发一条跟进消息 `APP_TRY_ANOTHER_MESSAGE`（"还有什么可以帮你？"）。

这样实现了"只有用户真正看到模板内容后才追问"，且不需要持久化任何真正的会话状态——Redis 只是被当作一个短期的"等待确认"标志位用，而不是常规的 session store。

## 发送层：`services/graph-api.js`

封装 `facebook-nodejs-business-sdk` 的 `FacebookAdsApi`。所有发送都走私有方法 `#makeApiCall`：
- 如果带有 `messageId`（即这是对一条收到消息的回复），先调一次 API 把消息标记为已读并显示"正在输入"指示器，再发真正的内容。
- 四个公开方法对应四种消息形态：
  - `messageWithInteractiveReply` — 正文文本 + 回复按钮。
  - `messageWithUtilityTemplate` — 带图片 header 的模板。
  - `messageWithLimitedTimeOfferTemplate` — 在上面基础上加一个 `limited_time_offer` 组件（过期时间 = 当前时间 + 48 小时，硬编码 `48*60*60*1000`）和一个携带优惠码的 `copy_code` 按钮组件。
  - `messageWithMediaCardCarousel` — `carousel` 组件；`imageLinks` 数组每个元素对应一张卡片（`card_index` = 下标）。
- 错误只记录日志并重新抛出，从不静默吞掉。

## 配置与缓存

- `services/config.js` — `dotenv` 加载 + `Object.freeze` 导出只读配置对象。`checkEnvVariables()` 对缺失的 `ACCESS_TOKEN`、`APP_SECRET`、`VERIFY_TOKEN`、`REDIS_HOST`、`REDIS_PORT` 只警告，不抛错。
- `services/redis.js` — 极简的 `Cache.insert`/`Cache.remove`。注释解释了原因：写这段代码时的 redis 客户端不支持给 Set 的单个成员设置 TTL，所以用一个普通 key（空字符串值）+ `EXPIRE 15` 来模拟。`remove` 返回这个 key 在被删除前是否还存在（即还没过期），`handleStatus` 据此判断是否需要发跟进消息。

## 代码背后的 Meta WhatsApp Business API 概念

平台分两条独立通道：

1. **出站 —— Graph API**：你的服务器调用 `POST https://graph.facebook.com/v18.0/{phone_number_id}/messages` 来发送消息。
2. **入站 —— Webhook**：每当用户给你发消息，或某条已发消息的状态发生变化时，Meta 会调用你服务器的 `/webhook`。

### 凭证（`.env`）

| 变量 | 作用 | 用在哪里 |
|---|---|---|
| `ACCESS_TOKEN` | 授权你的应用调用 Graph API、代表你的 WhatsApp Business Account 发消息 | `graph-api.js` |
| `APP_SECRET` | 用来验证入站 webhook 确实来自 Meta（HMAC 签名） | `app.js` → `verifyRequestSignature` |
| `VERIFY_TOKEN` | 你自己设定的字符串，仅在配置 webhook 时用一次，证明你掌控接收端服务器 | `app.js` → `GET /webhook` 握手 |
| `phone_number_id` | 不在 `.env` 里——它出现在每条 webhook payload 内部（`value.metadata.phone_number_id`），同时也用作回复时调用接口的 URL 路径段 | `app.js` 第 53 行附近，向下传给 `Conversation`/`GraphApi` |

### Webhook 握手（一次性）

注册 webhook URL 时，Meta 会发 `GET /webhook?hub.mode=subscribe&hub.verify_token=...&hub.challenge=...`。服务器需要在 `hub.verify_token` 匹配时原样回显 `hub.challenge`。这只发生在配置/重新配置时，不是每条消息都要走这个流程。

### 消息往返流程

1. 用户点击按钮 → Meta POST 过来 `entry[].changes[].value = { metadata: { phone_number_id }, messages: [{ id, from, type: "interactive", interactive: { button_reply: { id, title } } }] }`。`Message` 类把 `button_reply.id` 取出作为路由键。
2. 服务器通过 `POST /{phone_number_id}/messages` 回复，请求体结构取决于 `type`（`text`/`interactive`/`template`/...）——对应 `GraphApi.messageWith*` 四个构造方法。
3. 之后 Meta 会再次 POST，这次带的是 `statuses` 数组（不是 `messages`），汇报之前发出的某条消息的 `sent`/`delivered`/`read`/`failed` 状态——由 `Status` + `Conversation.handleStatus` 处理。

### 为什么"模板"是个独立概念

WhatsApp 的反垃圾消息规则：只有在用户最近一次给你发消息后的 24 小时内（"客服窗口期"），才能发自由文本消息。超出这个窗口，只能发**预先审核通过的模板消息**。

- `grocery_delivery_utility`、`strawberries_limited_offer`、`recipe_media_carousel` 是模板名称，必须事先在 WhatsApp Manager 里审核通过——`template.sh` 就是负责注册它们的脚本（通过 Meta 的 Asset Manager 上传图片，打印出生成的模板定义）。
- 模板有固定的 `components` 结构（header/body/button），发送时只填 `parameters`——你不是在写文案，而是在填 Meta 已经批准的内容里的空位。
- `messageWithInteractiveReply` **不是**模板——它是常规交互消息，只能在 24 小时窗口期内发送，这里之所以没问题，是因为它总是作为对用户刚发消息的直接回复来发的。

### HMAC 签名校验

`x-hub-signature-256` 是 Meta 用你的 `APP_SECRET` 对原始 POST body 做 HMAC-SHA256 签名后的值。服务器重新计算一遍并比对，借此确认这条 webhook 确实来自 Meta，而不是第三方伪造打到你公开端点的请求。这也是为什么这个校验回调必须跑在原始 buffer 上，发生在任何 JSON 解析/修改之前。

## 一处小备注

`app.js` 引入了 `Message`（`require('./services/message')`），但自己并没有直接使用——真正的 `new Message(rawMessage)` 是在 `conversation.js` 内部完成的。这是 Meta 官方示例自带的一个无害的多余 import，不是本地引入的问题，无需处理。
