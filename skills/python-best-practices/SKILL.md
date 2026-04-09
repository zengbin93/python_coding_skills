---
name: python-best-practices
description: >
  Python 编程最佳实践，涵盖性能优化、可读性提升、数据结构选择、并发编程和常见陷阱规避。
  在编写高质量 Python 代码时参考此 skill。
---

# Python 编程最佳实践

## 类型注解

- **始终**为函数参数和返回值添加类型注解
- 使用 `Type | None` 替代 `Optional[Type]`（Python 3.10+）
- 在文件顶部添加 `from __future__ import annotations`，启用延迟类型求值
- TypeVar 命名以 `T` 为后缀：`ChatResponseT = TypeVar("ChatResponseT", bound=ChatResponse)`
- 只读输入参数使用 `Mapping` 而非 `MutableMapping`
- 对于内部调用的方法，优先使用 `# type: ignore[具体规则]` 而非多余的类型转换或 `isinstance` 检查；注释需同时兼顾 mypy 和 pyright，避免遗漏其他错误

```python
from __future__ import annotations
from collections.abc import Mapping
from typing import TypeVar

ResponseT = TypeVar("ResponseT", bound=BaseResponse)

# ✅ 正确
def process(data: list[str], limit: int | None = None) -> dict[str, int]:
    ...

# ❌ 错误
from typing import List, Optional, Dict
def process(data: List[str], limit: Optional[int] = None) -> Dict[str, int]:
    ...
```

---

## 函数参数

- 位置参数：最多 3 个完全必须的参数
- 可选参数放在 `*` 之后（强制关键字参数）
- 提供基于字符串的重载，避免调用方引入额外的 import

```python
from typing import Literal

def create_agent(
    name: str,
    tool_mode: Literal["auto", "required", "none"] | ChatToolMode,
) -> Agent:
    if isinstance(tool_mode, str):
        tool_mode = ChatToolMode(tool_mode)
    ...
```

- 避免遮蔽内置名称（用 `next_item` 替代 `next`，用 `type_` 替代 `type`）
- 除非需要子类扩展，否则避免使用 `**kwargs`；优先使用具名参数

---

## 文档字符串

对所有公开 API 使用 Google 风格 docstring：

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

- 始终为自定义异常添加说明
- 有关键字参数时显式使用 `Keyword Args:` 块
- 标准 Python 异常仅在触发条件不显而易见时才需文档说明

---

## 数据结构选择

### 选择正确的容器类型

```python
from collections import defaultdict, Counter, deque

# ✅ 使用 set 进行成员检测（O(1) vs list 的 O(n)）
valid_statuses = {"pending", "active", "closed"}
if status in valid_statuses:
    ...

# ✅ 使用 defaultdict 避免 KeyError
word_count: defaultdict[str, int] = defaultdict(int)
for word in words:
    word_count[word] += 1

# ✅ 使用 Counter 统计频次
from collections import Counter
top_words = Counter(words).most_common(10)

# ✅ 使用 deque 实现队列（两端 O(1)，list 头部操作是 O(n)）
queue: deque[str] = deque()
queue.append("item")
queue.appendleft("first")
first = queue.popleft()
```

### 使用 dataclass、namedtuple 和 pydantic

```python
from dataclasses import dataclass, field
from typing import NamedTuple
from pydantic import BaseModel, Field

# ✅ 不可变数据用 NamedTuple
class Coordinate(NamedTuple):
    latitude: float
    longitude: float

# ✅ 可变结构用 dataclass
@dataclass
class Config:
    host: str = "localhost"
    port: int = 8080
    tags: list[str] = field(default_factory=list)

# ✅ 不可变且需要性能时使用 frozen=True
@dataclass(frozen=True)
class CacheKey:
    user_id: int
    resource: str

# ✅ 边界数据、配置加载、接口入参校验优先使用 Pydantic
class UserProfile(BaseModel):
    user_id: int
    name: str = Field(min_length=1)
    email: str
    tags: list[str] = Field(default_factory=list)
```

