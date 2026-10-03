# 钉钉工作台助手 · 开发与维护指引

> 本文件给**接手维护这个 Agent 的智能体或开发者**看，与 `CLAUDE.md` 内容相同（两份必须保持同步）。
> DesireCore 运行时**不读取**本仓库里的这两个文件（它只读用户工作目录里的 `AGENTS.md`），所以这里写的是维护指引，不是 Agent 的行为规则——行为规则在 `persona.md` 和 `principles.md`。

---

## 1. 这个仓库是什么

DesireCore 官方市场里「钉钉工作台助手」Agent 的**内容仓库**。它把自然语言意图翻译成钉钉官方 CLI（`dws`）的正确调用，覆盖钉钉 29 个产品 / 1256 个工具。

它**不是**钉钉命令的说明书——命令目录由钉钉官方技能与 `dws schema` 提供，随二进制升级而更新。这个 Agent 做的是官方技能不覆盖的四件事：

| 增量 | 说明 |
| --- | --- |
| **接入** | 钉钉官方 CLI 的 Agent 分发列表里没有 DesireCore，装到 DesireCore 的路径无人覆盖 |
| **纪律** | 钉钉 CLI 有 339 个「自己不拦」的写操作（占全部写操作 56%），由本 Agent 把关 |
| **编排** | 跨产品工作流；官方 `SKILL.md` 明写「定时调度由外层工作流负责」 |
| **降级** | 没装 / 没授权 / 没权限 / 没开通时如实停下，绝不编造成功 |

## 2. 两个仓库的关系（改动前必须懂）

```
desirecore-agent/dingtalk-workspace   ← 本仓库：Agent 的全部内容
        ↑ 被指向（pin 到具体 commit）
desirecore/market
  └── agents/dingtalk-workspace/
        ├── entry.json                 ← 卡片：写着「内容在本仓库，版本 <SHA>」
        └── catalog-metadata.v1.json   ← 审核信息（证据六项）
```

**市场侧只有这两个文件，不能多。** 校验器 `scripts/catalog/validate_catalog_metadata.py:253-261` 用 `inline_path.is_file() == pointer_path.is_file()` 判定——`agent.json` 与 `entry.json` **必须恰好存在一个**，都有或都没有都会让条目从市场消失。

**改内容 → 改本仓库；发新版 → 改市场卡片的 pin。** 两者是分开的两步：

| 你要做的 | 动哪里 |
| --- | --- |
| 修文档、调人格、加技能 | 只改本仓库，推 `main` |
| 让新装与已安装的用户拿到新版 | 改市场卡片的 pin（见 §4；v10.0.177 及更早的客户端例外，见下） |

新安装装到的是卡片上 pin 的那个 commit，不是本仓库的最新 `main`。这是**有意为之**：中间有显式审核关口，上游随便改不该立刻影响已发布的用户。已安装用户的更新是否同样受这道关口约束，取决于客户端版本：

- **v10.0.177 之后的客户端**（包含 desirecore/desirecore#3728）：只会被更新到卡片当前的 pin。`main` 上领先 pin 的提交，在市场 repin PR 合并、客户端同步到新目录之前不会送达；条目被拦、下架或与本仓库对不上时，更新暂停。
- **v10.0.177 及更早的客户端**：仍跟随本仓库 `main` 的最新提交。`main` 上的 `agent.json#version` 一旦递增，这些用户最迟约 10 分钟内会被无人值守更新到 `main` 头，中间所有提交一并带上；版本号不变的提交不会单独推送，但会随下一次版本递增一起送达。

所以在旧客户端退出使用之前，**合进 `main` 仍等于对一部分已安装用户发布**：版本递增的提交只在市场 repin PR 已准备好时合入，并紧接着合并 repin。

## 3. 目录结构与各文件职责

```
agent.json          AgentFS 运行时配置（纯运行时字段，不含市场展示字段）
persona.md          人格（L0/L1/L2 三层）
principles.md       行为规则：「必须做」17 条 + 「禁止做」6 条 + L2 消歧/降级矩阵
USAGE.md            使用说明——市场详情页「使用说明」区块渲染的就是这个文件
README.md           仓库说明（给人看）
CHANGELOG.md        Keep a Changelog 格式，市场同步会解析成结构化 changelog
LICENSE / NOTICE    许可证据，sidecar 的 compliance 指向它们
assets/avatar.webp  头像图片，agent.json 的 avatar.image.path 指向它
skills/
├── dingtalk-onboarding/   安装、授权、自检、多组织（官方 14 技能不覆盖的部分）
├── dingtalk-workflows/    5 个跨产品 recipe（晨间简报 / 会议闭环 / 逾期巡检 / 周报 / 归档）
└── dingtalk-guide/        13 篇功能文档，Agent 按需加载
    ├── SKILL.md           正文里有 13 行显式索引（见 §8 第 2 条，缺了 Agent 就看不见文档）
    └── references/
```

