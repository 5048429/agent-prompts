# ClinkBill 通用建站 Agent 短提示词

把下面这段复制给任意可以帮你构建、修改或部署网站的 AI agent。若该 agent 支持 skills，请先要求它安装或使用：

```text
https://github.com/5048429/agent-prompts/tree/main/skills/clink-integ-skills
```

然后发送：

```text
请使用 $clink-integ-skills 帮我把 ClinkBill 支付接入到当前网站项目。

目标：尽量全自动完成 sandbox/UAT 支付接入。优先使用 clink-dev-cli。除了 Secret Key 获取/本地 Dashboard 登录这一步，不要让我手动创建 product、price、webhook endpoint 或复制 webhook signing key。

本地桌面环境：如果没有现成 Secret Key，可以运行 clink login，让我在浏览器里手动登录；登录后你继续运行 clink dashboard apikey ensure-secret --save --json，并用 Secret Key API 完成后续接入。

云环境、低代码、sandbox 或无浏览器环境：让我自己到 ClinkBill Dashboard 获取 CLINK_SECRET_KEY；我提供后，你把它写入安全的服务端环境变量、平台 Secret 或本地忽略的 .env，然后运行 clink auth secret set --api-key env:CLINK_SECRET_KEY --env sandbox。

之后请自动扫描网站已有付费产品/订阅套餐，生成 clink-catalog.json，并用 clink catalog validate/plan/import 导入；实现 checkout/subscription 服务端接口和 webhook handler；用 clink webhook endpoint ensure --events core --save-secret --json 自动配置 webhook endpoint，并同步 CLINK_WEBHOOK_SIGNING_KEY 后重启/重新部署。

最终交付请列出修改文件、环境变量、curl 示例、CLI 验证结果，并区分 mock、签名模拟 webhook、真实 UAT checkout session 和真实 UAT 支付 webhook。
```

如果 agent 不支持 skills，让它先读取完整 fallback prompt：

```text
https://raw.githubusercontent.com/5048429/agent-prompts/main/clink-ai-auto-integration.zh-CN.md
```
