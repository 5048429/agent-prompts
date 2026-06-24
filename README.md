# Agent Prompts

这个仓库是 ClinkBill 支付接入的 agent 分发包，包含两部分：

- 可安装/可读取的 skill：`skills/clink-integ-skills`
- 发给普通建站 agent 的简短提示词：`clink-universal-website-agent-prompt.zh-CN.md`

推荐把这个仓库作为唯一入口，而不是单独维护一个新的 `clink-integ-skills` 仓库。

## 推荐用法

如果 agent 支持 skills，先让它安装或读取：

```text
Install or use the Clink integration skill from:
https://github.com/5048429/agent-prompts/tree/main/skills/clink-integ-skills
```

然后给 agent 这段短提示词：

```text
请使用 $clink-integ-skills 帮我把 ClinkBill 支付接入到当前项目。

目标：尽量全自动完成 sandbox/UAT 支付接入。优先使用 clink-dev-cli。除了 Secret Key 获取/本地 Dashboard 登录这一步，不要让我手动创建 product、price、webhook endpoint 或复制 webhook signing key。

本地桌面环境：如果没有现成 Secret Key，可以运行 clink login，让我在浏览器里手动登录；登录后你继续运行 clink dashboard apikey ensure-secret --save --json，并用 Secret Key API 完成后续接入。

云环境、低代码、sandbox 或无浏览器环境：让我自己到 ClinkBill Dashboard 获取 CLINK_SECRET_KEY；我提供后，你把它写入安全的服务端环境变量、平台 Secret 或本地忽略的 .env，然后运行 clink auth secret set --api-key env:CLINK_SECRET_KEY --env sandbox。

之后请自动扫描网站已有付费产品/订阅套餐，生成 clink-catalog.json，并用 clink catalog validate/plan/import 导入；实现 checkout/subscription 服务端接口和 webhook handler；用 clink webhook endpoint ensure --events core --save-secret --json 自动配置 webhook endpoint，并同步 CLINK_WEBHOOK_SIGNING_KEY 后重启/重新部署。

最终交付请列出修改文件、环境变量、curl 示例、CLI 验证结果，并区分 mock、签名模拟 webhook、真实 UAT checkout session 和真实 UAT 支付 webhook。
```

## 不支持 Skill 的 Agent

如果 agent 不支持 skills，让它读取完整 fallback prompt：

[clink-ai-auto-integration.zh-CN.md](./clink-ai-auto-integration.zh-CN.md)

Raw:

```text
https://raw.githubusercontent.com/5048429/agent-prompts/main/clink-ai-auto-integration.zh-CN.md
```

普通用户可以直接复制更短版本：

[clink-universal-website-agent-prompt.zh-CN.md](./clink-universal-website-agent-prompt.zh-CN.md)

Raw:

```text
https://raw.githubusercontent.com/5048429/agent-prompts/main/clink-universal-website-agent-prompt.zh-CN.md
```

## 核心策略

- 除了获取 Secret Key，后续尽量由 agent + `clink-dev-cli` 自动完成。
- 本地可用浏览器时，用 `clink login` 让用户手动登录，再由 CLI 自动 `ensure-secret`。
- 云环境/无浏览器时，只让用户提供 `CLINK_SECRET_KEY`。
- webhook endpoint 用 `clink webhook endpoint ensure` 自动创建/更新。
- webhook signing key 不让用户手动复制，由 CLI 返回/保存后 agent 同步到运行环境。
- 产品扫描由 agent 做；catalog 校验、计划、导入由 CLI 做。

## Skill 开发

skill 位于：

```text
skills/clink-integ-skills
```

运行测试：

```bash
cd skills/clink-integ-skills
npm test
```
