# ClinkBill 最快接入体验版 Agent 提示词

把下面这段复制给任意可以帮你构建、修改或部署网站的 AI agent。

这个版本的目标是：**最快让网站跑通 ClinkBill hosted checkout，让用户能点击购买并进入 ClinkBill 收银台**。不要一开始就做过重的测试矩阵、subscription、refund、复杂后台管理或完整事件系统。

```text
请帮我把 ClinkBill UAT 支付快速接入到当前网站项目。

目标：
用最少改动完成一个可体验的购买流程：用户在网站点击购买 -> 服务端创建 ClinkBill checkout session -> 前端跳转 ClinkBill hosted checkout -> 支付结果通过 webhook 最小处理。

优先级：
先让 checkout 跑起来，再做增强。不要一开始就做 subscription、refund、复杂订单后台、完整测试矩阵或重型重构。

请按下面流程执行：

1. 快速识别项目结构
   - 找到启动命令、前端购买按钮、商品价格来源、服务端/API 入口、环境变量方式。
   - 不要预设框架。
   - 如果没有任何安全后端能力，不要把 Clink Secret Key 放进前端；请停止并告诉我需要一个最小后端/API function。

2. 安装或使用 Clink CLI
   - 如果没有 `clink`，安装：
     npm install -g github:5048429/clink-dev-cli
   - 验证：
     clink --version

3. 登录并自动保存密钥
   - 运行：
     clink login
   - 如果打开 Dashboard 登录页，请让我手动登录，登录完成后继续。
   - 然后运行：
     clink dashboard apikey ensure-secret --save --json
     clink auth status --json
   - 把 Secret Key 写入本地环境变量或平台 Secret，不要写入前端代码、README、公开仓库或最终回复。

4. 实现最小 checkout
   - 新增一个服务端接口，例如：
     POST /api/clink/checkout
   - 前端购买按钮调用这个接口。
   - 服务端使用本地商品的名称、价格、数量、邮箱创建 ClinkBill checkout session。
   - 优先使用临时 `priceDataList`，不要为了快速体验强制创建 Clink product/price。
   - 服务端请求 Clink UAT API：
     POST https://uat-api.clinkbill.com/api/checkout/session
   - 请求头包含：
     X-API-Key: <CLINK_SECRET_KEY>
     X-Timestamp: <milliseconds timestamp>
     Content-Type: application/json
   - 用本地订单号或购买记录 ID 作为 `merchantReferenceId`。
   - 返回 checkout URL 给前端。
   - 兼容这些返回字段：
     checkoutUrl、url、sessionUrl、data.checkoutUrl、data.url、data.sessionUrl。
   - 前端拿到 checkout URL 后立即跳转。

5. 实现最小 webhook
   - 新增一个服务端接口，例如：
     POST /api/clink/webhook
   - 必须读取 raw body 后校验签名，不要先 JSON parse。
   - 签名规则：
     HMAC_SHA256(CLINK_WEBHOOK_SIGNING_KEY, X-Clink-Timestamp + "." + rawBody)
   - 最小处理这些事件即可：
     session.complete：记录 checkout 已完成，不要只凭它发货
     order.succeeded：标记本地订单已支付/完成，触发已有发货或成功态
     order.failed：标记支付失败
   - 其他事件可以先记录日志，不要阻塞第一次体验。

6. 本地或部署环境 webhook
   - 如果项目已经有公网部署 URL，直接配置 Dashboard webhook。
   - 如果只是本地体验，并且需要验证 webhook，用 cloudflared 暴露本地端口：
     cloudflared tunnel --url http://127.0.0.1:<PORT> --no-autoupdate
   - 如果 QUIC/UDP 失败或公网 URL 报 1033，改用：
     cloudflared tunnel --url http://127.0.0.1:<PORT> --no-autoupdate --protocol http2
   - 配置 webhook：
     clink dashboard webhook ensure --url <PUBLIC_URL>/api/clink/webhook --events core --save-secret --json
   - 注意：运行 `webhook ensure --save-secret` 后，要把最新 webhook signing key 同步到环境变量，并重启服务。

7. 最小验证即可
   必须验证：
   - 网站能启动
   - 点击购买能创建 checkout session
   - 前端能跳转到 ClinkBill hosted checkout
   - webhook 签名模拟能返回 200（如果已配置 webhook/tunnel）

   可以使用：
   clink webhook simulate order.succeeded --secret env:CLINK_WEBHOOK_SIGNING_KEY --forward-to <WEBHOOK_URL> --json

   不要为了第一次体验强制完成：
   - subscription
   - refund
   - invoice.paid
   - 完整事件矩阵
   - 大量端到端测试
   - 大规模重构

8. 真实性说明
   如果我没有打开 checkoutUrl 并完成 UAT 测试支付，不要说“真实付款全链路完成”。
   你可以说：
   - 真实 UAT checkout session 已创建
   - 前端已可跳转 ClinkBill checkout
   - webhook 签名模拟已通过
   - 等我完成 UAT 支付后，真实 order.succeeded webhook 会触发订单完成逻辑

完成后请简短交付：

1. 我现在怎么启动网站
2. 我点击哪里体验 ClinkBill checkout
3. 新增/修改了哪些文件
4. 新增了哪些环境变量
5. checkout API 和 webhook API 地址
6. 已完成的最小验证结果
7. 还剩哪些可选增强，比如 subscription、refund、更多 webhook 事件、生产部署

安全底线：
- CLINK_SECRET_KEY 只能在服务端使用
- CLINK_WEBHOOK_SIGNING_KEY 只能在服务端使用
- 不要把真实密钥写进前端、README、公开日志或最终回复
- 不要让浏览器直接请求 Clink UAT API
```

