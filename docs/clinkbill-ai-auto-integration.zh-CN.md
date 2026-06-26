# ClinkBill AI 自动接入介绍

这份文档面向希望用 AI agent 接入 ClinkBill 支付的商户、开发者和低代码平台用户。它说明 ClinkBill 的自动接入方案如何工作、哪些步骤可以自动完成、哪些步骤仍需要人工参与，以及 agent 应如何使用 `clink-integ-skills` 和 `clink-integ-cli` 完成可靠的 sandbox 集成。

## 适用场景

AI 自动接入适用于：

- 已有网站、SaaS、工具站、内容站或应用，需要接入 ClinkBill 收款。
- 已有付费产品、订阅套餐、价格页、CMS 商品或数据库商品，需要批量创建到 Clink。
- 使用低代码平台、云 IDE、托管 sandbox 或本地开发环境，希望减少 Dashboard 手动配置。
- 希望由 agent 自动完成 checkout、subscription、webhook、catalog import 和验证流程。

这套流程默认面向 sandbox。生产上线需要单独做生产准备和上线验证。

## 组成部分

ClinkBill AI 自动接入由两部分组成：

- `clink-integ-skills`：给 coding agent 使用的接入知识包。它约束 agent 如何识别项目架构、发现商品、设计 checkout/subscription/webhook 流程，并避免把密钥放到前端或公开文件里。
- `clink-integ-cli`：给 agent 和开发者执行的命令行工具。它用 Secret Key 调用 Clink API，完成产品和价格导入、checkout 创建、webhook endpoint 管理、签名模拟、本地验证和 smoke test。

目标是：除 Secret Key 获取或本地 Dashboard 登录外，尽量自动完成 sandbox 支付接入。用户不应该被要求手动创建 product、price、webhook endpoint，也不应该在初始接入时手动提供 `CLINK_WEBHOOK_SIGNING_KEY`。

## AI Agent 会自动完成什么

Agent 通常可以自动完成：

- 安装或读取 `clink-integ-skills`。
- 安装最新 `clink-integ-cli` 并检查命令能力。
- 识别项目后端、路由、环境变量和购买入口。
- 扫描已有付费产品、订阅套餐、价格页、配置或 CMS 数据。
- 生成 `clink-catalog.json`，并用 CLI 校验、计划、导入产品和价格。
- 实现服务端 checkout、subscription 和 webhook handler。
- 通过 Secret Key API 创建或更新 webhook endpoint。
- 获取、保存并同步 webhook signing key 到运行时环境。
- 运行本地测试、签名模拟 webhook、CLI smoke test 和真实 sandbox checkout session 创建。

## 用户通常只需要做什么

仍需要人工参与的步骤通常只有：

- 本地桌面环境：如果没有现成 Secret Key，需要用户在 `clink login` 打开的浏览器里完成 Dashboard 登录。
- 云端、低代码、sandbox 或无浏览器环境：用户需要到 ClinkBill Dashboard 获取一次 `CLINK_SECRET_KEY`，并把它提供给 agent 所在平台的安全 Secret 或环境变量。
- 真实付款验收：用户需要打开 `checkoutUrl` 并完成 sandbox 测试支付。

生产环境上线、生产 Secret Key、商户审批或财务流程不属于 sandbox 自动接入的默认范围。

## Secret Key 认证

Clink API 使用 Secret Key 认证。服务端请求需要发送：

```text
X-API-Key: <CLINK_SECRET_KEY>
X-Timestamp: <milliseconds timestamp>
Content-Type: application/json
```

Secret Key 来自 `Merchant Dashboard > Developers > API Keys`，点击 `Initialize Key` 后复制并安全保存。Secret Key 只显示一次。

### 本地桌面环境

如果 agent 运行在本地桌面环境，且没有现成 Secret Key，可以使用：

```bash
clink login
clink dashboard whoami --json
clink dashboard apikey ensure-secret --save --json
clink auth status --json
```

