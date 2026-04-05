---
name: feishu-task-execute
description: "通过飞书任务驱动开发工作流。读取飞书任务（URL 或 task_id），根据任务描述执行开发或分析工作，并通过飞书 IM 向任务负责人发送进度消息，在任务评论区添加重要细节评论。触发场景：用户提供飞书任务链接或 task_id 要求执行任务；用户要求根据飞书待办完成开发工作；用户提供以 applink.feishu.cn/client/todo 开头的链接。"
---

# Feishu Task Execute

通过飞书任务驱动开发：读取任务 → 理解需求 → 执行工作 → 汇报进度。

**前置依赖**: 必须先读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md) 了解认证和权限规则。

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

### Step 4: 执行开发/分析工作

按确认的方案执行工作。在以下关键节点通过飞书同步进度：
- 开始执行时
- 完成重要阶段时
- 遇到阻塞或需要决策时
- 任务完成时

### Step 5: 发送进度消息

向任务负责人发送飞书消息。使用 `--user-id` 直接发送私聊消息：

```bash
# 发送进度消息给任务负责人
lark-cli im +messages-send --user-id "<assignee_open_id>" --text $'<进度消息内容>'
```

**消息格式**（纯文本，飞书评论不支持 Markdown）：

```
【飞书任务进度通知】

任务：{summary}
链接：{url}

当前状态：{状态描述}
进度说明：{已完成的进展}
后续待执行：{接下来的计划}
```

> 注意：lark-cli im +messages-send 仅支持 bot 身份。需确保 bot 与目标用户已有单聊关系。

### Step 6: 添加任务评论

对开发过程中的重要细节，在任务评论区添加评论：

```bash
# 添加评论（注意：飞书评论不支持 Markdown）
lark-cli task +comment --task-id "<task_id>" --content "<纯文本评论内容>"
```

评论适用场景：
- 关键技术决策和原因
- 遇到的问题及解决方案
- 重要代码变更说明
- 测试结果摘要

## 进度同步时机

| 时机 | 操作 | 内容 |
|------|------|------|
| 开始执行 | 发送 IM 消息 | 确认收到任务，说明执行计划 |
| 重要阶段完成 | 发送 IM + 添加评论 | 进展摘要，重要细节 |
| 遇到阻塞 | 发送 IM 消息 | 问题描述，需要用户决策 |
| 任务完成 | 发送 IM + 添加评论 | 最终成果，完成的变更列表 |

## 注意事项

- 飞书评论**不支持 Markdown**，使用纯文本格式
- IM 消息使用 `--text` 发送纯文本，避免格式丢失
- 向用户确认后再发送消息和评论，避免意外打扰
- 如果任务描述不清晰，先通过 IM 向负责人确认需求
- 任务完成后可考虑使用 `lark-cli task +complete --task-id "<task_id>"` 标记完成（需用户确认）
