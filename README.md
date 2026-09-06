# 钉钉工作台助手

在 DesireCore 里用自然语言操作钉钉：查通讯录、收发消息、排日程、跟进待办与审批、
读写文档表格、管钉盘知识库、查考勤日志、看 AI 听记。

> **这个 Agent 提供的是编排能力，不是钉钉产品本身。**
> 它不捆绑、不授权、不安装、不代付钉钉 / DingTalk。你需要自备钉钉组织、
> 自行安装官方 CLI（`dws`）并完成授权。钉钉是独立授权的第三方 SaaS，
> `dingtalk-workspace-cli` 是独立的第三方命令行工具，二者的许可条款与费用由你与钉钉之间约定。

安装前的前置条件、五步配置、审批模式取舍与能力边界，见 [USAGE.md](./USAGE.md)。

## 仓库结构

| 路径 | 内容 |
|---|---|
| `agent.json` | Agent 配置（AgentFS `agent-config` schema） |
| `persona.md` | 人格描述 |
| `principles.md` | 执行纪律：写操作判据、标识符处置、降级规则 |
| `USAGE.md` | 使用说明，市场详情页的「使用说明」区块从这里取 |
| `CHANGELOG.md` | 变更历史 |
| `skills/dingtalk-guide/` | 私有技能：13 篇产品域参考文档，Agent 按需加载 |
| `skills/dingtalk-onboarding/` | 私有技能：安装与授权引导 |
| `skills/dingtalk-workflows/` | 私有技能：跨产品编排流程 |

三个技能随 Agent 一起安装到它的私有技能目录，不进全局技能目录。

## 市场条目

本仓库是内容事实源，市场条目 [`desirecore/market`](https://github.com/desirecore/market)
的 `agents/dingtalk-workspace/` 只保留 pointer（`entry.json` + `catalog-metadata.v1.json`
+ 头像），按不可变 commit ref 指向这里。

改动流程：本仓库合并 → market 条目重新 pin 到新 ref。

## 反馈

能力边界、命令行为与权限模型以钉钉官方 CLI 为准。本 Agent 的编排逻辑、澄清策略与
安全约束由 DesireCore 维护。