`clink login` 只用于让用户手动完成 Dashboard 登录。登录完成后，CLI 会读取或初始化 sandbox Secret Key，并保存到本地 CLI profile。后续产品、价格、checkout、subscription、webhook endpoint、doctor 和 smoke-test 都应走 Secret Key API，不再依赖 Dashboard Console token。

### 云环境、低代码或无浏览器环境

无浏览器环境不要卡在 `clink login`。用户只需要提供 `CLINK_SECRET_KEY`，agent 应把它写入安全的服务端环境变量、平台 Secret 或被 git 忽略的 `.env`，然后运行：

```bash
clink auth secret set --api-key env:CLINK_SECRET_KEY --env sandbox
clink auth status --json
clink doctor --json
```

不要在初始阶段向用户索要 `CLINK_WEBHOOK_SIGNING_KEY`。Webhook signing key 应由 `clink webhook endpoint ensure --save-secret` 创建、返回、保存或轮换后再同步到运行时。

## 商品目录与价格导入

如果网站、CMS、数据库、源码或价格页已经存在付费产品、一次性购买项、订阅套餐、价格、币种或计费周期，agent 应先自动发现这些信息，再生成确定性的 `clink-catalog.json`。

推荐发现顺序：

1. 运行中的 API、价格页 DOM、hydrated JSON。
2. 源码、配置、seed 数据、public/static 资源。
3. 信息仍不足时再询问用户。

CLI 不负责爬取网站。Agent 负责发现商品并写入 catalog；CLI 负责校验、计划、导入和维护 sourceId 映射。

每个 product 和 price 都需要稳定的 `sourceId`。每个 product 必须包含且只包含一个图片来源：

- `imageId`：已有 Clink OSS 图片 ID。
- `imageUrl`：公开 HTTP(S) 图片 URL，由 CLI 下载、校验、上传并缓存。
- `imageFile`：本地图片路径，相对 `clink-catalog.json` 解析，也可配合 `--project-root` 和 `--public-dir` 解析项目 public/static 资源。

导入流程：

```bash
clink catalog validate --file ./clink-catalog.json --project-root . --public-dir public --json
clink catalog plan --file ./clink-catalog.json --mapping-file ./.clink/catalog-map.json --project-root . --public-dir public --json
clink catalog import --file ./clink-catalog.json --mapping-file ./.clink/catalog-map.json --project-root . --public-dir public --json
```

`catalog validate` 会检查图片存在性、MIME 类型、大小和字段使用方式。URL 不应写入 `imageId`。`catalog import` 会把 `imageUrl` 或 `imageFile` 上传到 `/product/image/upload`，拿到 `ossId` 后创建产品，并用 sha256 缓存避免重复上传。

## Checkout 与 Subscription 接入

Agent 应在服务端实现 checkout session 创建接口。浏览器前端只能调用自己的后端，不应直接调用 Clink Secret Key API，也不应持有 `CLINK_SECRET_KEY`。

服务端接入需要明确产品模式：

- 注册产品模式：使用 `productId` 和 `priceId`。如果站点已有商品或套餐，优先通过 `clink-catalog.json` 和 `clink catalog import` 创建并维护映射，不要让用户手动复制 ID。
- 非注册产品模式：用服务端生成的 `priceDataList` 表达一次性购买项。商户系统仍需要本地订单模型来保存商品含义、金额、履约目标和对账信息。

`merchantReferenceId` 只用于对账和关联本地订单，不是 Clink 侧幂等键。

## Webhook 自动配置与验签

Webhook endpoint 管理应使用 Secret Key API：

```bash
clink webhook endpoint ensure \
  --url <public-webhook-url>/api/clink/webhook \
  --events core \
  --save-secret \
  --json
```

默认建议使用 `--events core`，它覆盖常见支付接入需要的事件：

- `session.complete`
- `order.succeeded`
- `order.failed`
- `refund.succeeded`
- `subscription.created`
- `invoice.paid`

