---
name: python-simplifier
description: >
  在简化或重构 Python 代码，且存在重复逻辑、深层嵌套、冗长分支或无意义中间变量，同时必须保持行为不变时使用此 skill。
---

# Python 代码简化与重构指南

## 概述

当 Python 代码的可读性、层次结构或重复度已经影响理解和维护，但又不能改变既有行为时，使用此 skill。

目标不是“写得更短”，而是在**完全保留原有功能、输出、副作用、异常行为和公开接口**的前提下，降低复杂度、提升可读性，并让后续修改更安全。

## 适用场景

在出现以下一种或多种情况时使用：

- 存在重复的校验、转换或分支逻辑
- 条件嵌套较深，主流程不清晰
- `if/elif` 分支链很长，且大多是值分发
- 存在没有表达额外语义的中间变量
- 数据类或样板代码过多，掩盖了核心意图
- 手动循环比推导式或内置函数更啰嗦
- 魔法数字、魔法字符串降低了代码可理解性

## 不适用场景

以下情况不要使用此 skill 作为主要指导：

- 跨模块重构、架构重设计
- 以性能优化为主要目标的改动
- 会改变 API、数据结构、异常协议或业务行为的修改
- 当前任务只涉及局部代码，却顺手发散成大范围清理
- 为了减少行数而引入更晦涩、更难调试的写法

## 范围约束

- 默认只处理当前任务涉及的代码、当前 diff 或用户明确指定的文件。
- 优先做局部、可证明安全的简化，不做仓库级“顺手整理”。
- 不要仅因为逻辑相邻，就把多个职责硬塞进同一个函数。
- 如果无法证明语义保持不变，应停止简化并保留原实现。

## 行为保持清单

除非用户明确要求，否则重构时必须保持以下内容不变：

- 返回值与返回类型
- 副作用与对象变更行为
- 求值顺序
- 惰性/立即求值语义
- 对外可见或被测试覆盖的异常类型与异常信息
- 日志、指标、追踪、重试等行为
- 公共函数签名与数据模型结构

## 与其他 Python Skills 的分工

- `python-code-quality`：负责格式化、lint、类型检查和基础规范
- `python-best-practices`：负责更广义的 Python 设计、性能、数据结构和并发建议
- `python-code-review`：负责整体质量审查与提交前检查
- `python-simplifier`：专注于在**不改变行为**的前提下降低复杂度、消除重复、提升可读性

## 简化工作流

1. 先确定最小可安全修改范围。
2. 判断当前主要问题属于重复、嵌套、分支膨胀、样板代码还是状态噪音。
3. 选择最清晰的局部改法，而不是最短的写法。
4. 保持行为完全一致，包括边界条件与失败路径。
5. 用测试、`ruff` 和 `basedpyright` 验证结果。
6. 只说明那些真正提升理解成本的改动。

## 核心原则

1. **保持功能不变**：重构只改变代码结构，不改变代码行为
2. **可读性优先**：代码首先是给人读的，其次才是给机器执行的
3. **单一职责**：每个函数和类只做一件事
4. **消除重复**：遵循 DRY（Don't Repeat Yourself）原则
5. **适度简化**：避免过度精简导致可读性下降

---

## 消除重复代码

### 提取公共逻辑为函数

仅当重复逻辑在语义上确实相同，而不只是“看起来像”时，才进行提取。

```python
# ❌ 重复代码
def process_admin(user):
    if not user.get("name"):
        raise ValueError("名称不能为空")
    if len(user["name"]) > 50:
        raise ValueError("名称过长")
    # admin 逻辑...

def process_member(user):
    if not user.get("name"):
        raise ValueError("名称不能为空")
    if len(user["name"]) > 50:
        raise ValueError("名称过长")
    # member 逻辑...

# ✅ 提取公共验证
def validate_user_name(user: dict) -> None:
    if not user.get("name"):
        raise ValueError("名称不能为空")
    if len(user["name"]) > 50:
        raise ValueError("名称过长")

def process_admin(user: dict) -> None:
    validate_user_name(user)
    # admin 逻辑...

def process_member(user: dict) -> None:
    validate_user_name(user)
    # member 逻辑...
```

