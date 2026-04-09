---
name: feishu-task-execute
description: "通过飞书任务驱动开发工作流。读取飞书任务（URL 或 task_id），根据任务描述执行开发或分析工作，并通过飞书 IM 向任务负责人发送进度消息，在任务评论区添加重要细节评论。触发场景：用户提供飞书任务链接或 task_id 要求执行任务；用户要求根据飞书待办完成开发工作；用户提供以 applink.feishu.cn/client/todo 开头的链接。"
---

# Feishu Task Execute

通过飞书任务驱动开发：读取任务 → 理解需求 → 执行工作 → 汇报进度。

## 核心原则

1. **优先用 lark-* skill，不要直接拼 lark-cli**。通过 `Skill` 工具加载对应的官方 skill（`lark-task`、`lark-im`、`lark-doc`、`lark-whiteboard`、`lark-wiki`、`lark-sheets` 等），它会注入最新命令签名和错误处理流程。只有在所有 lark-* skill 都无法覆盖时，才退回直接调用 lark-cli 或 `lark-openapi-explorer` 查原生 API。Skill 选用指南见 [`reference/lark-skills-guide.md`](reference/lark-skills-guide.md)。
2. **进度推送是必须动作**，不是可选。负责人不应该主动来问"做到哪了"。按流程主动推送 🚀📋✅⚠️❌ 卡片，无需逐一征求用户确认。
3. **前置依赖**：首次使用须先过 `lark-shared` skill 完成认证/权限配置。

> ⚠️ 以下所有 Step 中凡是提到"读取任务"、"发送卡片"、"读文档"等动作，**首选路径都是加载对应 lark-* skill**。Step 说明中出现的少量 `lark-cli` 命令仅用于快速参考，具体参数和最新用法以 skill 输出为准。命令 fallback 详见 [`reference/lark-cli-commands.md`](reference/lark-cli-commands.md)。

---

## 工作流程

### Step 1 — 解析任务标识

从用户输入中提取飞书任务标识，三种格式：

| 输入 | 提取 |
|------|------|
| URL `applink.feishu.cn/client/todo/detail?guid=xxx` | 正则 `guid=([a-f0-9-]+)` 取 guid |
| `task_id`（如 `t101041`） | 直接使用 |
| GUID（如 `1266ec4d-...`） | 直接使用 |

### Step 2 — 读取任务详情

调用 **`lark-task`** skill 按 GUID 拉取任务详情。重点提取：

- `summary` — 任务标题
- `description` — 任务描述（核心工作内容）
- `members.assignee` — 负责人 open_id
- `url` — 任务链接
- **`task_id` — 必须保存，后续 `+comment`/`+complete` 等操作命令必需**
- `status` — 任务状态

### Step 3 — 深入理解任务并规划

仅靠 `description` 通常不够，并行收集以下上下文：

1. **子任务**：通过 `lark-task` 获取子任务列表，理解拆分结构和完成进度
2. **历史评论**：通过 `lark-task` 读取评论，了解需求变更、补充说明、方案讨论。若 lark-task 暂不支持评论列表则跳过并在进度消息中注明
3. **关联文档**：从 `description`/子任务描述中提取飞书链接，按类型调用对应 skill：
   | 链接格式 | Skill |
   |---------|-------|
   | `feishu.cn/docx/<token>` | `lark-doc` |
   | `feishu.cn/wiki/<page_id>` | `lark-wiki` |
   | `feishu.cn/sheets/<token>` | `lark-sheets` |

   提取正则：`https?://[a-zA-Z0-9-]+\.feishu\.cn/(docx|wiki|sheets|doc)/[a-zA-Z0-9]+`

4. **综合规划**：汇总所有信息制定执行计划，开始执行前**向用户确认执行方案**

### Step 4 — 发送 🚀 开始执行卡片

规划确认后、动手执行前，通过 **`lark-im`** skill 发送 turquoise 开始执行卡片给负责人。

- 卡片必须包含：任务标题和链接、**详细执行计划**（具体到文件/行号/动作）、预计完成时间
- 模板见 [`reference/card-templates.md`](reference/card-templates.md) 的「🚀 开始执行」
- **`im +messages-send` 仅支持 bot 身份**，且 bot 与目标用户需已有单聊关系（见下方「权限处理」的例外说明）

### Step 5 — 执行工作并实时推送进度

按确认方案执行。每完成一个重要阶段，主动推送卡片：

| 执行节点 | 卡片 | 必须包含 |
|---------|------|---------|
| 分析完代码/需求 | 📋 blue | 分析发现、确认方案、下一步 |
| 完成代码变更 | 📋 blue | 变更文件/行号、内容摘要、验证结果 |
| 遇到阻塞/需要决策 | ⚠️ orange 立即 | 当前进度、阻塞原因、具体问题 |
| 执行出错 | ❌ red 立即 | 出错前进度、错误详情、影响范围 |

**摘要必须具体**，避免笼统：

| ❌ 笼统 | ✅ 具体 |
|--------|--------|
| "分析了代码" | "分析了 ast_factor_builder.py 中 verbose 参数的 5 处使用位置，确认第 333 行 display=False 需改为 self.verbose" |
| "完成了修改" | "修改了 vista/agents/ast_factor_builder.py 第 333 行，ruff/basedpyright 均通过" |

