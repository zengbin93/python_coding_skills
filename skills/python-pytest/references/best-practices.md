# Pytest 最佳实践和规范

## 测试原则

### FIRST 原则

- **F**ast - 测试应该快速运行
- **I**ndependent - 测试应该相互独立
- **R**epeatable - 测试应该可重复
- **S**elf-Validating - 测试应该有明确的通过/失败结果
- **T**imely - 测试应该及时编写

### AAA 模式

每个测试应该遵循 Arrange-Act-Assert 模式：

```python
def test_user_authentication():
    # Arrange（准备）
    user = User(username="test", password="password123")
    db.save(user)

    # Act（执行）
    result = authenticate("test", "password123")

    # Assert（断言）
    assert result.is_success
    assert result.user.id == user.id
```

### Given-When-Then 模式

```python
def test_order_fulfillment():
    # Given（给定条件）
    product = Product(name="Widget", stock=10)
    order = Order(items=[OrderItem(product, quantity=5)])

    # When（执行操作）
    order.fulfill()

    # Then（验证结果）
    assert order.status == "fulfilled"
    assert product.stock == 5
```

## 测试覆盖率

### 覆盖率目标

| 测试类型 | 最低覆盖率 | 目标覆盖率 |
|---------|----------|-----------|
| 单元测试 | 70% | 90% |
| 集成测试 | 60% | 80% |
| 端到端测试 | 50% | 70% |

### 配置覆盖率

```toml
# pyproject.toml
[tool.coverage.run]
source = ["src"]
omit = [
    "*/tests/*",
    "*/test_*.py",
    "*/__init__.py",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
    "class .*\\bProtocol\\):",
    "@(abc\\.)?abstractmethod",
]
```

### 运行覆盖率测试

```bash
# 运行带覆盖率的测试
pytest --cov=src --cov-report=html

# 设置最低覆盖率要求
pytest --cov=src --cov-fail-under=80

# 生成详细报告
pytest --cov=src --cov-report=term-missing
```

## 测试组织规范

### 目录结构

```
project/
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── module.py
│       └── utils.py
└── tests/
    ├── conftest.py              # 共享fixtures
    ├── unit/                    # 单元测试
    │   ├── test_module.py
    │   └── test_utils.py
    ├── integration/             # 集成测试
    │   └── test_database.py
    └── e2e/                     # 端到端测试
        └── test_api.py
```

### 命名约定

**测试文件**: `test_<module_name>.py`

**测试类**: `Test<ClassName>`

**测试函数**: `test_<function_name>_<scenario>`

```python
# ✅ 好的命名
def test_calculate_sum_positive_numbers():
    pass

def test_calculate_sum_with_negative_numbers():
    pass

def test_calculate_sum_empty_list():
    pass
```

### 测试分层

```python
# 单元测试 - 快速、隔离
class TestUserServiceUnit:
    def test_create_user_valid_data(self):
        pass

    def test_create_user_duplicate_email(self):
        pass

# 集成测试 - 测试组件协作
class TestUserServiceIntegration:
    def test_user_registration_flow(self):
        pass

# 端到端测试 - 测试完整流程
class TestUserAPIToDatabase:
    def test_create_user_via_api(self):
        pass
```

## Fixture 最佳实践

### Fixture 位置

```python
# ✅ 推荐：在 conftest.py 中定义共享 fixtures
# tests/conftest.py
@pytest.fixture
def db_session():
    """所有测试可用的数据库会话"""
    pass

# ✅ 推荐：在测试文件中定义特定 fixtures
# tests/test_users.py
@pytest.fixture
def user_data():
    """用户测试专用数据"""
    pass

# ❌ 避免：在多个地方重复定义相同 fixture
```

### Fixture 命名

```python
# ✅ 清晰的命名
@pytest.fixture
def authenticated_user_token():
    """返回认证用户的令牌"""
    pass

# ✅ 描述性命名
@pytest.fixture
def database_with_test_data():
    """包含测试数据的数据库"""
    pass

# ❌ 模糊的命名
@pytest.fixture
def data():
    """什么样的数据？"""
    pass
```

### Fixture 作用域选择

