# lark-cli 任务场景常用命令参考

本文档整理 feishu-task-execute 工作流中涉及的所有 lark-cli 命令，包含完整签名、参数说明和正确用法示例。

> 最后更新：2026-04-05

---

## 1. 任务读取

### 1.1 获取任务详情

```bash
lark-cli task tasks get --params '{"task_guid":"<guid>"}'
```

| 参数 | 说明 |
|------|------|
| `--params` | URL 查询参数 JSON，必须包含 `task_guid` |
| `--as` | 身份类型：`user` / `bot` / `auto`（默认 auto） |
| `--format` | 输出格式：`json`（默认）/ `ndjson` / `table` / `csv` |

**返回关键字段**：

- `task.summary` — 标题
- `task.description` — 描述
- `task.members` — 成员列表（`role: assignee` 为负责人）
- `task.task_id` — 任务 ID
- `task.guid` — 任务 GUID
- `task.status` — 状态（`todo` / `in-progress` / `done`）
- `task.url` — 任务链接

### 1.2 获取子任务列表

```bash
lark-cli task subtasks list --params '{"task_guid":"<guid>"}' --page-all
```

| 参数 | 说明 |
|------|------|
| `--params` | 必须包含 `task_guid` |
| `--page-all` | 自动翻页获取全部 |

### 1.3 获取我的任务列表

```bash
lark-cli task +get-my-tasks
```

---

## 2. 任务操作

### 2.1 添加评论

```bash
# 推荐：user 身份（权限更可靠）
lark-cli task +comment --task-id "<guid>" --content "<评论内容>" --as user

# 备选：bot 身份（需要 task:task:write 权限且被任务授权）
lark-cli task +comment --task-id "<guid>" --content "<评论内容>" --as bot
```

| 参数 | 说明 |
|------|------|
| `--task-id` | **任务 GUID**（如 `9902c0c8-0722-41d2-9d80-f93011fd464a`），从 `task tasks get` 返回的 `task.guid` 字段获取 |
| `--content` | 评论内容（纯文本，不支持 Markdown） |
| `--as` | 身份：`user`（推荐）或 `bot` |

> **重要**：`--task-id` 参数应传入 GUID 而非短 ID（如 `t101046`）。短 ID 会导致 `Resource not found` 错误。
>
> **权限说明**：bot 身份通常缺少评论权限（`Permission denied`），建议优先使用 `--as user`。若 user 身份也不可用，发送橙色阻塞卡片通知任务负责人。

### 2.2 标记任务完成

```bash
lark-cli task +complete --task-id "<task_id>" --as bot
```

| 参数 | 说明 |
|------|------|
| `--task-id` | 任务 ID |
| `--as` | 身份：`user`（默认）或 `bot` |

### 2.3 重新打开任务

```bash
lark-cli task +reopen --task-id "<task_id>" --as bot
```

### 2.4 创建任务

```bash
lark-cli task +create \
  --summary "任务标题" \
  --description "任务描述" \
  --assignee "ou_xxx" \
  --due "2026-04-10" \
  --as bot
```

| 参数 | 说明 |
|------|------|
| `--summary` | 任务标题 |
| `--description` | 任务描述 |
| `--assignee` | 负责人 open_id |
| `--due` | 截止日期（ISO 8601 / `date:YYYY-MM-DD` / `relative:+2d` / 毫秒时间戳） |
| `--tasklist-id` | 可选，所属清单 ID |
| `--data` | 可选，完整 JSON payload |

### 2.5 更新任务属性

```bash
lark-cli task +update --task-id "<task_id>" --summary "新标题" --as bot
```

| 参数 | 说明 |
|------|------|
| `--task-id` | 任务 ID（逗号分隔支持多个） |
| `--summary` | 新标题 |
| `--description` | 新描述 |
| `--due` | 新截止日期 |

### 2.6 分配/移除成员

```bash
# 添加负责人
lark-cli task +assign --task-id "<task_id>" --add "ou_xxx,ou_yyy" --as bot

# 移除负责人
lark-cli task +assign --task-id "<task_id>" --remove "ou_xxx" --as bot
```

---

## 3. 消息发送

