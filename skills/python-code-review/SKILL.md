---
name: python-code-review
description: >
  评估 Python 代码是否符合本仓库制定的质量标准，涵盖代码风格、类型注解、测试覆盖、
  可读性和性能。在提交代码前或审查他人代码时使用此 skill。
---

# Python 代码审查规范

## 审查目标

对照本仓库制定的以下标准评估代码质量：

- **[python-code-quality]** — 代码风格、类型注解、命名规范、文档字符串
- **[python-pytest]** — 测试结构、覆盖率、测试可维护性
- **[python-simplifier]** — 复杂度、重复代码、逻辑冗余
- **[python-best-practices]** — 数据结构选择、性能、并发模型、常见陷阱

---

## 审查维度与检查项

### 1. 代码风格（对照 python-code-quality）

**类型注解**

- [ ] 所有函数的参数和返回值都有类型注解
- [ ] 使用 `X | None` 而非 `Optional[X]`
- [ ] 使用内置泛型（`list[X]`、`dict[K, V]`）而非 `typing.List`、`typing.Dict`
- [ ] 文件顶部有 `from __future__ import annotations`（视项目配置）

**命名规范**

- [ ] 类名使用 PascalCase
- [ ] 函数、变量使用 snake_case
- [ ] 常量使用 UPPER_SNAKE_CASE
- [ ] 私有成员以 `_` 开头
- [ ] 没有遮蔽内置名称（`list`、`type`、`next` 等）

**文档字符串**

- [ ] 所有公开函数、类都有 Google 风格 docstring
- [ ] 有说明参数、返回值和可能抛出的异常
- [ ] 文档字符串与实际实现一致

**工具检查**

- [ ] `ruff check .` 无报错
- [ ] `ruff format --check .` 无格式问题
- [ ] `basedpyright` 无类型错误

---

### 2. 测试质量（对照 python-pytest）

**覆盖率与完整性**

- [ ] 核心逻辑测试覆盖率 ≥ 80%
- [ ] 正常路径、异常路径、边界值均有测试
- [ ] 每个测试只验证一个行为

**可读性**

- [ ] 测试名称清晰描述被测场景和预期结果
- [ ] 遵循 Arrange-Act-Assert（AAA）模式
- [ ] 没有调试用的 `print` 语句残留

**可维护性**

- [ ] 共享 fixture 放在 `conftest.py`
- [ ] Mock 只针对外部依赖（数据库、HTTP、文件系统）
- [ ] 无硬编码路径或特定环境的依赖

---

### 3. 代码简洁性（对照 python-simplifier）

**复杂度**

- [ ] 函数不超过 30 行
- [ ] 嵌套层级不超过 3 层
- [ ] 函数参数不超过 5 个（否则考虑用 dataclass 封装）

**重复代码**

- [ ] 无明显重复逻辑（相同代码块出现 2 次以上应提取）
- [ ] 相似函数已通过参数化合并

**可读性**

- [ ] 无魔法数字或魔法字符串（使用常量或枚举）
- [ ] 长链 `if-elif`（≥4 个分支）已替换为字典映射或 `match-case`
- [ ] 使用提前返回（Guard Clause）替代深层嵌套

---

### 4. 最佳实践（对照 python-best-practices）

**数据结构**

- [ ] 成员检测使用 `set` 而非 `list`
- [ ] 频次统计使用 `Counter`
- [ ] 队列操作使用 `deque` 而非 `list`

**性能**

- [ ] 大数据集使用生成器而非列表
- [ ] 重复计算结果有缓存（`lru_cache` / `cache`）
- [ ] 字符串拼接使用 `join` 而非 `+=`

**常见陷阱**

- [ ] 没有可变默认参数（`def f(x=[]):`）
- [ ] 没有在闭包中错误引用循环变量
- [ ] 与 `None` 的比较使用 `is`/`is not`

---

## 审查结论格式

在审查结束时，按以下格式给出结论：

```
## 代码审查结论

### ✅ 符合标准的方面
- 类型注解完整，使用了 Python 3.10+ 语法
- 测试遵循 AAA 模式，命名清晰
- ...

### ⚠️ 需要改进的方面
1. `process_data()` 函数缺少返回值类型注解
   - 位置：`src/processor.py:42`
   - 建议：添加 `-> list[Result]`

2. 测试覆盖率不足
   - 位置：`tests/test_processor.py`
   - 建议：补充异常路径测试（当输入为空列表时）

3. 魔法数字
   - 位置：`src/config.py:15`：`if retries > 3:`
   - 建议：提取为常量 `MAX_RETRIES = 3`

### 🚫 必须修复的问题（阻断合并）
- `basedpyright` 报告 2 个类型错误（见下方详情）
- `ruff` 报告 1 个 B006 错误（可变默认参数）

### 总体评分
- 代码风格：⭐⭐⭐⭐☆
- 测试质量：⭐⭐⭐☆☆
- 代码简洁性：⭐⭐⭐⭐⭐
- 最佳实践：⭐⭐⭐⭐☆
```

---

## 快速检查命令

在提交或合并前运行以下命令确保基础质量：

```bash
# 格式化 + lint
ruff format .
ruff check . --fix

# 类型检查
basedpyright

# 运行相关测试（只跑本次修改涉及的模块）
pytest tests/test_affected_module.py -v

# 带覆盖率
pytest tests/test_affected_module.py --cov=src/affected_module --cov-report=term-missing
```