**`agent.json` 里为什么没有 `category` / `i18n` / `persona` / `changelog`**：那些是市场展示字段，属于市场卡片 `entry.json`。两套 schema 互不相容（AgentFS 侧 `additionalProperties: false`），混写会让安装后的配置走宽松解析并刷 warning。DesireCore PR #2515 在安装时会把市场字段投影掉，但本仓库作为源头本就不该含它们。

**与官方 14 个钉钉技能零重叠。** 官方技能装在用户的**全局**技能目录，本仓库的 3 个私有技能只做官方不管的事。不要把官方技能内容复制进来——它们随 `dws upgrade` 更新，复制进来会过期。

## 4. 发新版：pin 要同时改三处

市场卡片 pin 一个 40 位 SHA。**改 ref 必须同时改三处，漏任何一处校验必红**：

| 文件 | 字段 |
| --- | --- |
| `entry.json` | `source.ref` |
| `catalog-metadata.v1.json` | `provenance.content.ref` |
| `catalog-metadata.v1.json` | `governance.compliance.reviewedRef` |

自检别按路径逐个查，**全文扫 40 位 hex** 确认三处一致：

```bash
grep -oE '[0-9a-f]{40}' agents/dingtalk-workspace/*.json | sort | uniq -c
```

同时更新 `timestamps.reviewedAt.value` 与 `compliance.reviewedAt`（两者必须相等）。

**顺序**：先推本仓库并用 `git ls-remote` 确认 SHA 可达，**再**改市场卡片。反了会造出指向不存在内容的死卡片，用户点安装报错。

市场 PR 前跑（`uv` 在 `~/.local/bin`，直接 `python3` 会缺 `yaml`）：

```bash
uv run --quiet scripts/catalog/validate_catalog_metadata.py --require-complete   # 要 0 error
uv run --quiet scripts/i18n/validate-i18n.py
```

`agents=N` 那个计数是关键信号——如果本条目被判非法，它会从计数里消失。

## 5. 硬性规则

### 5.1 公开信息边界（本仓库是公开的）

**禁止任何真实身份进入 tracked 文件**：姓名、手机号、邮箱、部门、群名、corpId、userId、组织名、文档标题、个人 HOME 路径。示例统一用「某某」，标识符用 `<corpId>` 这类占位符。

**截图尤其危险**。真机测试跑的是真实钉钉数据，任何返回业务内容的会话截图都会带 PII。只能截「不返回业务数据」的画面：命令审批卡片、空结果的确定性回答、降级说明、界面结构。截完必须逐张人工看过再入库。

推之前全树扫一遍：

```bash
grep -rInE "手机号正则|@[a-z0-9.-]+\.[a-z]{2,}|/Users/[a-z]+|corpId=|userId=" . --exclude-dir=.git
```

### 5.2 `USAGE.md` 不能放图片

市场详情页的 `MarkdownContent` 只解析 `dc-media://`，普通 `src` 被主动剥除——相对路径图片渲染不出来。截图只放在 `skills/dingtalk-guide/references/` 里给 Agent 读。

### 5.3 平台功能说明必须逐字引用平台文案

写审批模式、路由模式这类平台功能的说明时，**说明栏必须逐字引用 `app/data/i18n/zh-CN.ts` 里的 `description`，不得转述**。

这条是踩了三次同一个坑之后写下的：按模式 ID 字面（`ask-external` → 「问外部」）反推含义，结果写反了。七种审批模式的权威文案在 `USAGE.md` 的表格里，表头已标注「逐字引用平台内文案」，这一列不能自己改写。

### 5.4 不要把本机环境固化进公开分发物

`agent.json` 的 `llm` 保持 `smart` + `flagship` 默认。**不要**钉死到某个具体 Provider 的模型（如 `desirecore-cloud/deepseek-v4-flash-0731`）——那是本机为绕开配额临时改的，别人机器上未必有那个 Provider。

### 5.5 禁止把 `~/.desirecore` 写进代码或提示词

用 `${DESIRECORE_TEST_ROOT:-${DESIRECORE_HOME:-$HOME/.desirecore}}` 这样的形式。`~` 展开成真实 HOME，dev/测试实例照写字面量会读写到生产目录。

## 6. 设计决策与理由

### 为什么 principles 里的规则放在「必须做」编号列表而不是说明段

真机实测，**同一条规则写在不同位置效果完全不同**：

| 位置 | 效果 |
| --- | --- |
| L2 说明段里的一句话 | 无效 |
| L1 里的判据表（逐字包含失败用例本身作为例子） | 仍无效 |
| **提升为「必须做」段落的编号祈使规则** | **生效** |

