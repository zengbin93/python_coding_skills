# 飞书卡片模板参考（schema 2.0）

feishu-task-execute 工作流统一使用飞书卡片 **schema 2.0**。

> 最后更新：2026-04-07

---

## 与 schema 1.0 的关键差异

- 必须包含 `"schema": "2.0"`
- `elements` **必须嵌套在 `body` 对象内**，不能放在顶层

**典型错误**（顶层 elements，会返回 `HTTP 400 ... unknown property: elements, code: 200621`）：

```json
{"schema": "2.0", "config": {...}, "header": {...}, "elements": [...]}
```

**正确**：

```json
{"schema": "2.0", "config": {...}, "header": {...}, "body": {"elements": [...]}}
```

---

## 通用基础结构

所有 5 种进度卡片共用同一骨架，**只有 `header.template` / `header.title.content` / 详情区 content 三处不同**：

```json
{
  "schema": "2.0",
  "config": { "wide_screen_mode": true },
  "header": {
    "title": { "tag": "plain_text", "content": "<TITLE>" },
    "template": "<TEMPLATE>"
  },
  "body": {
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
          "content": "<DETAIL>"
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
}
```

---

## 5 种卡片差异表

把基础结构里的 `<TITLE>` / `<TEMPLATE>` / `<DETAIL>` 替换成下表对应内容即可。

| 场景 | TEMPLATE | TITLE | DETAIL（详情区 lark_md content） |
|------|----------|-------|-------------------------------|
| 🚀 开始执行 | `turquoise` | `🚀 开始执行任务` | `**当前状态：** 开始执行\n**执行计划：**\n{execution_plan}\n\n预计完成时间：{eta}` |
| 📋 进度通知 | `blue` | `📋 任务进度通知` | `**当前状态：** {status}\n**进度说明：** {progress}\n**后续计划：** {next_plan}` |
| ✅ 任务完成 | `green` | `✅ 任务已完成` | `**完成说明：** {completion_summary}\n**变更文件：** {changed_files}\n**执行摘要：** {execution_summary}` |
| ⚠️ 阻塞确认 | `orange` | `⚠️ 任务执行需要确认` | `**当前进度：** {current_progress}\n**阻塞原因：** {reason}\n\n**需要确认的问题：**\n{questions}\n\n请在 Claude Code 终端中回复，或通过飞书任务评论补充说明。` |
| ❌ 执行异常 | `red` | `❌ 任务执行异常` | `**当前进度：** {current_progress}\n**错误详情：** {error_detail}\n**影响范围：** {impact}\n\n请检查后告知是否继续。` |

---

## 占位符总览

| 占位符 | 适用卡片 | 说明 |
|-------|---------|------|
| `{summary}` / `{url}` / `{task_url}` | 全部 | 任务标题、链接（详情/按钮共用） |
| `{execution_plan}` / `{eta}` | 🚀 | 执行计划要点（换行编号）、预计完成时间 |
| `{status}` / `{progress}` / `{next_plan}` | 📋 | 当前状态、已完成进展、下一步计划 |
| `{completion_summary}` / `{changed_files}` / `{execution_summary}` | ✅ | 完成情况、变更文件列表、执行摘要回顾 |
| `{current_progress}` / `{reason}` / `{questions}` | ⚠️ | 阻塞前进度、原因、待确认问题列表 |
| `{current_progress}` / `{error_detail}` / `{impact}` | ❌ | 出错前进度、错误详情、影响范围 |

---

## 可用主题色

`blue` / `turquoise` / `green` / `orange` / `red` / `violet` / `indigo` / `wathet` / `yellow` / `grey` / `carmine`

---

## elements 常用组件

```json
// 文本块
{ "tag": "div", "text": { "tag": "lark_md", "content": "**加粗** [链接](url)\n换行" } }

// 分隔线
{ "tag": "hr" }

// 按钮组
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
```

- `lark_md` 支持加粗 `**x**`、链接 `[x](url)`、换行 `\n`
- 按钮 `type`：`primary` / `default` / `danger`
- 按钮支持 `url`（跳转）或 `value`（回调）
