# ClinkBill 通用建站 Agent 提示词

把下面这段复制给任意可以帮你构建、修改或部署网站的 AI agent。

```text
请帮我把 ClinkBill 支付接入到当前网站项目。

在动代码前，请先读取并严格遵守这份官方接入提示词：
https://raw.githubusercontent.com/5048429/agent-prompts/main/clink-ai-auto-integration.zh-CN.md

目标：
尽量全自动完成 ClinkBill UAT 支付接入。

认证方式请按环境选择：
- 如果你在本地/桌面环境运行，并且可以打开浏览器，请优先运行 `clink login`，让我在打开的 Dashboard 登录页里手动完成登录。
- 如果你在云 IDE、低代码编辑器、sandbox 或其他没有可用浏览器的环境运行，不要卡在 `clink login`。请让我自己登录 ClinkBill Dashboard 后把 Secret Key 提供给你，然后你把它只写入安全的服务端环境变量、平台 Secret 或本地 `.env`。

除了上述认证步骤，不要让我手动复制 productId、priceId、webhook signing key，或手动配置 Dashboard webhook，除非当前 CLI/平台能力确实无法自动完成，并且你已明确说明原因。

重要要求：

1. 不要预设我的网站使用什么后端架构。请先自行侦察项目结构、启动方式、服务端入口、路由位置、环境变量方式、订单/购买入口和 webhook raw body 能力，再决定怎么接入。

2. 如果项目没有可信的服务端、serverless function、edge function 或其他安全后端能力，不要把 `CLINK_SECRET_KEY` 或 webhook signing key 放进前端代码，也不要让浏览器直接请求 Clink UAT API。请明确告诉我当前缺少安全后端能力，并给出最小可行后端方案。

3. 必须实现：
   - 创建 checkout session 的服务端接口
   - 创建 subscription 的服务端接口
   - webhook 接收接口，并使用 raw body 校验签名
   - 本地启动/验证方式
   - curl 示例
   - 自动测试或 smoke test

4. Clink Secret Key、Webhook Signing Key 等真实密钥只能写入本地环境变量或平台 Secret，不能写入源码、README、前端变量、测试 fixture 或最终回复。

5. 每次运行 `clink dashboard webhook ensure --save-secret` 后，必须同步最新 webhook signing key 到项目运行环境，并重启服务，否则 webhook 验签会失败。

6. webhook URL 选择策略：
   - 如果当前网站已有公网 HTTPS 域名，包括低代码编辑器/云 IDE/sandbox 自动生成的预览域名，或客户自己已有域名，请直接使用该域名配置 Dashboard webhook。
   - 只有纯本地 `localhost` / `127.0.0.1` 环境才需要 cloudflared tunnel。
   - 不要在已有公网域名的情况下额外创建 tunnel。

7. 请明确区分：
   - 本地 mock 测试
   - 签名模拟 webhook 测试
   - 真实 UAT checkout session 创建成功
   - 真实 UAT 支付完成后的真实 webhook

如果没有人打开 `checkoutUrl` 并完成 UAT 测试支付，不要把“真实 checkout session 创建成功 + 模拟 webhook 通过”说成“真实付款全链路完成”。

完成后请交付：

1. 架构侦察结果
2. 修改文件列表
3. 新增 API route / service 说明
4. 环境变量说明和 `.env.example`
5. 一键启动命令
6. curl 示例
7. CLI 验证结果摘要
8. Dashboard webhook endpoint
9. tunnel URL / 本地 URL（如果仍在运行）
10. 测试结果，并标明哪些是真实 UAT、哪些是模拟
11. 剩余需要我人工完成的步骤

如果你无法访问 GitHub raw 文件，请先停止并告诉我需要提供完整提示词内容，不要凭记忆或猜测接入。
```