所以改 principles 时，重要纪律一律放进「必须做」编号列表，别写成段落。

### 为什么文档走技能 `references/` 而不是 Agent 根目录的 `docs/`

Agent 根目录的 `docs/` 没有任何消费入口——市场详情不渲染它，Agent 自己也读不到。放进技能的 `references/` 后 Agent 按需加载，不占每轮 token，且与既有技能模式零新概念。

### 为什么新建 `dingtalk-guide` 而不是塞进既有两个技能

`dingtalk-onboarding` 触发于安装/报错，`dingtalk-workflows` 触发于跨产品编排，都接不住「钉盘同步怎么用」这类单产品提问。

### 为什么 `USAGE.md` 只放「装之前该知道的」

它是市场详情页渲染的内容，有 16000 字符上限（超出截断到行边界）。长文档的出路是 `references/`，不是把全部塞进市场详情。

## 7. 测试方法

### 7.1 用什么表面

本机资源不足以稳定跑 Electron dev 实例（16GB 会被 OOM）。用 **standalone agent-service**：

```bash
npm run build   # 约 30s，产出 out/main/agent-service.js
DESIRECORE_PROTECTED_STATE_DIR=<隔离目录> DESIRECORE_PROTECTED_STATE_KEY=<串> \
DESIRECORE_HOME=<隔离 home> AGENT_SERVICE_TOKEN=<串> \
AGENT_SERVICE_MODE=external AGENT_SERVICE_PORT=18787 \
node out/main/agent-service.js
```

**这不是绕过真实链路**——GUI 用的也是 Socket.IO 的 `query:start`，standalone 只是同一条链路换了个客户端。

**token 不要放 `/tmp`**，macOS 会清掉，服务启动失败报「standalone 部署必须设置 AGENT_SERVICE_TOKEN」。放 `~/` 下。

### 7.2 证据从哪取

驱动对话用 Socket.IO `query:start`（载荷 `{prompt, clientRequestId, options:{agentId, conversationId}}`，顶层 `additionalProperties: false`）。证据来源：

| 证据 | 来源 |
| --- | --- |
| Agent 调了哪些工具 | `query:message` → `message.type === 'tool_invocation_audit'`，带 `risk_level` / `requires_confirmation` |
| 实际跑了什么 dws 命令 | `tool_progress` 的 `input.command` |
| 闸门拦了什么 | `query:message` → `message.type === 'exec:approval_request'`（**包在信封里，不是顶层事件**） |
| 最终回复 | `message.type === 'assistant'` 的 `content[].text` |

**两个必踩的坑**：

1. **审批闸门会无限等真人。** `allow-listed` 模式下非白名单命令弹审批卡片，无头客户端不响应 ⇒ 每个用例都挂到超时。必须监听审批请求并回 `exec:approval_response {requestId, decision:'approved', agentId}`——这同时把闸门拦截记录成了证据。
2. **超时必须显式 `query:abort`，不能只断开连接。** 否则 run 停在 idle 占着 `max_concurrent_sessions` 槽位，后续用例全部排队超时，表现为「批量全挂、单独跑就过」，极易误判成 Agent 缺陷。

### 7.3 统计口径

按命令前缀统计覆盖率会**系统性低估**：Agent 会用官方技能自带的 Python 脚本（如 `report_received_today.py`），命令行里没有字面 `dws`。

### 7.4 迭代 principles 的方法

- **只修单个用例不算修好**——加一条不同产品域、同类句式的泛化用例。曾出现泛化用例先过、原用例还挂的情况；只测原用例会得出错误结论
- **改之前先做隔离实验**——用例连续两轮修不好时，用 2×2（措辞 × 产品域）定位是词汇触发还是域触发，别瞎试
- **改完跑全量回归**——确认早期用例没被改坏

## 8. 踩过的坑（按重要性）

1. **schema 标的 `availability: unavailable` 不是终局。** 实测 `contact +list-roster-fields` 标 unavailable 却完全可用；`hrbrain +list-pools` / `live +list-my-lives` 的 unavailable 是上游响应被 dws 严格校验挡下，不是能力缺失。Agent 看到 unavailable 应先按只读试一次再分诊，已写进 principles 第 4 条。

2. **`Skill` 工具加载技能时不输出 `<skill-resources>` 清单。** `builtin-tools/skill.ts` 走 `loadSkillContent` 直接返回 SKILL.md 正文，绕过 `assembler.ts` 的 XML 格式化器。技能默认 `disable-model-invocation: true` 只能经 Skill 工具加载 ⇒ **光把文档丢进 `references/` 目录，Agent 一篇都看不见**。`dingtalk-guide/SKILL.md` 正文里那 13 行 `${SKILL_DIR}/references/...` 显式索引是必要的，删了 Agent 就找不到文档。

