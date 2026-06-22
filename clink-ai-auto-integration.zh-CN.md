# ClinkBill AI 自动接入提示词 v2

帮我把 ClinkBill 支付接入到当前项目。

目标：尽量全自动完成 UAT 支付接入。唯一允许人工介入的是：你运行 `clink login` 后，我在打开的 Dashboard 登录页里手动完成登录。除此之外，不要让我手动复制 Secret Key、productId、priceId、webhook signing key，或手动配置 Dashboard webhook。

请保持真实、可验证：如果没有人打开 `checkoutUrl` 并完成 UAT 测试支付，不要把“真实 checkout session 创建成功 + 签名模拟 webhook 通过”说成“真实付款全链路完成”。

## 资料

请先读取：

- https://github.com/clinkbillcom/clink-integ-skills
- https://docs.clinkbill.com/api-reference/introduction
- https://github.com/5048429/clink-dev-cli

如果资料中字段与本提示词不一致，以官方文档和当前 CLI 输出为准，并在最终回复中说明差异。

## 安装 CLI

不要使用 `node dist/index.js`。先安装并使用 `clink`：

```bash
npm install -g github:5048429/clink-dev-cli
clink --version
```

如果全局安装失败：

```bash
npm install --prefix ./.clink-tools github:5048429/clink-dev-cli
```

本地安装后：

- Linux/macOS: `./.clink-tools/node_modules/.bin/clink`
- Windows PowerShell: `.\.clink-tools\node_modules\.bin\clink.cmd`

下文统一写 `clink`，如果使用本地安装，请自动替换成对应路径。

## 登录与密钥

运行：

```bash
clink login
```

如果 CLI 没有自动完成登录，请暂停并提示我：

```text
请在打开的浏览器里完成 ClinkBill UAT Dashboard 登录。登录完成后告诉我继续。
```

我确认后继续：

```bash
clink dashboard whoami --json
clink dashboard apikey ensure-secret --save --json
clink auth status --json
```

如果项目需要 `.env`，可以用 CLI 获取真实 key 并写入 `.env`，但必须：

- 不要把真实 key 写进源码、README、测试 fixture 或最终回复
- 不要在最终回复中展示真实 key
- 确保 `.env` 被 git 忽略
- `.env.example` 只写占位符
- 每次运行 `clink dashboard webhook ensure --save-secret` 后，必须把 CLI profile 里最新的 webhook signing key 同步到 `.env`，然后重启本地服务，否则 webhook 验签会失败

推荐 `.env.example`：

```env
CLINK_ENV=sandbox
CLINK_BASE_URL=https://uat-api.clinkbill.com/api/
CLINK_SECRET_KEY=sk_uat_xxx
CLINK_WEBHOOK_SIGNING_KEY=whsec_xxx
CLINK_SUBSCRIPTION_PRODUCT_ID=prod_xxx
CLINK_SUBSCRIPTION_PRICE_ID=price_xxx
CLINK_PAYMENT_INSTRUMENT_ID=pi_xxx
```

## 架构侦察（强制前置）

不要预设当前项目使用哪种后端架构，也不要根据文件名先入为主。接入前必须先自行收集证据，识别项目的真实运行方式。

在修改代码前，先完成以下侦察：

1. 列出项目根目录关键文件和目录，例如 package/lock 文件、依赖清单、部署配置、server 入口、api/routes 目录、环境变量示例、构建脚本。
2. 读取启动脚本、依赖、部署配置和 README，判断：
   - 前端入口在哪里
   - 是否存在服务端运行时
   - HTTP 路由在哪里定义
   - 环境变量如何注入
   - 订单/购买按钮/发货逻辑在哪里
   - 当前框架是否会默认解析 body，webhook 是否能拿到 raw body
3. 用搜索定位已有支付、订单、checkout、webhook、subscription、email、download、fulfillment 等相关代码。
4. 输出一小段“架构侦察结果”，说明你准备把 Clink 接入到哪些文件/模块，依据是什么。

如果没有发现可信的服务端运行时：

- 不要把 `CLINK_SECRET_KEY` 或 webhook signing key 放进前端代码
- 不要从浏览器直接请求 Clink UAT API
- 不要声称已经完成支付接入
- 应说明当前项目缺少安全保存密钥和接收 webhook 的后端能力
- 如果项目已有可部署的 serverless/functions/edge 目录或部署平台配置，可以在该后端能力中接入
- 如果完全是静态站点，应给出需要新增的最小后端/API 服务方案，并只在当前项目结构允许安全落地时实现

## 参考 starter

可以运行官方 starter 到临时目录作为字段和代码参考，但不要直接覆盖当前项目：

```bash
clink init --framework generic --out <temp-dir>
```

starter 只能作为 Clink 字段、签名、curl 和示例流程参考，不能替代架构侦察结果。只有在已经确认当前项目框架后，才可以生成对应 starter 到临时目录辅助参考。

## 代码接入

基于“架构侦察结果”接入 Clink。使用项目现有 HTTP/server 能力，不要在已有后端旁边平行引入一个无关的新后端框架。若必须新增最小服务端能力，需要说明原因，并保持与项目现有部署方式兼容。