- `dataclass` 适合内部领域对象和轻量数据载体
- `NamedTuple` 适合不可变、偏位置语义的小型结构
- `pydantic` 适合系统边界的数据建模：配置解析、API 请求/响应、消息载荷校验与序列化
- 需要运行时校验、默认值规范化、字段约束、JSON Schema/OpenAPI 生成时，优先使用 `pydantic`

---

## 性能优化

### 生成器节省内存

```python
# ❌ 一次性加载所有数据到内存
def read_large_file(path: str) -> list[str]:
    with open(path) as f:
        return f.readlines()  # 可能占用大量内存

# ✅ 使用生成器按需处理
def read_large_file(path: str):
    with open(path) as f:
        yield from f

# ✅ 生成器表达式替代列表推导（当不需要随机访问时）
total = sum(x * x for x in range(1_000_000))  # 内存友好
```

### 缓存计算结果

```python
from functools import lru_cache, cache

# ✅ 缓存纯函数的计算结果
@cache  # Python 3.9+，等价于 lru_cache(maxsize=None)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# ✅ 带大小限制的 LRU 缓存
@lru_cache(maxsize=128)
def get_user_permissions(user_id: int) -> frozenset[str]:
    return frozenset(db.query_permissions(user_id))
```

### 字符串拼接

```python
# ❌ 循环中拼接字符串（每次创建新对象，O(n²)）
result = ""
for item in items:
    result += str(item) + ", "

# ✅ 使用 join（O(n)）
result = ", ".join(str(item) for item in items)

# ✅ 大量格式化用 f-string
parts = [f"{k}={v}" for k, v in config.items()]
output = ", ".join(parts)
```

---

## 并发与异步

### 选择正确的并发模型

```python
# I/O 密集型网络任务 → asyncio
import asyncio
import httpx

async def fetch_all(urls: list[str]) -> list[dict]:
    async with httpx.AsyncClient() as client:
        tasks = [client.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)
        return [r.json() for r in responses]

# 多线程、多进程任务统一使用 joblib
from joblib import Parallel, delayed

def process_data(chunk: list) -> list:
    return [expensive_computation(item) for item in chunk]

def parallel_process(data: list, workers: int = 4) -> list[int]:
    return Parallel(n_jobs=workers, prefer="processes")(
        delayed(expensive_computation)(item) for item in data
    )

def parallel_fetch(paths: list[str], workers: int = 8) -> list[str]:
    return Parallel(n_jobs=workers, prefer="threads")(
        delayed(read_text)(path) for path in paths
    )
```

- 协程并发和线程/进程并发分开看：异步 I/O 用 `asyncio`，多线程/多进程统一用 `joblib`
- `joblib.Parallel(..., prefer="threads")` 适合 I/O 密集型阻塞任务
- `joblib.Parallel(..., prefer="processes")` 适合 CPU 密集型任务
- 默认优先 `loky` 进程后端，避免直接散落 `ThreadPoolExecutor` / `ProcessPoolExecutor` 的实现细节
- 需要共享大数组或缓存中间结果时，优先利用 `joblib` 的内存映射和持久化能力

### 异步最佳实践

```python
import asyncio

# ✅ 并发执行独立任务
async def main():
    # 并发（同时执行）
    results = await asyncio.gather(
        fetch_users(),
        fetch_products(),
        fetch_orders(),
    )

    # 有超时限制
    try:
        result = await asyncio.wait_for(slow_operation(), timeout=5.0)
    except asyncio.TimeoutError:
        logger.warning("操作超时")

# ✅ 异步上下文管理器
async def process():
    async with aiofiles.open("data.txt") as f:
        content = await f.read()
```

---

## 错误处理

### 定义业务异常层次

```python
# ✅ 创建应用专属异常类
class AppError(Exception):
    """应用基础异常。"""

class ValidationError(AppError):
    """数据验证失败。"""
    def __init__(self, field: str, message: str) -> None:
        self.field = field
        super().__init__(f"字段 '{field}' 验证失败: {message}")

class NotFoundError(AppError):
    """资源不存在。"""
    def __init__(self, resource: str, id: int | str) -> None:
        super().__init__(f"{resource} (id={id}) 不存在")
```

