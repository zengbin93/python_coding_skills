---
name: feishu-task-execute
description: "通过飞书任务驱动开发工作流。读取飞书任务（URL 或 task_id），根据任务描述执行开发或分析工作，并通过飞书 IM 向任务负责人发送进度消息，在任务评论区添加重要细节评论。触发场景：用户提供飞书任务链接或 task_id 要求执行任务；用户要求根据飞书待办完成开发工作；用户提供以 applink.feishu.cn/client/todo 开头的链接。"
---

# Feishu Task Execute

通过飞书任务驱动开发：读取任务 → 理解需求 → 执行工作 → 汇报进度。

**前置依赖**: 必须先读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md) 了解认证和权限规则。

**命令参考**: 所有命令的完整签名、参数说明和正确用法见 [`reference/lark-cli-commands.md`](reference/lark-cli-commands.md)。执行命令前务必查阅，避免参数错误。

## 核心原则

> **推送进度不是可选操作，是必须操作。负责人不应该需要主动来问"做到哪了"。**
>
> 进度卡片（🚀📋✅⚠️❌）按流程主动推送，**无需逐一询问用户确认**。仅以下场景需要先确认：
> - 评论内容包含敏感信息或未证实的结论
> - 标记任务完成（`+complete`）前
> - 向非任务负责人发送消息

## 工作流程

### Step 1: 解析任务标识

从用户输入中提取飞书任务标识，支持三种格式：

| 输入格式 | 提取方式 | 示例 |
|---------|---------|------|
| 飞书任务 URL | 提取 `guid` 参数 | `applink.feishu.cn/client/todo/detail?guid=xxx` |
| task_id | 直接使用 | `t101041` |
| GUID | 直接使用 | `1266ec4d-4e25-44a1-892d-07520be060d5` |

URL 解析正则: `guid=([a-f0-9-]+)` 提取 guid。

### Step 2: 读取任务详情

```bash
# 通过 GUID 获取任务详情
lark-cli task tasks get --params '{"task_guid":"<guid>"}'
```

从返回结果中提取关键字段：

- `summary` — 任务标题
- `description` — 任务描述（核心工作内容）
- `members` — 任务成员（assignee 为负责人，open_id 格式）
- `url` — 任务链接
- `task_id` — 任务 ID
- `status` — 任务状态（todo/in-progress/done）

### Step 3: 深入理解任务并规划

仅靠 `description` 往往不足以完整理解任务。必须并行收集以下上下文：

#### 3.1 读取子任务

```bash
# 获取子任务列表
lark-cli task subtasks list --params '{"task_guid":"<guid>"}' --page-all
```

提取每个子任务的 `summary`、`description`、`status`，了解任务的拆分结构和完成进度。

#### 3.2 读取任务评论

```bash
# 尝试通过原生 API 读取评论（lark-cli 暂无评论列表快捷命令）
lark-cli schema task.tasks 2>&1 | grep -i comment
```

如果评论 API 可用，读取历史评论了解讨论上下文和需求变更。评论中常包含任务负责人的补充说明、优先级调整、技术方案讨论等关键信息。

> 若 lark-cli 暂不支持评论列表，跳过此步，在进度消息中注明。

#### 3.3 读取关联的飞书文档

从 `description`、子任务描述中提取飞书文档链接，常见格式：

| 链接格式 | 文档类型 | 读取方式 |
|---------|---------|---------|
| `feishu.cn/docx/<doc_id>` | 云文档 | 使用 `lark-doc` skill 读取 |
| `feishu.cn/wiki/<page_id>` | 知识库页面 | 使用 `lark-wiki` skill 读取 |
| `feishu.cn/sheets/<sheet_id>` | 电子表格 | 使用 `lark-sheets` skill 读取 |

提取链接正则：
```
https?://[a-zA-Z0-9-]+\.feishu\.cn/(docx|wiki|sheets|doc)/[a-zA-Z0-9]+
```

对每个文档链接，使用对应的 lark skill 读取内容，提取需求细节、技术规格、设计稿等关键信息。

#### 3.4 综合分析并规划

汇总任务描述 + 子任务 + 评论 + 关联文档的完整信息，制定执行计划。在开始执行前向用户确认执行方案。

### Step 4: 发送开始执行卡片 🚀

规划确认后、动手执行前，**必须发送 turquoise 开始执行卡片**：