3. **默认审批模式 `ai-approve` 需要配好审批 chat 模型。** 没配的话所有命令被 fail-closed 拦掉，连 Agent 自己写 `PLAN.md` 都会被拦。文档里已指导用户换「白名单」并把 `dws` 加进去。

4. **每条 `dws` 命令都被判 high 风险。** 实测 18 用例 / 34 次 Bash 触发 41 次审批。不加白名单用户要不停点确认。

5. **官方文档内部有自相矛盾之处**，实测定论（官方各错一处）：视频会议**不走** misc（CLI 无入口）；个人身份**能**撤回消息（`chat +messages-recall` 存在）；在线表格**能**导出 xlsx（`dws sheet export` 存在）。遇到文档冲突一律以 `dws <cmd> --help` 与 `dws schema` 实际输出为准。

6. **消息搜索是单独权益**，未开通返回 `SearchRightsDenied`。这是权益问题不是权限配置问题，其余 chat 能力不受影响。

7. **`oa +list-pending` 可能返回 `missing_collection`**——这不是「没有待审批」也不是报错，是 dws 拒绝把未知响应结构当空结果。Agent 应说「无法确认」而非「你没有待审批」。

8. **同一份内容不要存多份副本。** 迁到 pointer 形态之前，一处文案错误要在四个副本目录里各改一遍。现在内容只在本仓库，市场只留卡片。

## 9. 已知边界与缺口

### 未验证（有条件但没测）

| 产品 | 工具数 | 说明 |
| --- | ---: | --- |
| `attendance` | 58 | 多数命令需 `--users` 参数，闭环成本较高 |
| `devapp` | 25 | 开发者向 |
| `hrbrain` | 22 | 11/22 工具标 unavailable，该域可靠性存疑 |
| `markdown` / `whiteboard` / `agoal` / `recruit` / `audit` / `live` / `devdoc` / `mcp` / `pat` | 合计 30 | 长尾低频 |

### 环境不具备（无法测）

| 产品 | 工具数 | 阻塞原因 |
| --- | ---: | --- |
| `sheet` | 99 | 全部 99 个工具都必须带 `--node`（表格 ID），而测试组织内没有任何在线表格。解锁需准备一个表格 |
| `aitable` 写路径 | — | 组织内 0 个 Base |

### 依赖平台侧

- `event` 长连接的**实际事件投递**依赖 DesireCore 的长驻输出流事件接收器（ADR-138 / PR #2488，已合并但尚无工具入口）。Agent 侧只验证了「正确识别为未来事件并选对命令，不用轮询模拟」。

### 已实测的能力覆盖

真机触达 15 个产品 / 1011 工具（80%），18 个用例全部通过，33 条真实钉钉操作，**编造成功 0 次**。逐用例证据在 DesireCore 主仓库的 `.desirecore/plans/dingtalk-agent/`（gitignored，不进版本库）。

## 10. 相关记录

| 类型 | 位置 |
| --- | --- |
| 市场卡片 | `desirecore/market` → `agents/dingtalk-workspace/` |
| 钉钉官方 CLI 市场入口 | `desirecore/market` → `skills/dingtalk-cli/`（listing-only，官方仓库不公开） |
| `USAGE.md` 约定与市场详情区块 | DesireCore PR #2600，ADR-143 |
| 内联市场 Agent 的 Schema 冲突修复 | DesireCore PR #2515 |
| Skill 外部依赖声明与检测 | DesireCore PR #2485（`requires.bins`；注入只覆盖已加载技能，见 issue #2514） |
| 长驻输出流事件接收器 | DesireCore PR #2488，ADR-138 |
| 本 Agent 迁到 pointer 形态 | market PR #141 |
| 首次上架 | market PR #104（CLI 入口）/ #107（Agent 条目）/ #113（文档接入 references） |

## 11. 改动前的自检清单

- [ ] 改的是本仓库还是市场卡片？（内容 → 本仓库；发版 → 卡片）
- [ ] 公开信息边界扫描零命中
- [ ] `USAGE.md` 没有图片
- [ ] 平台功能说明逐字引用 i18n 文案，没有转述
- [ ] `agent.json` 没混入市场展示字段，`llm` 没钉死本机 Provider
- [ ] 新增的 references 文档已加进 `dingtalk-guide/SKILL.md` 的显式索引
- [ ] 改了 principles 的重要纪律放在「必须做」编号列表，不是说明段
- [ ] 改了 principles 跑过全量回归（18 用例）
- [ ] 若要发版：先推本仓库确认可达，再改市场三处 ref，跑校验看 `agents` 计数没掉
- [ ] `AGENTS.md` 与 `CLAUDE.md` 内容一致
