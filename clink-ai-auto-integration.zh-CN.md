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

安装并使用最新 `clink-dev-cli`。Agent 默认安装到项目本地，避免全局 npm 目录权限、旧版本残留或 Windows junction 文件锁问题：

```bash
npm install --prefix ./.clink-tools github:5048429/clink-dev-cli
./.clink-tools/node_modules/.bin/clink auth secret set --help
./.clink-tools/node_modules/.bin/clink api request --help
./.clink-tools/node_modules/.bin/clink catalog import --help
./.clink-tools/node_modules/.bin/clink webhook endpoint ensure --help
```

Windows PowerShell 使用：

```powershell
.\.clink-tools\node_modules\.bin\clink.cmd auth secret set --help
.\.clink-tools\node_modules\.bin\clink.cmd api request --help
.\.clink-tools\node_modules\.bin\clink.cmd catalog import --help
.\.clink-tools\node_modules\.bin\clink.cmd webhook endpoint ensure --help
```

如果你确认当前机器全局 npm 可用，也可以全局安装；全局安装必须带 `--install-links=true`：

```bash
npm install -g --install-links=true github:5048429/clink-dev-cli
clink auth secret set --help
clink api request --help
clink catalog import --help
clink webhook endpoint ensure --help
```

如果安装失败，请说明脱敏错误，不要因为缺少 Node 类型声明就在业务项目里补 TypeScript 构建依赖。GitHub 安装应使用 CLI 仓库已提交的 `dist/` 产物。

如果需要重试，请仍优先使用本地安装：

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

1. 由你按“运行中的 API / 价格页 DOM / hydrated JSON > 源码 / 配置 / seed / public-static 资产 > 询问用户”的顺序扫描这些来源。
2. 生成确定性的 `clink-catalog.json`，每个 product/price 都要有稳定 `sourceId`。
3. 每个 product 必须包含且只包含一个图片来源：`imageId`、`imageUrl` 或 `imageFile`。URL 必须写 `imageUrl`，本地 public/static 图片写 `imageFile`，不要把 URL 写进 `imageId`。
4. 用 CLI 校验、预览并导入：

```bash
clink catalog validate --file ./clink-catalog.json --project-root . --public-dir public --json
clink catalog plan --file ./clink-catalog.json --mapping-file ./.clink/catalog-map.json --project-root . --public-dir public --json
clink catalog import --file ./clink-catalog.json --mapping-file ./.clink/catalog-map.json --project-root . --public-dir public --json
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

### 云端托管 / 低代码平台硬规则

如果当前项目运行在云端托管平台、低代码平台、云 IDE、sandbox 或类似无浏览器托管环境，并且 `CLINK_SECRET_KEY` 已经在平台 Secret 中配置：

- 不要要求用户再到本地终端运行 `scripts/clink-bootstrap.sh` 来复制 `CLINK_WEBHOOK_SIGNING_KEY`。
- 不要把“脚本会打印 signing key，请用户粘贴到平台 Secret”作为正常最终交付。
- 先在 agent 环境安装项目内 CLI，并确认 `clink webhook endpoint ensure --help` 支持 `--show-secret` 和 `--sync-env-file`。
- 使用平台已配置的 `CLINK_SECRET_KEY`，或在受控的一次性命令环境里让用户只提供 `CLINK_SECRET_KEY`，运行 `clink auth secret set --api-key env:CLINK_SECRET_KEY --env sandbox`。
- 部署包含 webhook route 的版本，拿到公网 HTTPS webhook URL。
- 运行 `clink webhook endpoint ensure --url <public-webhook-url>/api/clink/webhook --events core --save-secret --show-secret --json`。
- agent 如果有平台 Secret 写入能力，必须自己把返回或轮换得到的 signing key 写入平台后端 Secret：`CLINK_WEBHOOK_SIGNING_KEY`，然后重新发布/重启后端。
- 只有当平台不允许 agent 写入 Secret、也没有可用平台 Secret API 时，才把“请用户把这一个 signing key 写入平台后端 Secret 并重新发布”列为阻塞的人类步骤，并明确说明这是平台写入权限限制，不是 Clink CLI 能力缺失。

`CLINK_CATALOG_MAP` 不是密钥。优先把 catalog mapping 写入仓库内受控文件，例如 `.clink/catalog-map.json`，或写入后端可读取的配置；不要让用户手动复制粘贴 catalog map，除非平台没有可写文件/配置能力并且已经说明原因。

配置 webhook endpoint：

```bash
clink webhook endpoint ensure \
  --url <public-webhook-url>/api/clink/webhook \
  --events core \
  --save-secret \
  --sync-env-file .env.local \
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

本地 `.env` 项目优先使用 `--sync-env-file <env-file>` 自动写入；只有平台 Secret 写入权限缺失时，才把写入 signing key 作为剩余人工步骤。

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
- 使用 `merchantReferenceId` + `sessionId` 双重匹配本地订单；如果两个字段指向不同本地订单，必须拒绝、隔离或升级处理，不能只依赖其中一个字段

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

然后创建真实 UAT checkout session。只有当有人打开 `checkoutUrl` 并完成 UAT 测试支付，且你确认本地订单 paid/completed、额度/权益/发货/下载权限或其他 fulfillment 已完成后，才能报告真实 UAT 支付全链路完成。webhook 返回 200 只代表传输成功，不代表本地订单和业务交付已经完成。

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