### 使用参数化替代相似函数

如果多个函数只有输入常量不同，而流程完全一致，可以考虑参数化。

```python
# ❌ 相似的多个函数
def send_welcome_email(user_email: str) -> None:
    subject = "欢迎加入！"
    body = "感谢您的注册..."
    _send_email(user_email, subject, body)

def send_reset_email(user_email: str) -> None:
    subject = "密码重置"
    body = "点击链接重置密码..."
    _send_email(user_email, subject, body)

# ✅ 统一函数 + 枚举
from enum import Enum

class EmailTemplate(Enum):
    WELCOME = ("欢迎加入！", "感谢您的注册...")
    RESET = ("密码重置", "点击链接重置密码...")

def send_email(user_email: str, template: EmailTemplate) -> None:
    subject, body = template.value
    _send_email(user_email, subject, body)
```

---

## 简化条件逻辑

### 提前返回替代深层嵌套

当每个失败条件都能自然提前退出时，优先用 Guard Clause 展开主流程。

```python
# ❌ 深层嵌套（箭头形代码）
def process_order(order):
    if order is not None:
        if order.is_valid():
            if order.user.is_active():
                if order.items:
                    return fulfill_order(order)
    return None

# ✅ 提前返回（Guard Clause）
def process_order(order: Order | None) -> OrderResult | None:
    if order is None:
        return None
    if not order.is_valid():
        return None
    if not order.user.is_active():
        return None
    if not order.items:
        return None
    return fulfill_order(order)
```

### 用字典替代长链 if-elif

适用于“输入值 -> 输出值”的简单映射；如果分支里还有校验、副作用或多步逻辑，就不要强行替换。

```python
# ❌ 冗长的 if-elif 链
def get_status_label(status: str) -> str:
    if status == "pending":
        return "待处理"
    elif status == "processing":
        return "处理中"
    elif status == "completed":
        return "已完成"
    elif status == "cancelled":
        return "已取消"
    else:
        return "未知"

# ✅ 字典映射
STATUS_LABELS: dict[str, str] = {
    "pending": "待处理",
    "processing": "处理中",
    "completed": "已完成",
    "cancelled": "已取消",
}

def get_status_label(status: str) -> str:
    return STATUS_LABELS.get(status, "未知")
```

### 用 match-case 处理复杂分支（Python 3.10+）

当你表达的是结构化模式匹配，而不仅仅是简单值映射时，`match-case` 会更清晰。

```python
# ✅ 使用 match-case
def handle_event(event: dict) -> str:
    match event["type"]:
        case "click":
            return handle_click(event)
        case "hover":
            return handle_hover(event)
        case "submit":
            return handle_submit(event)
        case _:
            return handle_unknown(event)
```

---

## 简化集合操作

### 使用推导式替代循环

推导式适合表达简单、线性的转换。过长或多层嵌套的推导式反而会降低可读性。

```python
# ❌ 命令式循环
result = []
for item in items:
    if item.is_active():
        result.append(item.name.upper())

# ✅ 列表推导式
result = [item.name.upper() for item in items if item.is_active()]

# 字典推导式
user_map = {user.id: user.name for user in users}

# 集合推导式
unique_tags = {tag for item in items for tag in item.tags}
```

### 使用内置函数

当 `sum`、`any`、`all`、`max` 等内置函数能够更直接表达意图时，优先使用它们。

```python
# ❌ 手动循环
total = 0
for price in prices:
    total += price

has_admin = False
for user in users:
    if user.role == "admin":
        has_admin = True
        break

# ✅ 内置函数
total = sum(prices)
has_admin = any(user.role == "admin" for user in users)
all_active = all(user.is_active() for user in users)
max_price = max(prices, default=0)
```

---

## 简化函数设计

### 移除不必要的中间变量

只有在中间变量没有承载领域语义、也不利于调试时，才考虑删除。

