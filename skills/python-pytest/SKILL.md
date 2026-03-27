---
name: python-pytest
description: >
  使用 pytest 编写高质量单元测试的通用规范。涵盖测试结构、fixture 使用、断言风格、
  参数化测试、异步测试和覆盖率目标。在创建或修改测试文件时使用此 skill。
---

# Python pytest 测试规范

## 核心原则

- 测试应当**快速、独立、可重复、自验证**（FIRST 原则）
- 每个测试函数只验证一个行为
- 测试名称应清晰表达被测行为和预期结果
- 目标测试覆盖率：**核心逻辑 ≥ 80%**

## 目录结构

```
project/
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── calculator.py
│       └── user_service.py
└── tests/
    ├── conftest.py          # 共享 fixture
    ├── test_calculator.py
    └── test_user_service.py
```

**注意**：`tests/` 目录不需要 `__init__.py` 文件。

## 测试文件命名

- 测试文件以 `test_` 开头：`test_calculator.py`
- 测试函数以 `test_` 开头：`def test_add_returns_correct_sum()`
- 辅助文件不使用 `test_` 前缀：`helpers.py`、`factories.py`

## 测试函数命名规范

遵循 `test_<功能>_<场景>_<预期结果>` 模式：

```python
# ✅ 清晰的命名
def test_divide_by_zero_raises_value_error():
def test_create_user_with_valid_data_returns_user():
def test_login_with_invalid_password_returns_401():

# ❌ 模糊的命名
def test_divide():
def test_user():
def test_login2():
```

## 基本测试结构（AAA 模式）

```python
def test_calculate_discount_returns_correct_price():
    # Arrange（准备）
    price = 100.0
    rate = 0.2

    # Act（执行）
    result = calculate_discount(price, rate)

    # Assert（验证）
    assert result == 80.0
```

## 使用 pytest fixture

### 定义 fixture

```python
# tests/conftest.py
import pytest
from mypackage.database import Database

@pytest.fixture
def db():
    """提供测试数据库连接，测试结束后自动清理。"""
    database = Database(":memory:")
    database.create_tables()
    yield database
    database.close()

@pytest.fixture
def sample_user():
    return {"id": 1, "name": "张三", "email": "zhang@example.com"}
```

### 使用 fixture

```python
def test_save_user_persists_to_database(db, sample_user):
    db.save_user(sample_user)
    retrieved = db.get_user(sample_user["id"])
    assert retrieved["name"] == sample_user["name"]
```

### fixture 作用域

```python
@pytest.fixture(scope="session")   # 整个测试会话共享一次
@pytest.fixture(scope="module")    # 每个模块共享一次
@pytest.fixture(scope="class")     # 每个测试类共享一次
@pytest.fixture(scope="function")  # 默认，每个测试函数独立
```

## 参数化测试

使用 `@pytest.mark.parametrize` 避免重复测试代码：

```python
import pytest

@pytest.mark.parametrize("price,rate,expected", [
    (100.0, 0.0, 100.0),   # 无折扣
    (100.0, 0.5, 50.0),    # 五折
    (100.0, 1.0, 0.0),     # 全额折扣
    (0.0, 0.3, 0.0),       # 零价格
])
def test_calculate_discount(price, rate, expected):
    assert calculate_discount(price, rate) == expected


@pytest.mark.parametrize("invalid_rate", [-0.1, 1.1, 2.0])
def test_calculate_discount_with_invalid_rate_raises(invalid_rate):
    with pytest.raises(ValueError, match="折扣率必须在"):
        calculate_discount(100.0, invalid_rate)
```

## 异常测试

```python
def test_divide_by_zero_raises_zero_division_error():
    with pytest.raises(ZeroDivisionError):
        divide(10, 0)

# 验证异常消息
def test_invalid_email_raises_with_message():
    with pytest.raises(ValueError, match="邮箱格式不正确"):
        validate_email("not-an-email")

# 捕获并检查异常对象
def test_custom_exception_has_correct_code():
    with pytest.raises(AppError) as exc_info:
        raise_app_error(code=404)
    assert exc_info.value.code == 404
```

## Mock 和 Patch

```python
from unittest.mock import MagicMock, patch

def test_send_email_calls_smtp_once():
    with patch("mypackage.email.smtplib.SMTP") as mock_smtp:
        mock_instance = mock_smtp.return_value.__enter__.return_value
        send_welcome_email("user@example.com")
        mock_instance.sendmail.assert_called_once()


def test_get_user_calls_api_with_correct_id():
    mock_client = MagicMock()
    mock_client.get.return_value = {"id": 1, "name": "李四"}

    service = UserService(client=mock_client)
    user = service.get_user(1)

    mock_client.get.assert_called_once_with("/users/1")
    assert user["name"] == "李四"
```

## 异步测试

```python
import pytest

# pytest.ini 或 pyproject.toml 中配置：
# [tool.pytest.ini_options]
# asyncio_mode = "auto"

async def test_async_fetch_returns_data():
    result = await fetch_data("https://api.example.com/data")
    assert result is not None


async def test_async_operation_with_fixture(db):
    await db.async_save({"key": "value"})
    result = await db.async_get("key")
    assert result == "value"
```

## 标记（Markers）

```python
# 标记慢速测试
@pytest.mark.slow
def test_large_data_processing():
    ...

# 标记需要网络的测试
@pytest.mark.network
def test_external_api_integration():
    ...

# 跳过测试
@pytest.mark.skip(reason="功能尚未实现")
def test_future_feature():
    ...

# 条件跳过
@pytest.mark.skipif(sys.platform == "win32", reason="不支持 Windows")
def test_unix_only_feature():
    ...
```

在 `pyproject.toml` 中注册自定义 marker：

```toml
[tool.pytest.ini_options]
markers = [
    "slow: 标记为慢速测试，可用 -m 'not slow' 跳过",
    "network: 需要网络连接的集成测试",
]
```

## pytest 配置（pyproject.toml）

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
addopts = "-v --tb=short"

[tool.coverage.run]
source = ["src"]
omit = ["tests/*", "**/__init__.py"]

[tool.coverage.report]
fail_under = 80
show_missing = true
```

## 运行命令

```bash
# 运行所有测试
pytest

# 运行指定文件
pytest tests/test_calculator.py

# 运行指定测试函数
pytest tests/test_calculator.py::test_add_returns_correct_sum

# 按标记运行
pytest -m slow
pytest -m "not slow"

# 带覆盖率报告
pytest --cov=src --cov-report=term-missing

# 失败时立即停止
pytest -x

# 并行执行（需安装 pytest-xdist）
pytest -n auto
```

## 最佳实践

- **只运行相关测试**，而非整个测试套件；按模块或功能范围执行，节省时间
- **编写新测试前先阅读现有测试**，理解项目的测试风格和惯用模式，保持一致性
- **用 `print` 辅助调试，完成后及时删除**；不要将调试输出留在最终提交的代码中
- **提交前解决所有错误和警告**；通过 ruff、basedpyright 检查后再提交

## 测试质量清单

在提交测试代码前确认：

- [ ] 测试名称清晰描述被测行为
- [ ] 遵循 AAA（Arrange-Act-Assert）模式
- [ ] 没有直接修改全局状态
- [ ] Mock 只针对外部依赖（数据库、API、文件系统）
- [ ] 没有硬编码的路径或环境依赖
- [ ] 异常场景有对应测试
- [ ] 边界值（空列表、零、None 等）有对应测试
- [ ] 已删除所有调试用 `print` 语句
- [ ] ruff 和 basedpyright 检查无报错