```bash
lark-cli im +messages-send \
  --user-id "<assignee_open_id>" \
  --msg-type interactive \
  --content '<开始执行卡片JSON>' \
  --as bot
```

卡片内容必须包含：
- 任务标题和链接
- **详细执行计划**（不要笼统的"分析代码"，要具体到"阅读 xxx.py 的 verbose 传递链路"）
- 预计完成时间

卡片模板详见 [`reference/card-templates.md`](reference/card-templates.md) 的「模板 2：开始执行通知」。

> 注意：`im +messages-send` 仅支持 bot 身份。需确保 bot 与目标用户已有单聊关系。

### Step 5: 执行工作并实时推送进度

按确认的方案执行工作。**每完成一个重要阶段，必须发送蓝色进度卡片**。

#### 5.1 执行过程中的 Feishu 动作

| 执行节点 | Feishu 动作 | 卡片 | 必须包含的内容 |
|---------|------------|------|--------------|
| 分析完代码/理解完需求 | 发送卡片 | 📋 blue | 分析发现、确认的方案、下一步 |
| 完成代码变更 | 发送卡片 | 📋 blue | 变更的文件和内容摘要、验证结果 |
| 遇到阻塞/需要决策 | **立即**发送卡片 | ⚠️ orange | 当前进度、阻塞原因、具体问题 |
| 执行出错 | **立即**发送卡片 | ❌ red | 出错前进度、错误详情、影响范围 |

#### 5.2 执行摘要要求

每张卡片的进度/摘要字段**必须具体**，避免笼统描述：

| 笼统（禁止） | 具体（要求） |
|-------------|------------|
| "分析了代码" | "分析了 ast_factor_builder.py 中 verbose 参数的 5 处使用位置，确认第 333 行的 display=False 需改为 self.verbose" |
| "完成了修改" | "修改了 vista/agents/ast_factor_builder.py 第 333 行，将 display=False 改为 display=self.verbose，ruff lint 检查通过" |
| "验证通过" | "ruff check 通过，basedpyright 类型检查通过，verbose=True 时 ClaudeAgent.chat() 将展示 Rich 输出" |

### Step 6: 任务完成收尾（强制门控）

> **⚠️ 在向终端用户报告"任务完成"之前，必须完成以下 6.1 ~ 6.4 所有事项。缺少任何一项，不能认为任务执行完毕。**

#### 6.1 发送完成卡片 ✅

发送绿色完成卡片：

```bash
lark-cli im +messages-send \
  --user-id "<assignee_open_id>" \
  --msg-type interactive \
  --content '<完成卡片JSON>' \
  --as bot
```

卡片内容必须包含（对应模板 3 占位符）：

| 占位符 | 要求 |
|--------|------|
| `{completion_summary}` | 完成情况概述：做了什么、为什么这样做 |
| `{changed_files}` | 变更的文件列表（精确到行号） |
| `{execution_summary}` | 整个执行过程的摘要回顾，包含：1. 做了什么（代码变更概述）2. 为什么这样做（设计决策）3. 验证结果（lint/测试/试跑）4. 后续建议（可选） |

#### 6.2 添加任务评论

在任务评论区添加技术评论：

```bash
lark-cli task +comment --task-id "<task_id>" --content "<纯文本评论内容>" --as bot
```

评论必须包含：
- 关键技术决策和原因
- 重要代码变更说明（哪些文件、改了什么）
- 验证结果（lint/测试是否通过）

> 注意：飞书评论不支持 Markdown 和卡片，使用纯文本格式。

#### 6.3 完成检查清单

向用户报告"任务完成"之前，**逐项确认**：

- [ ] 已发送 🚀 开始执行卡片
- [ ] 已发送至少一张 📋 蓝色进度卡片（重要阶段完成时）
- [ ] 已发送 ✅ 绿色完成卡片（包含 changed_files 和 execution_summary）
- [ ] 已在任务评论区添加技术评论
- [ ] 终端已向用户展示完整的执行结果

**以上全部 ✅ 后，方可向用户报告任务执行完毕。**

#### 6.4 标记任务完成（可选）

如果用户确认，可标记任务完成：

```bash
lark-cli task +complete --task-id "<task_id>" --as bot
```

## 遇到需要用户审核/回答时的处理

当执行过程中遇到需要用户确认、审核或回答的问题时，**必须立即发送橙色阻塞卡片通知任务负责人**，避免任务因等待回复而卡住。