```python
# Session - 昂贵的资源，测试间不共享状态
@pytest.fixture(scope="session")
def database_container():
    """启动一次，所有测试共享"""
    pass

# Module - 相对昂贵的资源
@pytest.fixture(scope="module")
def database_connection():
    """模块级别共享"""
    pass

# Function - 默认，大多数情况
@pytest.fixture
def test_data():
    """每个测试独立的数据"""
    pass
```

## 断言最佳实践

### 使用有意义的断言

```python
# ✅ 具体、有意义的断言
def test_user_creation():
    user = create_user(username="alice")
    assert user.username == "alice"
    assert user.id is not None
    assert user.is_active is True
    assert user.created_at <= datetime.now()

# ❌ 模糊的断言
def test_user_creation():
    user = create_user(username="alice")
    assert user  # 这个测试什么？
```

### 单一断言原则

```python
# ✅ 每个断言测试一个方面
def test_user_validation():
    with pytest.raises(ValidationError):
        User(username="")  # 空用户名

    with pytest.raises(ValidationError):
        User(username="a" * 100)  # 过长的用户名

# ❌ 一个测试包含太多断言
def test_user_validation():
    user = User(username="test")
    assert user.username == "test"
    assert user.email is not None
    assert user.age > 0
    assert user.is_active
    # ... 更多断言
```

### 使用 pytest.raises

```python
# ✅ 测试异常
def test_division_by_zero():
    with pytest.raises(ZeroDivisionError):
        1 / 0

# ✅ 测试异常消息
def test_validation_error_message():
    with pytest.raises(ValueError, match="must be positive"):
        calculate_square_root(-1)

# ❌ 不使用 pytest.raises
def test_division_by_zero():
    try:
        1 / 0
        assert False  # 失败了但没抛出异常
    except ZeroDivisionError:
        pass  # 通过了
```

## Mock 最佳实践

### 只 Mock 外部依赖

```python
# ✅ 只 Mock 外部 API
def test_process_external_data(mocker):
    # Mock 外部 API 调用
    mocker.patch("app.external_api.call",
                 return_value={"data": "test"})

    result = process_data()
    assert result == "processed: test"

# ❌ Mock 被测试的代码
def test_process_external_data(mocker):
    # 不要 Mock 你要测试的函数！
    mocker.patch("app.process_data", return_value="mocked")
    result = process_data()
    assert result == "mocked"  # 测试了 Mock，不是真实代码
```

### 使用 autospec

```python
# ✅ 使用 autospec 确保签名正确
def test_with_autospec(mocker):
    mocker.patch("app.api.call", autospec=True)
    # 如果调用签名不匹配，会立即失败

# ❌ 不使用 autospec
def test_without_autospec(mocker):
    mocker.patch("app.api.call")
    # 签名错误不会被发现
```

### 验证 Mock 调用

```python
# ✅ 验证 Mock 被正确调用
def test_database_interaction(mocker):
    mock_save = mocker.patch("app.db.save")
    create_user(username="alice")

    mock_save.assert_called_once()
    call_args = mock_save.call_args
    assert call_args[1]["username"] == "alice"

# ❌ 不验证调用
def test_database_interaction(mocker):
    mocker.patch("app.db.save")
    create_user(username="alice")
    # 没有验证是否真的调用了 save
```

## 参数化测试

### 避免重复代码

```python
# ✅ 使用参数化
@pytest.mark.parametrize("input,expected", [
    (2, 4),
    (3, 9),
    (5, 25),
])
def test_square(input, expected):
    assert square(input) == expected

# ❌ 重复的测试
def test_square_2():
    assert square(2) == 4

def test_square_3():
    assert square(3) == 9

def test_square_5():
    assert square(5) == 25
```

### 使用有意义的 IDs

```python
@pytest.mark.parametrize("email,valid", [
    ("user@example.com", True),
    ("invalid", False),
    ("", False),
], ids=["valid_email", "invalid_format", "empty_string"])
def test_email_validation(email, valid):
    assert is_valid_email(email) == valid
```

## 异步测试

### 配置 asyncio 模式

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"  # 推荐使用 auto 模式
```

### 异步 Fixture 作用域

```python
# ✅ 合理使用作用域
@pytest_asyncio.fixture(scope="module")
async def expensive_resource():
    """昂贵的资源，模块级别共享"""
    resource = await create_resource()
    yield resource
    await resource.cleanup()