必须实现：

1. 创建 checkout session 的服务端接口
2. 创建 subscription 的服务端接口
3. webhook 接收接口，并校验签名
4. 一个可一键启动的最小 demo
5. curl 示例
6. 本地自动测试或 smoke test

Clink UAT API：

```text
https://uat-api.clinkbill.com/api/
```

服务端请求必须带：

```text
X-API-Key: <CLINK_SECRET_KEY>
X-Timestamp: <milliseconds timestamp>
Content-Type: application/json
```

不要把 `CLINK_SECRET_KEY` 暴露到浏览器。前端只能调用自己后端的 API。

## Checkout

必须支持：

- 使用已有 Clink `productId` / `priceId`
- 使用临时 `priceDataList`

推荐服务端 route：

```text
POST /api/clink/checkout
```

推荐上游 API path：

```text
POST /checkout/session
```

请求 payload 至少支持：

- `customerEmail`
- `merchantReferenceId`
- `successUrl`
- `cancelUrl`
- `uiMode`
- `paymentMethodType`
- `productId` / `priceId`
- `priceDataList`

字段映射注意：

- CLI 的 `--amount 10 --currency USD` 表示 10 USD；服务端传给 checkout 的 `originalAmount` / `unitAmount` 应按用户可见金额传递，除非官方文档明确要求 minor unit。
- webhook fixture 或某些事件里的 `amount` 可能看起来像 minor unit。不要用 webhook 的金额重新计算本地订单金额；保留为 provider 原始字段即可。
- 创建本地订单时，用本地订单号作为 `merchantReferenceId`，后续通过 webhook 的 `merchantReferenceId` 回填本地订单。
- Clink checkout 返回字段可能在不同层级，提取 checkout URL 时请兼容：
  - `checkoutUrl`
  - `url`
  - `sessionUrl`
  - `data.checkoutUrl`
  - `data.url`
  - `data.sessionUrl`
- 提取 session id 时请兼容：
  - `sessionId`
  - `id`
  - `data.sessionId`
  - `data.id`

用 CLI 验证：

```bash
clink checkout create \
  --customer-email buyer@example.com \
  --amount 10 \
  --currency USD \
  --name "AI Integration Test" \
  --quantity 1 \
  --success-url https://example.com/success \
  --cancel-url https://example.com/cancel \
  --json
```

## Subscription

推荐服务端 route：

```text
POST /api/clink/subscription
```

推荐上游 API path：

```text
POST /subscription
```

如果需要 product/price，请用 CLI 自动创建，不要让我手动找 ID：

```bash
clink product create \
  --name "AI Subscription Test" \
  --image-file <local-image-path> \
  --tax-category software_service \
  --amount 10 \
  --currency USD \
  --type recurring \
  --interval month \
  --default \
  --json
```

注意：

- 当前 CLI 创建 product 可能要求 `--image-id` 或 `--image-file`。如果项目没有现成图片，可以自动生成临时 PNG 作为 product image，不要让我手动准备。
- 从返回值自动读取 `productId` 和 `defaultPrice` / `initialPriceId`，写入 `.env` 或项目配置。
- 不要把真实 ID 写进源码；可以写入本地 `.env`。

创建订阅：

```bash
clink subscription create \
  --customer-email buyer@example.com \
  --product-id <productId> \
  --price-id <priceId> \
  --payment-instrument-id <paymentInstrumentId> \
  --payment-method-type CARD \
  --payment-currency USD \
  --return-url https://example.com/subscription/return \
  --json
```

如果没有 `paymentInstrumentId`：

- subscription route 仍然要实现
- 返回清晰错误，说明需要先完成一笔 UAT checkout，并从真实 `order.succeeded` webhook 中读取
- 不要假造 `paymentInstrumentId`
- 不要为了让测试通过而把 subscription 伪装成真实成功

## Webhook

推荐 route：

```text
POST /api/clink/webhook
```

必须读取 raw body 后验签，不要先调用任何会消耗请求体的 JSON parser 或框架自动 body parser。验签通过后再解析 JSON。

签名：

```text
HMAC_SHA256(CLINK_WEBHOOK_SIGNING_KEY, X-Clink-Timestamp + "." + rawBody)
```

读取 headers：

```text
X-Clink-Timestamp
X-Clink-Signature
```

至少处理：

| 事件 | 行为 |
| --- | --- |
| `session.complete` | 记录 checkout session 完成，可更新订单为等待最终支付结果，但不要仅凭这个事件发货 |
| `order.succeeded` | 标记本地订单已支付/已完成，授予权益，触发发货 |
| `order.failed` | 标记支付失败，不发货 |
| `refund.succeeded` | 标记退款成功 |
| `subscription.created` | 记录订阅创建状态 |
| `invoice.paid` | 记录发票/续费付款 |

必须记录收到的 webhook event，便于验证：

- event id
- event type
- receivedAt
- `merchantReferenceId`
- provider order/session/subscription id
- 验签是否通过
- 处理动作

