# ClinkBill AI 自动接入提示词

帮我把 ClinkBill 支付接入到当前项目。

如果你支持 skills，请先安装或读取 skill，然后立即使用它完成接入，不要要求用户再发送第二段提示词：

```text
https://github.com/5048429/agent-prompts/tree/main/skills/clink-integ-skills
```

安装/读取后使用：

```text
$clink-integ-skills
```

如果你不支持 skills，请严格遵守下面的 fallback 规则。

## 目标

尽量全自动完成 sandbox/UAT 支付接入。除了 Secret Key 获取或本地 Dashboard 登录这一步，不要让我手动创建 product、price、webhook endpoint，也不要让我手动复制 webhook signing key。

请保持真实、可验证：如果没有人打开 `checkoutUrl` 并完成真实 UAT 测试支付，不要把“真实 checkout session 创建成功 + 签名模拟 webhook 通过”说成“真实付款全链路完成”。

## CLI

安装并使用最新 `clink-dev-cli`：

```bash
npm install -g github:5048429/clink-dev-cli
clink auth secret set --help
clink api request --help
clink catalog import --help
clink webhook endpoint ensure --help
```

如果全局安装失败，可以本地安装：

```bash
npm install --prefix ./.clink-tools github:5048429/clink-dev-cli
```

后续命令中的 `clink` 可替换为本地二进制路径。

## 认证路径

根据运行环境选择。

### 本地桌面环境

如果没有现成 Secret Key，且可以打开浏览器：

```bash
clink login
```

暂停并让我在打开的浏览器中手动完成 ClinkBill Dashboard 登录。不要自动输入密码、MFA、验证码或绕过登录。

我确认登录完成后继续：

```bash
clink dashboard whoami --json
clink dashboard apikey ensure-secret --save --json
clink auth status --json
```

如果项目运行时也需要 `CLINK_SECRET_KEY`，只能在受控的本地 secret 写入步骤里使用：

```bash
clink dashboard apikey ensure-secret --save --show-secret --json
```

把值写入被 git 忽略的 `.env`、平台 Secret 或 secret manager。不要把真实 key 写进源码、README、测试 fixture、前端变量、公开日志或最终回复。

### 云环境、低代码、sandbox 或无浏览器环境

不要卡在 `clink login`。请让我自己登录 ClinkBill Dashboard 并提供 `CLINK_SECRET_KEY`。

拿到后只写入安全的服务端环境变量、平台 Secret 或本地忽略的 `.env`，然后运行：

```bash
clink auth secret set --api-key env:CLINK_SECRET_KEY --env sandbox
clink auth status --json
clink doctor --json
```

初始阶段不要向我索取 `CLINK_WEBHOOK_SIGNING_KEY`。

## 架构侦察

动代码前先确认：

- 项目启动方式、框架和部署方式
- 是否有可信服务端、serverless function 或 edge function
- API route 位置
- 环境变量注入方式
- 购买入口、订单模型、发货/权益逻辑
- webhook route 是否能拿到 raw body

如果没有可信后端，不要把 `CLINK_SECRET_KEY` 或 webhook signing key 放进前端，也不要让浏览器直接请求 Clink API。请说明缺口，并提出最小安全后端方案。

## Product Catalog

如果网站、CMS、数据库、源码或价格页已经存在付费产品、订阅套餐、价格、币种或计费周期：

1. 由你扫描这些来源。
2. 生成确定性的 `clink-catalog.json`，每个 product/price 都要有稳定 `sourceId`。
3. 用 CLI 校验、预览并导入：

```bash
clink catalog validate --file ./clink-catalog.json --json
clink catalog plan --file ./clink-catalog.json --mapping-file ./.clink/catalog-map.json --json
clink catalog import --file ./clink-catalog.json --mapping-file ./.clink/catalog-map.json --json
```

不要让我手动复制 `productId` 或 `priceId`，除非 CLI/平台能力确实失败，并且你已说明原因。

## Checkout / Subscription

实现服务端接口：

- checkout session 创建
- subscription 创建或订阅购买路径
- webhook 接收和验签

服务端请求 Clink 时必须带：

```text
X-API-Key: <CLINK_SECRET_KEY>
X-Timestamp: <milliseconds timestamp>
Content-Type: application/json
```

`merchantReferenceId` 只用于对账，不是幂等键。支付成功、订阅状态和退款状态以 webhook / 服务端查询为准，不要把 `successUrl` 当最终确认。

## Webhook 自动配置

先实现并部署可公网访问的 HTTPS webhook route。已有公网 HTTPS 域名时直接使用该域名；只有纯本地 `localhost` 才需要 tunnel。

配置 webhook endpoint：

```bash
clink webhook endpoint ensure \
  --url <public-webhook-url>/api/clink/webhook \
  --events core \
  --save-secret \
  --json
```

如果需要把 signing key 写入平台 Secret，并且你有平台 Secret 写入能力，可以在受控步骤中使用：

```bash
clink webhook endpoint ensure \
  --url <public-webhook-url>/api/clink/webhook \
  --events core \
  --save-secret \
  --show-secret \
  --json
```

成功后必须：

1. 把返回或轮换后的 signing key 同步到项目运行环境的 `CLINK_WEBHOOK_SIGNING_KEY`。
2. 重启或重新部署服务。
3. 使用签名模拟 webhook 验证 handler。

每次 webhook URL、预览域名或 path 变化，都要重新运行 `ensure --save-secret --json`，重新同步 `CLINK_WEBHOOK_SIGNING_KEY` 并重启/重新部署。

`clink dashboard webhook ensure` 只是兼容别名，新接入优先使用 `clink webhook endpoint ensure`。不要把 webhook endpoint 管理描述成 Dashboard-only。

## Webhook Handler

handler 必须：

- 保留 raw body，再解析 JSON
- 读取 `X-Clink-Timestamp`
- 读取 `X-Clink-Signature`
- 用 `X-Clink-Timestamp + "." + rawBody` 做 HMAC SHA-256 验签
- 安全比较签名
- 防重放
- 幂等处理
- 支持 retry
- 容忍乱序事件

核心事件通常包括：

- `session.complete`
- `order.succeeded`
- `order.failed`
- `refund.succeeded`
- `subscription.created`
- `invoice.paid`

## 验证顺序

推荐顺序：

```bash
npm test
clink doctor --json
clink webhook simulate order.succeeded --secret env:CLINK_WEBHOOK_SIGNING_KEY --forward-to <webhook-url> --json
clink smoke-test --webhook-url <webhook-url> --json
```

然后创建真实 UAT checkout session。只有当有人打开 `checkoutUrl` 并完成 UAT 测试支付后，才能报告真实 UAT 支付 webhook 已完成。

## 最终交付

请交付：

1. 架构侦察结果
2. 修改文件列表
3. 新增 API route / service 说明
4. `.env.example`
5. 启动命令
6. curl 示例
7. CLI 验证结果摘要
8. webhook endpoint URL
9. tunnel URL 或本地 URL，如果仍在运行
10. 测试结果，并区分 mock、签名模拟 webhook、真实 UAT checkout session、真实 UAT 支付 webhook
11. 剩余人工步骤

预期剩余人工步骤通常只有：

- 本地桌面环境：在 `clink login` 打开的浏览器里手动登录
- 云/无浏览器环境：提供一次 `CLINK_SECRET_KEY`
- 需要验证真实支付时：手动打开 `checkoutUrl` 完成 UAT 测试支付
