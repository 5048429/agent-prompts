# Agent Prompts

这个仓库是 ClinkBill 支付接入的 agent 分发包，包含：

- `skills/clink-integ-skills`：可安装/可读取的 Clink 支付接入 skill
- `clink-universal-website-agent-prompt.zh-CN.md`：复制给任意建站 agent 的一段式提示词
- `clink-ai-auto-integration.zh-CN.md`：不支持 skill 时使用的 fallback 详细规则

## 一段式提示词

用户只需要把下面这一段发给 agent。agent 应该自己安装/读取 skill，然后立即按 skill 开始接入，不要要求用户再发第二段提示词。

```text
请先安装或读取 Clink 支付接入 skill：
https://github.com/5048429/agent-prompts/tree/main/skills/clink-integ-skills

安装/读取完成后，立即使用 $clink-integ-skills 帮我把 ClinkBill 支付接入到当前项目。

目标：尽量全自动完成 sandbox/UAT 支付接入。优先使用 clink-dev-cli。除了 Secret Key 获取或本地 Dashboard 登录这一步，不要让我手动创建 product、price、webhook endpoint，也不要让我手动复制 webhook signing key。

本地桌面环境：如果没有现成 Secret Key，可以运行 clink login，让我在浏览器里手动登录；登录后你继续运行 clink dashboard apikey ensure-secret --save --json，并用 Secret Key API 完成后续接入。

云环境、低代码、sandbox 或无浏览器环境：让我自己到 ClinkBill Dashboard 获取 CLINK_SECRET_KEY；我提供后，你把它写入安全的服务端环境变量、平台 Secret 或本地忽略的 .env，然后运行 clink auth secret set --api-key env:CLINK_SECRET_KEY --env sandbox。

之后请按“运行中 API/价格页 DOM > 源码/配置 > 询问用户”的顺序自动扫描网站已有付费产品/订阅套餐，生成 clink-catalog.json；每个 product 必须包含 imageId、imageUrl 或 imageFile 之一。用 clink catalog validate/plan/import 导入；实现 checkout/subscription 服务端接口和 webhook handler；用 clink webhook endpoint ensure --events core --save-secret --json 自动配置 webhook endpoint，并同步 CLINK_WEBHOOK_SIGNING_KEY 后重启/重新部署。本地 .env 项目优先使用 --sync-env-file。webhook handler 必须用 merchantReferenceId + sessionId 双重匹配订单；真实支付验收必须确认本地订单 paid/completed 和权益/发货等 fulfillment 完成，不能只看 webhook 200。

如果你的运行环境不支持安装或读取 skill，请改为读取并遵守这份 fallback 规则：
https://raw.githubusercontent.com/5048429/agent-prompts/main/clink-ai-auto-integration.zh-CN.md

最终交付请列出修改文件、环境变量、curl 示例、CLI 验证结果，并区分 mock、签名模拟 webhook、真实 UAT checkout session 和真实 UAT 支付 webhook。
```

同一段也保存在：

[clink-universal-website-agent-prompt.zh-CN.md](./clink-universal-website-agent-prompt.zh-CN.md)

Raw:

```text
https://raw.githubusercontent.com/5048429/agent-prompts/main/clink-universal-website-agent-prompt.zh-CN.md
```

## Skill 位置

```text
skills/clink-integ-skills
```

运行测试：

```bash
cd skills/clink-integ-skills
npm test
```