### Step 6 — 任务完成收尾（强制门控）

> **向用户报告"任务完成"之前，必须完成 6.1 ~ 6.3 全部事项。**

**6.1** 通过 `lark-im` 发送 ✅ green 完成卡片，内容包含：
- `{completion_summary}` 完成概述（做了什么、为什么）
- `{changed_files}` 变更文件列表（精确到行号）
- `{execution_summary}` 执行回顾（代码变更、设计决策、验证结果、后续建议）

**6.2** 通过 `lark-task` 在任务评论区添加纯文本技术评论（评论**不支持 Markdown 和卡片**），内容：
- 关键技术决策和原因
- 重要代码变更说明
- 验证结果

**6.3** 逐项确认检查清单：
- [ ] 已发送 🚀 开始执行卡片
- [ ] 已发送至少一张 📋 蓝色进度卡片
- [ ] 已发送 ✅ 绿色完成卡片（含 changed_files 和 execution_summary）
- [ ] 已在任务评论区添加技术评论
- [ ] 终端已向用户展示完整执行结果

**6.4（可选）** 经用户确认后通过 `lark-task` 标记任务完成（`+complete`）。

---

## 阻塞与用户确认

执行过程遇到需要用户确认/审核的问题时，**立即发送 ⚠️ orange 阻塞卡片**通知负责人，不要只在终端里等。

触发场景：
- 任务描述不清晰，需补充需求
- 多个技术方案需选择
- 发现潜在风险/依赖问题
- 变更范围超出预期
- 缺少资源访问授权
- 权限不足，需负责人协助申请 scope 或将 bot 加为任务成员

发送阻塞卡片后，终端也会同时等待用户回复，两种渠道任一回复都可推动任务继续。

仅以下场景需**先向用户确认**再操作：
- 评论内容含敏感信息或未证实结论
- 标记任务完成（`+complete`）前
- 向**非任务负责人**发送消息

---

## 附录 A — 常见错误处理

### A.1 卡片 JSON 结构错误

```
HTTP 400 ... unknown property, property: elements, code: 200621
```

**根因**：用了 `"schema": "2.0"` 但 `elements` 放在顶层。schema 2.0 要求 `elements` 必须嵌套在 `body` 对象内。

**修复**：把 `elements` 包进 `body`，严格复用 [`reference/card-templates.md`](reference/card-templates.md) 中的基础结构。

### A.2 命令参数错误

- `unknown flag: --task-guid` → `+comment`/`+complete` 等操作命令只接受 `--task-id`，不支持 `--task-guid`
- 用 GUID 查任务 → `tasks get --params '{"task_guid":"<guid>"}'`，从返回中提取 `task_id`
- 用 `task_id` 操作任务 → `--task-id "<task_id>"`

详见 [`reference/lark-cli-commands.md`](reference/lark-cli-commands.md) 第 4、5 节。

### A.3 权限不足（Permission denied, code 1470403）

**标准处理流程**：

1. **切换身份重试**：bot 权限不足时优先改用 `--as user`（需用户已 `lark-cli auth login`）
2. **IM 发送例外**：`im +messages-send` **仅支持 bot 身份**，`--as user` 无效。遇到 bot IM 权限不足时只有两条路：
   - 让负责人先向 bot 发送任意消息建立单聊
   - 在[飞书开放平台](https://open.feishu.cn/app)申请 `im:message:send` scope
3. **检查应用 scope**：任务类操作需 `task:task:read` + `task:task:write`
4. **检查可见性**：bot 需为任务的创建者/负责人/关注者
5. **仍失败** → 发送 ⚠️ orange 阻塞卡片给负责人，说明缺失的 scope/code 和申请步骤，同时在终端等待用户指示
6. **权限相关问题优先查 `lark-shared` skill**，它是所有 lark-* 的认证/权限共享基础

---

## 附录 B — 产出文档与绘图

### B.1 把任务产出写到飞书文档

**首选 `lark-doc` skill**。常见操作：

- **追加进度/结果**：`docs +update --mode append`（最常用，避免误删原内容）
- **按标题插入**：`docs +update --mode insert_after --selection-by-title "## 变更记录"`
- **全新报告**：`docs +create --title "..." --markdown "$(cat report.md)"`
- **先搜后读**：不确定文档位置时先 `docs +search`，再 `docs +fetch`
- **身份默认 user**：docs 系列 bot 大多无权，遇到权限错误首选 `--as user`

完整的 docs 命令签名和 mode 速查见 [`reference/lark-cli-commands.md`](reference/lark-cli-commands.md) 第 6 节。

### B.2 绘图统一走 lark-whiteboard

流程图、时序图、状态机、思维导图、架构图、关系图等**一律用 `lark-whiteboard` skill** 完成，**不要**在 Markdown 中手写 ASCII 图或 mermaid 代码块。

三步工作流：

1. 加载 `lark-whiteboard` skill 获取白板 DSL 规范
2. 用 `lark-doc` 在文档中插入占位：`docs +update --mode append --markdown '<whiteboard type="blank"></whiteboard>'`（需要多张图就重复占位）
3. 用 `docs +whiteboard-update` 把生成的 DSL 注入对应白板
