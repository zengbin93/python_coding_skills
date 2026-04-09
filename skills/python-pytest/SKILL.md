---
name: python-pytest
description: >
  使用 pytest 编写高质量测试的通用规范。涵盖单元测试、集成测试（内部流程）、生产环境冒烟测试、
  测试目录三层分离、fixture 使用、Mock 策略、参数化测试、异步测试和覆盖率目标。
  在创建或修改测试文件时使用此 skill。
---

# Python pytest 测试规范

## 核心原则

- 测试应当**快速、独立、可重复、自验证**（FIRST 原则）
- 每个测试函数只验证一个行为
- 测试名称应清晰表达被测行为和预期结果
- 目标测试覆盖率：**核心逻辑 ≥ 80%**

## 测试类型决策树

| 要测试什么 | 推荐方式 |
|-----------|---------|
| 函数/方法的返回值 | 基本单元测试，AAA 模式 |
| 函数的行为（调用、副作用） | Mock 验证函数调用和参数 |
| 数据库/API 等外部依赖 | Fixtures + Mock 外部服务 |
| 异步代码 | `pytest-asyncio`，配置 `asyncio_mode = "auto"` |
| 多种输入组合 | `@pytest.mark.parametrize` 参数化测试 |
| 异常情况 | `pytest.raises` |
| 共享测试数据或资源 | Fixtures（按作用域选择） |

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
    ├── unit/                # 单元测试（快，无外部依赖）
    ├── integration/         # 集成测试（测试环境，连真实依赖）
    └── production/          # 生产冒烟（只读、健康检查）
```

**注意**：`tests/` 目录不需要 `__init__.py` 文件。

按层执行：

```bash
pytest tests/unit/                    # 快速验证，CI 必跑
pytest tests/integration/             # 连真实依赖，较慢
pytest tests/production/ -v           # 发布后冒烟
pytest -m "not production"            # 排除生产测试
```

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

### monkeypatch vs pytest-mock

**monkeypatch** — 简单替换，适合修改环境变量、配置等：
```python
def test_with_monkeypatch(monkeypatch):
    monkeypatch.setenv("API_KEY", "test_key")
    monkeypatch.setattr("module.function", mock_function)
```

**pytest-mock (mocker)** — 强大的 mock 功能，适合验证调用：
```python
def test_with_mocker(mocker):
    mock_func = mocker.patch("module.function")
    mock_func.return_value = "result"
    mock_func.assert_called_with("args")
```

### Patch 策略

✅ **正确**：在使用的地方 patch（patch 被测试代码所引用的路径）
```python
mocker.patch("myapp.utils.helper.function")  # myapp 使用 function
```

❌ **错误**：在定义的地方 patch
```python
from myapp.utils import helper
mocker.patch("helper.function")  # 不会生效
```

### 常见 Mock 场景

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

## 集成测试（内部流程）

集成测试验证多模块串联的完整业务流程，与单元测试的核心区别在于 **Mock 范围**：

| | 单元测试 | 集成测试 |
|---|---|---|
| Mock 范围 | Mock 一切外部依赖 | 只 Mock 真正的外部边界（DB、HTTP、文件） |
| 测试粒度 | 单个函数/方法 | 完整业务流程 |
| 速度 | 极快 | 较慢，可接受 |

### 只 Mock 外部边界，内部逻辑走真实代码

```python
# ✅ 内部流程全部走真实代码，只隔离外部 I/O
def test_factor_compute_pipeline(mocker, sample_data):
    mocker.patch("myapp.data.get_klines", return_value=sample_data)  # 只 Mock 外部边界

    engine = TimeSeriesEngine(data=sample_data, factor=factor)
    result = engine.results

    assert not result.empty
    assert "F#SMA60#DEFAULT" in result.columns
```

### 按流程阶段组织，而非按模块

```python
# tests/integration/test_factor_pipeline.py

def test_factor_compute_returns_valid_output(sample_data):
    """阶段1：因子计算"""
    engine = TimeSeriesEngine(data=sample_data, factor=factor)
    assert not engine.results.empty

