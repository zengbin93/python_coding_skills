# 飞书卡片模板参考

feishu-task-execute 工作流中使用的标准飞书卡片 JSON v2 模板。

> 最后更新：2026-04-05

---

## 卡片基本结构

所有卡片遵循相同的结构模式：

```
config → header → elements[任务信息区, hr分隔线, 详情区, 操作按钮区]
```

```json
{
  "config": { "wide_screen_mode": true },
  "header": {
    "title": { "tag": "plain_text", "content": "<标题>" },
    "template": "<主题色>"
  },
  "elements": [
    { "任务信息 div" },
    { "tag": "hr" },
    { "详情 div" },
    { "action 按钮区" }
  ]
}
```

---

## 主题色对照

| 场景 | header.template | header.title |
|------|----------------|--------------|
| 通用进度 | `blue` | `📋 任务进度通知` |
| 开始执行 | `turquoise` | `🚀 开始执行任务` |
| 重要阶段完成 | `blue` | `📋 任务进度通知` |
| 任务完成 | `green` | `✅ 任务已完成` |
| 需要确认/阻塞 | `orange` | `⚠️ 任务执行需要确认` |
| 执行出错 | `red` | `❌ 任务执行异常` |

可用主题色：`blue` / `turquoise` / `green` / `orange` / `red` / `violet` / `indigo` / `wathet` / `yellow` / `grey` / `carmine`

---

## 模板 1：通用进度通知（蓝色）

用于：开始执行、重要阶段完成等常规进度同步。

```json
{
  "config": { "wide_screen_mode": true },
  "header": {
    "title": { "tag": "plain_text", "content": "📋 任务进度通知" },
    "template": "blue"
  },
  "elements": [
    {
      "tag": "div",
      "text": {
        "tag": "lark_md",
        "content": "**任务：** {summary}\n**链接：** [{url}]({url})"
      }
    },
    { "tag": "hr" },
    {
      "tag": "div",
      "text": {
        "tag": "lark_md",
        "content": "**当前状态：** {status}\n**进度说明：** {progress}\n**后续计划：** {next_plan}"
      }
    },
    {
      "tag": "action",
      "actions": [
        {
          "tag": "button",
          "text": { "tag": "plain_text", "content": "查看任务" },
          "type": "primary",
          "url": "{task_url}"
        }
      ]
    }
  ]
}
```

**占位符说明**：

| 占位符 | 内容 | 示例 |
|-------|------|------|
| `{summary}` | 任务标题 | 优化 feishu-task-execute 技能 |
| `{url}` | 任务链接 | `https://applink.feishu.cn/client/todo/...` |
| `{status}` | 当前执行状态 | 正在编写参考文档 |
| `{progress}` | 已完成的进展摘要 | 已创建 lark-cli-commands.md，正在更新 SKILL.md |
| `{next_plan}` | 接下来的计划 | 添加权限处理章节，验证命令签名 |
| `{task_url}` | 同 `{url}`，用于按钮跳转 | |

---

## 模板 2：开始执行通知（青色）

用于：确认收到任务，说明执行计划。

```json
{
  "config": { "wide_screen_mode": true },
  "header": {
    "title": { "tag": "plain_text", "content": "🚀 开始执行任务" },
    "template": "turquoise"
  },
  "elements": [
    {
      "tag": "div",
      "text": {
        "tag": "lark_md",
        "content": "**任务：** {summary}\n**链接：** [{url}]({url})"
      }
    },
    { "tag": "hr" },
    {
      "tag": "div",
      "text": {
        "tag": "lark_md",
        "content": "**当前状态：** 开始执行\n**执行计划：**\n{execution_plan}\n\n预计完成时间：{eta}"
      }
    },
    {
      "tag": "action",
      "actions": [
        {
          "tag": "button",
          "text": { "tag": "plain_text", "content": "查看任务" },
          "type": "primary",
          "url": "{task_url}"
        }
      ]
    }
  ]
}
```

**占位符说明**：

| 占位符 | 内容 |
|-------|------|
| `{execution_plan}` | 执行计划要点，用换行编号列出 |
| `{eta}` | 预计完成时间（如"约 10 分钟"） |

---

## 模板 3：任务完成通知（绿色）

用于：任务执行完毕。