## 自动配置 Dashboard webhook

本地测试必须使用 HTTPS tunnel，优先 cloudflared，不要用 localtunnel。

Windows 如果没有 cloudflared，可以尝试：

```powershell
winget install --id Cloudflare.cloudflared -e --accept-package-agreements --accept-source-agreements
```

启动 tunnel：

```bash
cloudflared tunnel --url http://127.0.0.1:<PORT> --no-autoupdate
```

如果日志显示 QUIC / UDP 超时，例如 `QUIC connection failed` 或公网 URL 返回 Cloudflare 1033，请切到 HTTP/2：

```bash
cloudflared tunnel --url http://127.0.0.1:<PORT> --no-autoupdate --protocol http2
```

得到公网 URL 后运行：

```bash
clink dashboard webhook ensure \
  --url <public-webhook-url>/api/clink/webhook \
  --events core \
  --save-secret \
  --json
```

`--events core` 会提交 `22,2,3,5,7,14`，对应：

- `session.complete`
- `order.succeeded`
- `order.failed`
- `refund.succeeded`
- `subscription.created`
- `invoice.paid`

重要：每次 quick tunnel URL 变化后，都要重新运行 `clink dashboard webhook ensure --save-secret --json`，同步新的 webhook signing key 到 `.env`，并重启本地服务。

## 验证

先运行本地自动测试。如果真实 UAT API 不适合自动测试，可以加入明确的本地 mock 开关，例如 `CLINK_MOCK=true`，但最终要清楚区分 mock 与真实 UAT。

推荐验证顺序：

1. 本地测试：

```bash
npm test
```

2. Clink CLI 健康检查：

```bash
clink doctor --json
```

3. 签名模拟 webhook：

```bash
clink webhook simulate order.succeeded \
  --secret env:CLINK_WEBHOOK_SIGNING_KEY \
  --forward-to <local-or-public-webhook-url>/api/clink/webhook \
  --json
```

4. CLI smoke test：

```bash
clink smoke-test \
  --webhook-url <public-webhook-url>/api/clink/webhook \
  --json
```

5. 创建真实 UAT checkout session：

```bash
clink checkout create \
  --customer-email buyer@example.com \
  --amount 10 \
  --currency USD \
  --name "Webhook Real Test" \
  --quantity 1 \
  --payment-method-type CARD \
  --merchant-reference-id "ai-webhook-test-<timestamp>" \
  --success-url https://example.com/success \
  --cancel-url https://example.com/cancel \
  --json
```

如果需要确认真实付款 webhook：

- 打开 `checkoutUrl`
- 完成 UAT 测试支付
- 等待 Dashboard webhook 调用本地公网 URL
- 确认真实收到 `session.complete` 和 `order.succeeded`

如果没有完成这一步，只能报告：

- 真实 UAT checkout session 创建成功
- 签名模拟 webhook 通过公网 tunnel 返回 200
- 本地订单处理逻辑通过模拟事件验证

不能报告“真实付款 webhook 已完成”。

## curl 示例

最终 README 或交付说明里至少提供：

创建 checkout：

```bash
curl -X POST http://localhost:3000/api/clink/checkout \
  -H "Content-Type: application/json" \
  -d '{"productId":"beijing-72h","customerEmail":"buyer@example.com"}'
```

使用已有 Clink product/price：

```bash
curl -X POST http://localhost:3000/api/clink/checkout \
  -H "Content-Type: application/json" \
  -d '{"productId":"prod_xxx","priceId":"price_xxx","amount":10,"currency":"USD","customerEmail":"buyer@example.com","name":"AI Integration Test"}'
```

创建 subscription：

```bash
curl -X POST http://localhost:3000/api/clink/subscription \
  -H "Content-Type: application/json" \
  -d '{"customerEmail":"buyer@example.com","productId":"prod_xxx","priceId":"price_xxx","paymentInstrumentId":"pi_xxx","paymentMethodType":"CARD","paymentCurrency":"USD","returnUrl":"http://localhost:3000/subscription/return"}'
```

## 最终交付

请给我：

1. 架构侦察结果：项目运行方式、服务端入口、路由位置、订单入口、raw body 处理方式
2. 修改文件列表
3. 新增 API route / service 说明
4. `.env.example`
5. 一键启动命令
6. curl 示例
7. CLI 验证结果摘要
8. Dashboard webhook endpoint
9. tunnel URL 和本地 URL（如果仍在运行）
10. 测试结果，明确区分：
   - 本地 mock
   - 签名模拟 webhook
   - 真实 UAT checkout session
   - 真实 UAT 付款 webhook
11. 剩余人工步骤

预期剩余人工步骤只能是：

- 我在 `clink login` 打开的 Dashboard 页面里手动登录
- 如需验证真实付款 webhook，我手动打开 `checkoutUrl` 完成 UAT 测试支付

如果启动了临时服务或 tunnel：

- 如果是为了交付给我试用，请保留运行并给出 URL / PID
- 如果只是自动验证，请在最终回复前停止临时进程
