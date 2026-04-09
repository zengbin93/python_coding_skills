# lark-xxx 系列 Skill 使用指南

`lark-*` 是飞书官方发布的 Claude Code Skill 套件，统一封装 `lark-cli` 命令行工具，覆盖飞书所有主要业务模块。本 skill（feishu-task-execute）在执行任务过程中应主动调用对应的 lark 子 skill，而不是自行猜测 lark-cli 命令。

> 触发原则：**遇到任何飞书业务操作 → 先想"有没有对应的 lark-* skill" → 用 Skill 工具加载它 → 按其指引操作**。不要凭记忆敲 `lark-cli` 命令。

## 调用方式

通过 `Skill` 工具加载，例如：

```
Skill(skill="lark-doc")
Skill(skill="lark-im")
Skill(skill="lark-whiteboard")
```

加载后该 skill 的指令会注入上下文，按其工作流执行即可。

## 共享前置：lark-shared

**所有 lark-* skill 都依赖 `lark-shared`**，它定义了认证、身份切换、权限申请、错误处理的统一规则。

何时触发：
- 第一次使用 lark-cli（需要 `lark-cli config init` + `auth login`）
- 遇到 `Permission denied` / scope 不足
- 需要在 `--as user` / `--as bot` 之间切换
- 想知道如何申请缺失的 OpenAPI 权限

## 业务模块速查表

按"我要做什么"选择对应 skill：

| 我要做什么 | 用哪个 skill | 关键命令 |
|-----------|-------------|---------|
| 收发 IM 消息、群聊管理、消息卡片 | `lark-im` | `im +messages-send` / `im +messages-search` |
| 读写飞书云文档（docx）、搜索文档 | `lark-doc` | `docs +fetch/+create/+update/+search` |
| 在文档里画流程图、思维导图、架构图 | `lark-whiteboard` | `docs +whiteboard-update` |
| 操作电子表格 | `lark-sheets` | `sheets +read/+write/+append` |
| 操作多维表格（Base） | `lark-base` | 建表/字段/记录/视图/公式 |
| 云空间文件管理（上传/下载/移动） | `lark-drive` | `drive +upload/+download` |
| 知识库节点（wiki） | `lark-wiki` | 空间/节点/层级管理 |
| 任务/待办管理 | `lark-task` | `task tasks get` / `task +comment` |
| 日历日程、忙闲、会议邀请 | `lark-calendar` | `+agenda/+create/+freebusy/+suggestion` |
| 视频会议记录与纪要 | `lark-vc` | 查询会议、获取总结/逐字稿 |
| 妙记（minutes）AI 产物 | `lark-minutes` | 总结/待办/章节 |
| 邮件起草/收发/搜索 | `lark-mail` | 草稿/发送/回复/搜索 |
| 通讯录查询、找人、部门结构 | `lark-contact` | 用户/部门/搜索 |
| 实时事件订阅（WebSocket） | `lark-event` | 监听消息/通讯录/日历变更 |

## 工作流编排类（多步组合）

| 场景 | Skill |
|------|-------|
| 从飞书任务驱动开发（本 skill） | `feishu-task-execute` |
| 整理一段时间内的会议纪要 → 报告 | `lark-workflow-meeting-summary` |
| 生成今日/本周日程 + 待办摘要 | `lark-workflow-standup-report` |

## 进阶：原生 OpenAPI 与自定义 Skill

| 场景 | Skill |
|------|-------|
| 现有 lark-* skill 和 lark-cli 都不能满足，需要查找原生 OpenAPI | `lark-openapi-explorer` |
| 把一组飞书操作封装成自己的 Skill | `lark-skill-maker` |

## feishu-task-execute 与其它 lark-* skill 的协作

本 skill 是**编排者**，常用以下子 skill 完成具体动作：

| 阶段 | 调用的 skill | 用途 |
|------|------------|------|
| Step 2 读取任务 | `lark-task` | `task tasks get` 拿任务详情、子任务、评论 |
| Step 3 读取关联文档 | `lark-doc` / `lark-wiki` / `lark-sheets` | 从任务描述里抓 docx/wiki/sheets 链接读详情 |
| Step 4/5/6 发送进度卡片 | `lark-im` | 用 schema 2.0 卡片向负责人推送 🚀📋✅⚠️❌ |
| 文档产出含图表 | `lark-whiteboard` | 流程图/思维导图/架构图统一走白板 |
| Step 6.2 任务评论 | `lark-task` | `task +comment` 添加纯文本技术评论 |
| 权限/认证问题 | `lark-shared` | 切换身份、申请 scope、排查 Permission denied |

## 选用决策树

```
飞书相关需求
├── 是任务驱动的工作流？           → feishu-task-execute（本 skill）
├── 操作单一业务模块？             → 对应 lark-<模块> skill
│   ├── IM 消息 → lark-im
│   ├── 文档    → lark-doc
│   ├── 表格    → lark-sheets / lark-base
│   ├── 任务    → lark-task
│   ├── 日历    → lark-calendar
│   └── ...
├── 需要画图？                     → lark-whiteboard
├── 多步组合的固定工作流？         → lark-workflow-* 系列
├── 现有 skill 都不行 → 查原生 API → lark-openapi-explorer
├── 想沉淀成可复用 skill           → lark-skill-maker
└── 任何身份/权限/认证问题         → lark-shared
```

## 共同最佳实践

- **先 Skill 后命令**：不要凭记忆敲 `lark-cli`，先加载对应 skill 拿到最新命令签名
- **身份默认 user**：除 `im +messages-send`（必须 bot）外，其它 docs/sheets/drive/wiki/task 操作优先 `--as user`
- **权限报错先看 lark-shared**：所有 `Permission denied` 的处理流程都在 `lark-shared` 中
- **不要混用多 skill**：一次操作只用一个 skill 的命令，避免命令格式互相污染
- **schema 2.0 卡片**：通过 `lark-im` 发送的交互式卡片必须用 schema 2.0（`body.elements`），见 [`card-templates.md`](card-templates.md)
