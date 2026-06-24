# ClinkBill 一段式建站 Agent 提示词

把下面这一段直接复制给任意可以帮你构建、修改或部署网站的 AI agent。它会同时要求 agent 安装/读取 skill，并立刻按 skill 接入 Clink 支付。

```text
请先安装或读取 Clink 支付接入 skill：
https://github.com/5048429/agent-prompts/tree/main/skills/clink-integ-skills

安装/读取完成后，立即使用 $clink-integ-skills 帮我把 ClinkBill 支付接入到当前项目。

目标：尽量全自动完成 sandbox/UAT 支付接入。优先使用 clink-dev-cli。除了 Secret Key 获取或本地 Dashboard 登录这一步，不要让我手动创建 product、price、webhook endpoint，也不要让我手动复制 webhook signing key。

本地桌面环境：如果没有现成 Secret Key，可以运行 clink login，让我在浏览器里手动登录；登录后你继续运行 clink dashboard apikey ensure-secret --save --json，并用 Secret Key API 完成后续接入。

云环境、低代码、sandbox 或无浏览器环境：让我自己到 ClinkBill Dashboard 获取 CLINK_SECRET_KEY；我提供后，你把它写入安全的服务端环境变量、平台 Secret 或本地忽略的 .env，然后运行 clink auth secret set --api-key env:CLINK_SECRET_KEY --env sandbox。

之后请自动扫描网站已有付费产品/订阅套餐，生成 clink-catalog.json，并用 clink catalog validate/plan/import 导入；实现 checkout/subscription 服务端接口和 webhook handler；用 clink webhook endpoint ensure --events core --save-secret --json 自动配置 webhook endpoint，并同步 CLINK_WEBHOOK_SIGNING_KEY 后重启/重新部署。

如果你的运行环境不支持安装或读取 skill，请改为读取并遵守这份 fallback 规则：
https://raw.githubusercontent.com/5048429/agent-prompts/main/clink-ai-auto-integration.zh-CN.md

最终交付请列出修改文件、环境变量、curl 示例、CLI 验证结果，并区分 mock、签名模拟 webhook、真实 UAT checkout session 和真实 UAT 支付 webhook。
```