### 优雅地处理异常链

```python
# ✅ 保留原始异常上下文
try:
    data = json.loads(raw_content)
except json.JSONDecodeError as e:
    raise ValidationError("body", "JSON 格式无效") from e

# ✅ 明确抑制异常链（有意为之时）
try:
    return cache[key]
except KeyError:
    return compute_value(key)  # KeyError 不是真正的错误
```

---

## 路径与文件操作

```python
from pathlib import Path

# ✅ 使用 pathlib 替代 os.path
base_dir = Path(__file__).parent
config_path = base_dir / "config" / "settings.toml"

# 检查并创建目录
output_dir = Path("output")
output_dir.mkdir(parents=True, exist_ok=True)

# 遍历文件
for py_file in base_dir.rglob("*.py"):
    print(py_file.relative_to(base_dir))

# 读写文件
content = config_path.read_text(encoding="utf-8")
config_path.write_text(updated_content, encoding="utf-8")
```

---

## 类型安全

### 使用 Protocol 替代抽象基类

```python
from typing import Protocol, runtime_checkable

# ✅ 结构化子类型（鸭子类型的类型安全版）
@runtime_checkable
class Serializable(Protocol):
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "Serializable": ...

def save(obj: Serializable) -> None:
    data = obj.to_dict()
    # 任何实现 to_dict/from_dict 的类都可以传入
```

### TypedDict 为字典添加类型

```python
from typing import TypedDict

class UserData(TypedDict):
    id: int
    name: str
    email: str

class UserDataPartial(TypedDict, total=False):
    name: str
    email: str

def update_user(user_id: int, updates: UserDataPartial) -> None:
    ...
```

---

## 常见陷阱规避

### 可变默认参数

```python
# ❌ 危险！所有调用共享同一个列表
def add_item(item, container=[]):
    container.append(item)
    return container

# ✅ 使用 None 作为默认值
def add_item(item: str, container: list[str] | None = None) -> list[str]:
    if container is None:
        container = []
    container.append(item)
    return container
```

### 闭包中的循环变量

```python
# ❌ 所有函数都引用同一个 i（循环结束后 i=9）
funcs = [lambda: i for i in range(10)]

# ✅ 通过默认参数捕获当前值
funcs = [lambda i=i: i for i in range(10)]
```

### 整数缓存范围

```python
# ⚠️ Python 只缓存 -5 到 256 的小整数
a = 256
b = 256
assert a is b  # True（缓存）

a = 257
b = 257
assert a == b  # True（值相等）
assert a is not b  # True（不是同一对象）

# ✅ 比较值用 ==，比较身份用 is（仅用于 None/True/False）
```

### 字符串驻留

```python
# ✅ 与 None 比较使用 is
value = get_optional_value()
if value is None:
    ...

# ✅ 类型检查
if isinstance(value, str):
    ...
```

---

## 代码组织

### 模块级常量放顶部

```python
# ✅ 模块常量与配置
MAX_RETRIES = 3
DEFAULT_TIMEOUT = 30.0
SUPPORTED_FORMATS = frozenset({"json", "csv", "parquet"})
```

### 避免循环导入

```python
# ✅ 在函数内部导入（打破循环依赖）
def get_service():
    from mypackage.services import UserService  # 延迟导入
    return UserService()

# ✅ 使用 TYPE_CHECKING 仅在类型检查时导入
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from mypackage.models import User
```

### 公共 API 设计

```python
# ✅ 明确声明公共接口
__all__ = ["PublicClass", "public_function"]

def public_function() -> None:
    """公开函数。"""
    _internal_helper()

def _internal_helper() -> None:
    """内部实现，不对外暴露。"""
    ...
```

### 公共 API 内部调用流程图

当公共 API 入口函数内部涉及多个私有辅助函数、嵌套循环或多阶段流水线时，**必须**在模块顶部 docstring 中绘制 ASCII 调用流程图，并用 `# ═══ §N ═══` 分节注释呼应流程图。详见 [reference/public-api-flow-diagram.md](reference/public-api-flow-diagram.md)。