### 3.1 发送纯文本消息

```bash
lark-cli im +messages-send --user-id "<open_id>" --text "消息内容"
```

### 3.2 发送 Markdown 消息

```bash
lark-cli im +messages-send --user-id "<open_id>" --markdown "**加粗** 普通文本"
```

> 自动包装为 post 格式，图片 URL 自动解析。

### 3.3 发送卡片消息

```bash
lark-cli im +messages-send \
  --user-id "<open_id>" \
  --msg-type interactive \
  --content '<卡片JSON字符串>'
```

| 参数 | 说明 |
|------|------|
| `--user-id` | 接收人 open_id（`ou_xxx`），与 `--chat-id` 互斥 |
| `--chat-id` | 群聊 ID（`oc_xxx`），与 `--user-id` 互斥 |
| `--msg-type` | 消息类型：`text` / `post` / `image` / `file` / `audio` / `media` / `interactive` / `share_chat` / `share_user` |
| `--content` | 消息内容 JSON（与 `--text` / `--markdown` 等互斥） |
| `--text` | 纯文本（自动推断 msg_type 为 text） |
| `--markdown` | Markdown 文本（自动推断 msg_type 为 post） |
| `--image` | 图片（file_key 或本地路径） |
| `--file` | 文件（file_key 或本地路径） |
| `--idempotency-key` | 幂等键，防止重复发送 |

**卡片 JSON v2 结构示例**：

```json
{
  "config": { "wide_screen_mode": true },
  "header": {
    "title": { "tag": "plain_text", "content": "卡片标题" },
    "template": "blue"
  },
  "elements": [
    {
      "tag": "div",
      "text": { "tag": "lark_md", "content": "**内容**" }
    },
    { "tag": "hr" },
    {
      "tag": "action",
      "actions": [
        {
          "tag": "button",
          "text": { "tag": "plain_text", "content": "按钮" },
          "type": "primary",
          "url": "https://example.com"
        }
      ]
    }
  ]
}
```

**卡片主题色**：`blue` / `turquoise` / `green` / `orange` / `red` / `violet` / `indigo` / `wathet` / `yellow` / `grey` / `carmine`

> **注意**：`im +messages-send` 仅支持 bot 身份，`--as` 参数无效。需确保 bot 与目标用户已有单聊关系。

---

## 4. 常见错误速查

| 错误现象 | 原因 | 解决方案 |
|---------|------|---------|
| `unknown flag: --task-guid` | `+comment` / `+complete` 等命令不支持 `--task-guid` | 使用 `--task-id`（任务 ID，如 `t101043`）；读取任务详情用 `tasks get --params` |
| `Resource not found (code: 1470404)` | `+comment` 传入短 ID（如 `t101046`）而非 GUID | 改用完整 GUID：`--task-id "9902c0c8-..."` |
| `Permission denied (code: 1470403)` | 当前身份无权限操作该任务 | 改用 `--as user`；见下方权限处理指引 |
| `unknown flag: --content-file` | 不支持文件参数 | 直接使用 `--content` 传入字符串 |
| bot 发消息失败 | bot 与用户无单聊 | 用户需先向 bot 发送过任意消息建立会话 |

---

## 5. 参数传递方式对照

不同命令使用不同的标识方式：

| 命令 | 标识参数 | 说明 |
|------|---------|------|
| `task tasks get` | `--params '{"task_guid":"<guid>"}'` | 读取类命令用 `--params` 传 GUID |
| `task subtasks list` | `--params '{"task_guid":"<guid>"}'` | 同上 |
| `task +comment` | `--task-id "<guid>"` | 操作类命令，传 GUID（非短 ID） |
| `task +complete` | `--task-id "<task_id>"` | 同上 |
| `task +reopen` | `--task-id "<task_id>"` | 同上 |
| `task +update` | `--task-id "<task_id>"` | 同上 |
| `task +assign` | `--task-id "<task_id>"` | 同上 |
| `task +create` | 无需标识 | 创建新任务 |
| `im +messages-send` | `--user-id "<open_id>"` | 按人发送 |
| `im +messages-send` | `--chat-id "<chat_id>"` | 按群发送 |