触发条件包括但不限于：
- 任务描述不够清晰，需要补充需求细节
- 存在多个技术方案，需要负责人选择
- 发现潜在风险或依赖问题，需要确认处理方式
- 代码变更范围超出预期，需要确认是否继续
- 需要访问特定资源（服务器、数据库等）但缺少授权信息
- 权限不足，需要负责人协助申请权限或将 bot 加为任务成员

> 发送阻塞卡片后，在 Claude Code 终端中也会同时等待用户回复。两种渠道的回复均可推动任务继续。

## 进度同步时机总览

| 时机 | 操作 | 卡片模板 | 必须包含的内容 |
|------|------|---------|--------------|
| 规划确认后、执行前 | 发送卡片消息 | 🚀 turquoise | 详细执行计划、预计时间 |
| 重要阶段完成 | 发送卡片 | 📋 blue | 具体的进展摘要、下一步计划 |
| 遇到阻塞/需要确认 | **立即**发送卡片消息 | ⚠️ orange | 当前进度、阻塞原因、具体问题 |
| 执行出错 | **立即**发送卡片消息 | ❌ red | 出错前进度、错误详情、影响范围 |
| 任务完成 | 发送卡片 + 添加评论 | ✅ green | 完成说明、变更文件、执行摘要 |

## 注意事项

- **必须积极主动推送进度**，每次推送附带最近执行摘要，让负责人无需查看终端即可了解任务动态
- 进度消息使用**飞书卡片**（`--msg-type interactive`），卡片模板详见 [`reference/card-templates.md`](reference/card-templates.md)
- **进度卡片按流程主动推送，无需逐一询问用户确认**。仅在评论含敏感信息或标记完成前需确认
- 飞书任务评论**不支持 Markdown 和卡片**，使用纯文本格式
- 遇到需要用户确认/审核的问题时，**必须立即发送橙色阻塞卡片**通知负责人，不能仅依赖终端等待
- 如果任务描述不清晰，先发送阻塞卡片并通过 IM 向负责人确认需求
- 任务完成后可考虑使用 `lark-cli task +complete --task-id "<task_id>"` 标记完成（需用户确认）

## 常见错误与权限处理

### 命令参数错误

**错误示例**：`unknown flag: --task-guid`

`+comment`、`+complete`、`+reopen`、`+update`、`+assign` 等操作命令只接受 `--task-id`（任务 ID，如 `t101043`），不支持 `--task-guid`。

- 需要用 GUID 查询任务 → 用 `task tasks get --params '{"task_guid":"<guid>"}'`，从返回结果中获取 `task_id`
- 需要操作任务（评论、完成等）→ 用 `--task-id "<task_id>"`

详细对照见 [`reference/lark-cli-commands.md`](reference/lark-cli-commands.md) 第 5 节。

### 权限不足（Permission denied）

**典型错误**：

```json
{
  "error": {
    "type": "permission_error",
    "code": 1470403,
    "message": "Permission denied (Details: Invoker is unauthorized to create a comment for the task with guid '...'.)"
  }
}
```

**排查步骤**：

1. **尝试切换身份**：bot 权限不足时，改用 `--as user`（需要用户已通过 `lark-cli auth login` 登录）
   ```bash
   # 用 user 身份替代 bot
   lark-cli task +comment --task-id "<task_id>" --content "评论" --as user
   ```

2. **检查应用权限**：在[飞书开放平台后台](https://open.feishu.cn/app) → 应用 → 权限管理，确认已开通：
   - `task:task:read` — 读取任务
   - `task:task:write` — 编辑任务（含评论、完成）
   - `im:message:send` — 发送消息

3. **申请权限**：如果权限未开通：
   - 在应用后台「权限管理」中搜索并申请对应权限
   - 需要管理员审批通过后才生效
   - 若无管理员权限，将所需权限列表发送给任务负责人协助申请

4. **检查任务可见性**：bot 应用需要是任务的成员（创建者、负责人或关注者）才能操作

**遇到权限错误时的标准处理流程**：

1. 先尝试 `--as user` 切换身份重试
2. 如果仍失败，发送橙色阻塞卡片通知任务负责人，说明：
   - 缺少的具体权限名称和 code
   - 申请权限的操作步骤
   - 或请负责人将 bot 应用添加为任务关注者
3. 在终端中同时等待用户指示，提供备选方案（如手动在飞书中操作）