如确实需要完整事件目录，可使用 `--events all`。它会展开为 Secret Key API 当前支持的 38 个事件名。公开 API 请求体使用事件名，不使用 Dashboard 数字 event code。

本地 `.env` 项目优先使用：

```bash
clink webhook endpoint ensure \
  --url <public-webhook-url>/api/clink/webhook \
  --events core \
  --save-secret \
  --sync-env-file .env.local \
  --json
```

如果平台 Secret API 需要在同一步拿到明文 signing key，可在受控写入步骤中使用 `--show-secret`，由 agent 直接写入平台 Secret。不要把真实 signing key 写入源码、README、测试 fixture、公开日志或最终回复。

每次 webhook URL、预览域名或 path 变化，都需要重新运行 `ensure --save-secret --json`，同步新的 `CLINK_WEBHOOK_SIGNING_KEY`，并重启或重新部署服务。

Webhook handler 必须：

- 在 JSON 解析前保留 raw body。
- 读取 `X-Clink-Timestamp`。
- 读取 `X-Clink-Signature`。
- 使用 `X-Clink-Timestamp + "." + rawBody` 做 HMAC SHA-256 验签。
- 安全比较签名。
- 拒绝过期或重放请求。
- 幂等处理事件。
- 容忍重试和乱序投递。
- 使用 `merchantReferenceId` + `sessionId` 双重匹配本地订单；如果两个字段指向不同订单，应拒绝、隔离或升级处理。

Webhook 是支付和订阅状态同步的权威来源。`successUrl`、iframe 回调或前端 SDK 事件只能作为用户体验信号，不能作为最终付款确认。

## 验收标准

推荐验证顺序：

```bash
npm test
clink doctor --json
clink webhook simulate order.succeeded --secret env:CLINK_WEBHOOK_SIGNING_KEY --forward-to <webhook-url> --json
clink smoke-test --webhook-url <webhook-url> --json
```

然后再创建真实 sandbox checkout session：

```bash
clink checkout create \
  --customer-email buyer@example.com \
  --amount 10 \
  --currency USD \
  --name "Clink Sandbox Test" \
  --quantity 1 \
  --merchant-reference-id "clink-test-<timestamp>" \
  --success-url https://example.com/success \
  --cancel-url https://example.com/cancel \
  --json
```

真实支付验收必须确认：

- 用户打开 `checkoutUrl` 并完成 sandbox 测试支付。
- Webhook 签名验证通过。
- 本地订单通过 `merchantReferenceId` + `sessionId` 匹配并变为 paid/completed。
- 权益、发货、额度、下载权限或其他 fulfillment 已完成。

Webhook 返回 200 只表示传输成功，不代表本地订单和业务交付已经完成。

## 与 Skill 和 Prompt 的关系

支持 skills 的 agent 应先安装或读取：

```text
https://github.com/5048429/agent-prompts/tree/main/skills/clink-integ-skills
```

然后直接执行 `$clink-integ-skills` 所定义的流程。

不支持 skills 的 agent 可使用 fallback 详细规则：

```text
https://raw.githubusercontent.com/5048429/agent-prompts/main/clink-ai-auto-integration.zh-CN.md
```

给任意建站 agent 的一段式提示词：

```text
https://raw.githubusercontent.com/5048429/agent-prompts/main/clink-universal-website-agent-prompt.zh-CN.md
```

## 安全注意事项

- 不要把 Secret Key、webhook signing key 或 Dashboard token 写入源码、README、公开日志、测试 fixture、前端变量或最终回复。
- Secret Key 只应存在于服务端环境变量、平台 Secret、secret manager 或被 git 忽略的本地 `.env`。
- Webhook signing key 不应由用户初始提供，应由 CLI 自动创建、保存、同步或轮换。
- 前端只能使用 publishable key 或自己的后端接口，不能直接调用 Clink Secret Key API。
- 真实支付完成必须以服务端 webhook 和订单状态为准，而不是只看前端跳转或 webhook HTTP 200。