def test_factor_eval_ic_is_reasonable(factor_results):
    """阶段2：因子评估"""
    ic = calc_ic(factor_results)
    assert abs(ic) > 0.01

def test_model_weights_sum_to_one(factor_results):
    """阶段3：策略建模"""
    model = CSSorting(factor_results, factor="F#SMA60#DEFAULT", top_pct=0.2)
    weights = model.run()
    assert abs(weights["weight"].sum()) <= 1.0
```

### fixture scope 选择

| 场景 | 推荐 scope |
|------|-----------|
| 生成测试数据（无状态） | `module` 或 `session` |
| 有状态的对象（如引擎实例） | `function` |
| 数据库连接 | `session` |

### 常见坑

- **Mock 太多 → 等于没测**：如果内部多个模块都被 Mock，流程没有真正串联
- **测试间隐式依赖**：测试 B 不能依赖测试 A 的副作用，每个用例应能独立运行

## 生产环境测试原则

pytest 可以做生产环境验证，但必须严格区分场景。

### 可以做的（安全场景）

| 场景 | 说明 |
|------|------|
| 冒烟测试 | 发布后验证服务启动、关键接口可用 |
| 健康检查 | 验证环境变量、证书、依赖服务连通性 |
| 流量回放 | 复制生产流量对比新旧版本，不影响用户 |
| 性能基准 | 验证新版本不比旧版慢、无内存泄漏 |

### 绝对禁止

- 带写操作的用例（建数据、删数据、改状态）→ 污染生产库
- 高并发测试 → 打垮生产服务
- 普通单元测试直接在生产跑（很多单测会清表、Mock 外部依赖）

### 生产冒烟示例

```python
@pytest.mark.production
def test_api_health(client):
    resp = client.get("/health")
    assert resp.status_code == 200
    assert resp.json()["status"] == "ok"

@pytest.mark.production
def test_api_response_time(client):
    import time
    start = time.time()
    client.get("/api/user/info")
    assert time.time() - start < 0.3  # 响应时间断言
```

生产测试强制规则：只读不写、禁止造脏数据、单线程串行、禁止 `truncate`/`drop`。

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

## 工作流程

### 编写新测试的步骤

1. **确定测试类型** — 参考上方决策树，选择单元测试、集成测试或异步测试
2. **创建测试文件** — `test_<module>.py`
3. **编写测试函数** — 遵循 AAA 模式
4. **添加必要的 fixtures** — 复用测试数据和资源
5. **Mock 外部依赖** — 保持测试隔离和快速
6. **验证覆盖率** — `pytest --cov`
7. **运行测试** — `pytest -v`

### 诊断测试失败

1. **查看详细输出** — `pytest -v` 或 `pytest -vv`
2. **打印调试信息** — 在测试中使用 `print()`（提交前删除）
3. **运行单个测试** — `pytest tests/test_file.py::test_function`
4. **查看完整错误栈** — `pytest --tb=long`

## 详细参考文档

本 skill 包含以下详细参考文档（位于 `references/` 目录）：

### [fixtures.md](references/fixtures.md)
Fixtures 完全指南：基础 fixtures、作用域、参数化、依赖关系、conftest.py、内置 fixtures（tmp_path、capsys、caplog、monkeypatch）、Async fixtures

### [mocking.md](references/mocking.md)
Mocking 和 Patching 完全指南：monkeypatch 用法、pytest-mock 使用、Mock 对象配置和验证、常见 mock 模式（API、数据库、文件系统）、Patch 策略、常见陷阱

### [async-testing.md](references/async-testing.md)
异步测试完全指南：pytest-asyncio 配置（auto/strict 模式）、基本异步测试、异步 fixtures、Loop scope 控制、Mock 异步操作

### [patterns.md](references/patterns.md)
常用测试模式和示例：测试组织、参数化测试、异常测试、标记和跳过、测试配置、数据驱动测试、集成测试模式、边界条件测试

### [best-practices.md](references/best-practices.md)
测试最佳实践和规范：FIRST 原则、AAA/Given-When-Then 模式、覆盖率配置、Fixture/Mock/断言最佳实践、性能考虑、CI/CD 集成

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
