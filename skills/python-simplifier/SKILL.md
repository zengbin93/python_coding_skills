---
name: python-simplifier
description: >
  Python 代码重构指南，目标是提高可读性、消除逻辑冗余和重复代码，同时完整保留原有功能。
  在重构或审查 Python 代码时使用此 skill。
---

# Python 代码简化与重构指南

## 核心原则

1. **保持功能不变** — 重构只改变代码结构，不改变代码行为
2. **可读性优先** — 代码是给人读的，其次才是给机器执行的
3. **单一职责** — 每个函数和类只做一件事
4. **消除重复** — DRY（Don't Repeat Yourself）原则
5. **适度简化** — 避免过度精简导致可读性下降

---

## 消除重复代码

### 提取公共逻辑为函数

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

```python
# ❌ 旧方式
merged = {**defaults, **overrides}

# ✅ 更清晰的方式
merged = defaults | overrides
```

---

## 消除魔法值

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

## 重构检查清单

在重构前后对照此清单：

**识别重构机会：**
- [ ] 存在重复代码块（通常 ≥3 行相同逻辑，实际可根据代码复杂度灵活判断）
- [ ] 函数超过 30 行
- [ ] 嵌套层级超过 3 层
- [ ] 函数参数超过 5 个
- [ ] 存在魔法数字/字符串
- [ ] 长链 if-elif（超过 4 个分支）

**重构后验证：**
- [ ] 所有测试通过
- [ ] 代码可读性提升
- [ ] 没有引入新的副作用
- [ ] 类型注解完整
- [ ] ruff 和 basedpyright 检查无报错