```python
# ❌ 多余的中间赋值
def is_valid_age(age: int) -> bool:
    result = age >= 0 and age <= 150
    return result

# ✅ 直接返回
def is_valid_age(age: int) -> bool:
    return 0 <= age <= 150
```

### 使用 dataclass 替代繁琐的类定义

适用于以数据承载为主的类；如果类包含复杂不变量、构造规则或行为逻辑，就不要机械替换。

```python
# ❌ 样板代码过多
class Point:
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y

    def __repr__(self):
        return f"Point(x={self.x}, y={self.y})"

    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

# ✅ 使用 dataclass
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float
```

### 使用 `|` 合并字典（Python 3.9+）

在目标 Python 版本允许的前提下，优先选择更清晰的现代语法。

```python
# ❌ 旧方式
merged = {**defaults, **overrides}

# ✅ 更清晰的方式
merged = defaults | overrides
```

---

## 消除魔法值

用命名常量、枚举或语义化配置替代含义不明的数字和字符串。

```python
# ❌ 含义不明的数字和字符串
if status == 2:
    send_alert()

if response.status_code == 429:
    time.sleep(60)

# ✅ 使用命名常量或枚举
from enum import IntEnum

class OrderStatus(IntEnum):
    PENDING = 1
    CONFIRMED = 2
    SHIPPED = 3

RATE_LIMIT_STATUS = 429
RATE_LIMIT_WAIT_SECONDS = 60

if status == OrderStatus.CONFIRMED:
    send_alert()

if response.status_code == RATE_LIMIT_STATUS:
    time.sleep(RATE_LIMIT_WAIT_SECONDS)
```

---

## 重构上下文管理

优先使用上下文管理器，减少资源泄漏风险，并让资源生命周期更明显。

```python
# ❌ 手动资源管理
file = open("data.txt")
try:
    content = file.read()
finally:
    file.close()

# ✅ 使用 with 语句
with open("data.txt") as file:
    content = file.read()

# 多个上下文管理器
with open("input.txt") as src, open("output.txt", "w") as dst:
    dst.write(src.read())
```

---

## 常见误区

- 不要把所有循环都改成推导式。过长或多层推导式往往更难读。
- 不要把所有 `if/elif` 都改成字典。只要分支里有副作用、多步逻辑或校验，原写法可能更清晰。
- 不要把所有重复都提成 helper。抽象过度会把上下文切碎，反而影响理解。
- 不要为了“代码更短”把实现压成一行，牺牲调试性和可维护性。
- 不要把所有类都改成 `dataclass`。行为型类和带不变量的类往往需要显式结构。
- 不要合并“形似但语义不同”的逻辑，否则后续业务演化会变得更危险。

## 快速判断启发式

- 重复逻辑出现 2 次以上：检查是否真的可以抽取
- 嵌套超过 3 层：优先考虑 Guard Clause
- 值分发分支超过 4 个：考虑字典映射或 `match-case`
- 函数长度超过约 30 行：检查是否混入多个职责
- 参数超过 5 个：检查是否需要更清晰的数据对象或接口

这些只是提醒你检查，不是自动改写规则。

## 重构检查清单

在重构前后对照此清单：

**识别重构机会：**
- [ ] 存在重复代码块（通常 ≥3 行相同逻辑，实际可根据代码复杂度灵活判断）
- [ ] 函数超过 30 行
- [ ] 嵌套层级超过 3 层
- [ ] 函数参数超过 5 个
- [ ] 存在魔法数字/字符串
- [ ] 长链 if-elif（超过 4 个分支）

**保持行为一致：**
- [ ] 返回值、异常、日志和副作用保持不变
- [ ] 没有改变求值顺序或惰性行为
- [ ] 没有修改公共接口或数据结构

**重构后验证：**
- [ ] 所有测试通过
- [ ] 代码可读性提升
- [ ] 没有引入新的副作用
- [ ] 类型注解完整
- [ ] `ruff format`、`ruff check` 和 `basedpyright` 检查无报错

## 底线

好的 Python 简化，减少的是阅读和维护阻力，而不是信息量。

优先做显式、局部、可证明安全的重构，让下一位读代码的人更快理解，也更不容易改坏。