# ✅ 默认作用域
@pytest_asyncio.fixture
async def simple_resource():
    """简单的资源，每个测试独立"""
    return await create_simple_resource()
```

## 测试标记

### 定义标记

```toml
# pytest.ini 或 pyproject.toml
[tool.pytest.ini_options]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
    "smoke: quick smoke tests",
]
```

### 使用标记

```python
import pytest

@pytest.mark.slow
def test_long_running_operation():
    import time
    time.sleep(10)

@pytest.mark.integration
def test_database_integration():
    pass

@pytest.mark.unit
def test_unit_function():
    pass
```

### 运行特定标记

```bash
# 只运行快速测试
pytest -m "not slow"

# 只运行集成测试
pytest -m integration

# 运行单元或smoke测试
pytest -m "unit or smoke"
```

## 测试数据管理

### 使用工厂模式

```python
# tests/factories.py
class UserFactory:
    @staticmethod
    def create(**kwargs):
        defaults = {
            "username": "test_user",
            "email": "test@example.com",
            "is_active": True,
        }
        defaults.update(kwargs)
        return User(**defaults)

# test file
def test_user_update():
    user = UserFactory.create(username="alice")
    user.update(username="bob")
    assert user.username == "bob"
```

### 使用 pytest-parametrize 和工厂

```python
@pytest.mark.parametrize("factory_func", [
    UserFactory.create,
    AdminFactory.create,
    GuestFactory.create,
])
def test_different_user_types(factory_func):
    user = factory_func()
    assert user.is_valid()
```

## 性能考虑

### 保持测试快速

```python
# ✅ 快速测试 - 使用 Mock
def test_fast_operation(mocker):
    mocker.patch("app.external_api.slow_call")
    result = process_data()
    assert result == "success"

# ❌ 慢速测试 - 真实外部调用
def test_slow_operation():
    result = process_data()  # 调用真实的慢API
    assert result == "success"
```

### 使用标记分离慢速测试

```python
@pytest.mark.slow
def test_expensive_computation():
    # 耗时的计算
    pass

# 运行快速测试
pytest -m "not slow"
```

## 代码质量

### 遵循项目代码规范

```python
# 测试代码也应遵循项目规范
def test_user_service_create_user():
    """测试用户服务创建用户功能"""
    # Arrange
    user_data = {
        "username": "testuser",
        "email": "test@example.com",
    }

    # Act
    user = user_service.create_user(user_data)

    # Assert
    assert user.username == "testuser"
    assert user.email == "test@example.com"
    assert user.id is not None
```

### 测试也需要文档

```python
def test_complex_business_logic():
    """
    测试复杂的业务逻辑：

    场景：用户购买商品时，如果库存不足，应该抛出异常
    期望：抛出 InsufficientStockError 异常
    原因：业务规则不允许超卖
    """
    product = Product(name="Widget", stock=5)

    with pytest.raises(InsufficientStockError):
        product.purchase(quantity=10)
```

## 持续集成

### CI 配置示例

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: [3.9, 3.10, 3.11]

    steps:
    - uses: actions/checkout@v3
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    - name: Install dependencies
      run: |
        pip install -e ".[test]"
    - name: Run tests
      run: |
        pytest --cov=src --cov-report=xml
    - name: Upload coverage
      uses: codecov/codecov-action@v3
```

## 文档和资源

### 相关文档

- [Fixtures 详细指南](fixtures.md)
- [Mocking 和 Patching](mocking.md)
- [异步测试](async-testing.md)
- [测试模式和示例](patterns.md)

### 外部资源

- [Pytest 官方文档](https://docs.pytest.org/)
- [pytest-asyncio 文档](https://pytest-asyncio.readthedocs.io/)
- [pytest-mock 文档](https://pytest-mock.readthedocs.io/)

## 检查清单

在提交代码前，确保：

- [ ] 所有测试通过
- [ ] 新功能有对应的测试
- [ ] 测试覆盖率没有下降
- [ ] 遵循命名约定
- [ ] 使用了适当的标记
- [ ] Mock 外部依赖
- [ ] 验证异常处理
- [ ] 测试边界条件
- [ ] 文档复杂逻辑