```json
{
  "config": { "wide_screen_mode": true },
  "header": {
    "title": { "tag": "plain_text", "content": "✅ 任务已完成" },
    "template": "green"
  },
  "elements": [
    {
      "tag": "div",
      "text": {
        "tag": "lark_md",
        "content": "**任务：** {summary}\n**链接：** [{url}]({url})"
      }
    },
    { "tag": "hr" },
    {
      "tag": "div",
      "text": {
        "tag": "lark_md",
        "content": "**完成说明：** {completion_summary}\n**变更文件：** {changed_files}\n**执行摘要：** {execution_summary}"
      }
    },
    {
      "tag": "action",
      "actions": [
        {
          "tag": "button",
          "text": { "tag": "plain_text", "content": "查看任务" },
          "type": "primary",
          "url": "{task_url}"
        }
      ]
    }
  ]
}
```

**占位符说明**：

| 占位符 | 内容 |
|-------|------|
| `{completion_summary}` | 完成情况概述 |
| `{changed_files}` | 变更的文件列表 |
| `{execution_summary}` | 整个执行过程的摘要回顾 |

---

## 模板 4：阻塞确认通知（橙色）

用于：遇到需要用户确认、审核或回答的问题时，**必须立即发送**。

```json
{
  "config": { "wide_screen_mode": true },
  "header": {
    "title": { "tag": "plain_text", "content": "⚠️ 任务执行需要确认" },
    "template": "orange"
  },
  "elements": [
    {
      "tag": "div",
      "text": {
        "tag": "lark_md",
        "content": "**任务：** {summary}\n**链接：** [{url}]({url})"
      }
    },
    { "tag": "hr" },
    {
      "tag": "div",
      "text": {
        "tag": "lark_md",
        "content": "**当前进度：** {current_progress}\n**阻塞原因：** {reason}\n\n**需要确认的问题：**\n{questions}\n\n请在 Claude Code 终端中回复，或通过飞书任务评论补充说明。"
      }
    },
    {
      "tag": "action",
      "actions": [
        {
          "tag": "button",
          "text": { "tag": "plain_text", "content": "查看任务" },
          "type": "primary",
          "url": "{task_url}"
        }
      ]
    }
  ]
}
```

**占位符说明**：

| 占位符 | 内容 |
|-------|------|
| `{current_progress}` | 阻塞前已完成的进度摘要 |
| `{reason}` | 阻塞原因 |
| `{questions}` | 需要确认的具体问题列表 |

---

## 模板 5：执行出错通知（红色）

用于：执行过程中遇到无法继续的错误。

```json
{
  "config": { "wide_screen_mode": true },
  "header": {
    "title": { "tag": "plain_text", "content": "❌ 任务执行异常" },
    "template": "red"
  },
  "elements": [
    {
      "tag": "div",
      "text": {
        "tag": "lark_md",
        "content": "**任务：** {summary}\n**链接：** [{url}]({url})"
      }
    },
    { "tag": "hr" },
    {
      "tag": "div",
      "text": {
        "tag": "lark_md",
        "content": "**当前进度：** {current_progress}\n**错误详情：** {error_detail}\n**影响范围：** {impact}\n\n请检查后告知是否继续。"
      }
    },
    {
      "tag": "action",
      "actions": [
        {
          "tag": "button",
          "text": { "tag": "plain_text", "content": "查看任务" },
          "type": "primary",
          "url": "{task_url}"
        }
      ]
    }
  ]
}
```

**占位符说明**：

| 占位符 | 内容 |
|-------|------|
| `{current_progress}` | 出错前已完成的进度摘要 |
| `{error_detail}` | 错误的具体描述 |
| `{impact}` | 错误的影响范围 |

---

## 卡片 elements 常用组件参考

### 文本块（div）

```json
{ "tag": "div", "text": { "tag": "lark_md", "content": "**加粗** 普通 [链接](url)" } }
```

- `tag: "lark_md"` 支持飞书 Markdown 子集（加粗、链接、换行等）
- `tag: "plain_text"` 纯文本

### 分隔线（hr）

```json
{ "tag": "hr" }
```

### 按钮组（action）

```json
{
  "tag": "action",
  "actions": [
    {
      "tag": "button",
      "text": { "tag": "plain_text", "content": "按钮文字" },
      "type": "primary",
      "url": "https://example.com"
    }
  ]
}
```

- `type` 可选：`primary` / `default` / `danger`
- 支持 `url`（跳转链接）或 `value`（回调数据）
