# python_coding_skills

Python 编程相关的 skill 仓库，旨在指导大模型生成高质量的 Python 代码。所有 skill 均使用中文编写，可注册到 Claude Code 等大模型插件系统中使用。

## Skills 列表

| Skill | 说明 |
|-------|------|
| [python-code-quality](.github/skills/python-code-quality/SKILL.md) | 代码风格与质量检查规范，使用 ruff 和 basedpyright 实现 |
| [python-pytest](.github/skills/python-pytest/SKILL.md) | 使用 pytest 编写高质量单元测试的通用规范 |
| [python-simplifier](.github/skills/python-simplifier/SKILL.md) | Python 代码重构简化指南，提高可读性、消除冗余 |
| [python-best-practices](.github/skills/python-best-practices/SKILL.md) | Python 编程最佳实践，指导编写高性能、高可读性代码 |
| [python-code-review](.github/skills/python-code-review/SKILL.md) | Python 代码审查规范，评估代码是否符合本仓库制定的质量标准 |

## 使用方式

### 在 Claude Code 中注册

将本仓库中的 `.github/skills/` 目录下的 skill 注册到 Claude Code 插件系统：

1. 打开 Claude Code 设置
2. 添加 Skills 仓库，指向本仓库地址
3. 选择需要启用的 skill

### 直接引用

也可以直接将 `SKILL.md` 文件的内容复制到你的 `CLAUDE.md` 等项目配置文件中作为上下文参考。

## Skill 格式说明

每个 skill 位于 `.github/skills/<skill-name>/SKILL.md`，文件格式如下：

```markdown
---
name: skill-name
description: >
  简要描述该 skill 的用途和适用场景。
---

# Skill 标题

详细内容...
```
