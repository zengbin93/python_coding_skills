---
name: python-code-quality
description: >
  Python 代码风格与质量检查规范，使用 ruff 进行格式化和 lint 检查，使用 basedpyright 进行类型检查。
  在编写、修改或检查 Python 代码时使用此 skill。
---

# Python 代码质量规范

## 工具配置

本项目使用以下工具保障代码质量：

- **ruff** — 代码格式化 + lint 检查（替代 black、isort、flake8）
- **basedpyright** — 严格的静态类型检查（替代 mypy）

## 快速命令

```bash
# 格式化代码（自动修复）
ruff format .

# Lint 检查（自动修复可修复的问题）
ruff check . --fix

# 仅检查不修复
ruff check .

# 类型检查
basedpyright

# 对单个文件进行类型检查
basedpyright src/module.py
```

## ruff 配置（pyproject.toml）

```toml
[tool.ruff]
target-version = "py310"
line-length = 120
fix = true

[tool.ruff.lint]
select = [
    "E",   # pycodestyle 错误
    "W",   # pycodestyle 警告
    "F",   # pyflakes
    "I",   # isort
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "UP",  # pyupgrade
    "SIM", # flake8-simplify
    "RET", # flake8-return
    "N",   # pep8-naming
]
ignore = [
    "E501",  # 行长度（由 formatter 控制）
]

[tool.ruff.lint.isort]
known-first-party = ["your_package_name"]  # 替换为实际的包名

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

## basedpyright 配置（pyproject.toml）

```toml
[tool.basedpyright]
pythonVersion = "3.10"
typeCheckingMode = "standard"
reportMissingImports = true
reportMissingTypeStubs = false

# 严格模式（推荐用于新项目）
# typeCheckingMode = "strict"
```

## 代码风格规范

### 类型注解

- **始终**为函数参数和返回值添加类型注解
- 使用 `X | None` 替代 `Optional[X]`（Python 3.10+）
- 使用 `X | Y` 替代 `Union[X, Y]`（Python 3.10+）
- 使用 `list[X]`、`dict[K, V]`、`tuple[X, ...]` 替代 `List`、`Dict`、`Tuple`

```python
# ✅ 正确
def process(data: list[str], limit: int | None = None) -> dict[str, int]:
    ...

# ❌ 错误
from typing import List, Optional, Dict
def process(data: List[str], limit: Optional[int] = None) -> Dict[str, int]:
    ...
```

### 字符串格式化

- 优先使用 f-string，避免使用 `%` 格式化和 `.format()`

```python
# ✅ 正确
name = "world"
message = f"Hello, {name}!"

# ❌ 错误
message = "Hello, %s!" % name
message = "Hello, {}!".format(name)
```

### 导入顺序

遵循 isort 规范（ruff 自动处理）：
1. 标准库
2. 第三方库
3. 本地模块

```python
# ✅ 正确
import os
import sys
from pathlib import Path

import numpy as np
import pandas as pd

from mypackage.utils import helper
```

### 文档字符串

使用 Google 风格的 docstring：

```python
def calculate_discount(price: float, rate: float) -> float:
    """计算折扣后的价格。

    Args:
        price: 原始价格，必须为正数。
        rate: 折扣率，范围 0.0 到 1.0。

    Returns:
        折扣后的价格。

    Raises:
        ValueError: 当 rate 不在 [0, 1] 范围内时抛出。

    Examples:
        >>> calculate_discount(100.0, 0.2)
        80.0
    """
    if not 0.0 <= rate <= 1.0:
        raise ValueError(f"折扣率必须在 0 到 1 之间，当前值: {rate}")
    return price * (1 - rate)
```

### 命名规范

| 类型 | 风格 | 示例 |
|------|------|------|
| 模块 | snake_case | `user_service.py` |
| 类 | PascalCase | `UserService` |
| 函数/方法 | snake_case | `get_user_by_id` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRIES` |
| 私有 | 前缀 `_` | `_internal_state` |
| 类型变量 | PascalCase + T | `UserT` |

### 异常处理

- 捕获具体异常，不使用裸 `except:`
- 异常信息应包含上下文

```python
# ✅ 正确
try:
    result = risky_operation()
except ValueError as e:
    logger.error(f"参数无效: {e}")
    raise

# ❌ 错误
try:
    result = risky_operation()
except:
    pass
```

## CI 集成

在 GitHub Actions 中集成质量检查：

```yaml
- name: Lint and Format Check
  run: |
    ruff format --check .
    ruff check .

- name: Type Check
  run: basedpyright
```

## 常见 ruff 错误及修复

| 规则 | 说明 | 修复方法 |
|------|------|---------|
| `F401` | 未使用的导入 | 删除未使用的 import |
| `B006` | 可变默认参数 | 使用 `None` 替代可变默认值 |
| `SIM108` | 三元表达式可简化 | 使用三元运算符 |
| `UP006` | 使用旧式类型注解 | 使用内置类型替代 typing 模块 |
| `RET504` | 不必要的赋值后返回 | 直接返回表达式 |
