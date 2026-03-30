# Pytest 常用测试模式和示例

## 目录
- [测试组织](#测试组织)
- [参数化测试](#参数化测试)
- [异常测试](#异常测试)
- [标记和跳过](#标记和跳过)
- [测试配置](#测试配置)
- [常用模式](#常用模式)

## 测试组织

### 文件和目录结构

```
tests/
├── conftest.py                 # 全局fixtures和配置
├── test_utils.py               # 工具函数测试
├── test_services.py            # 服务层测试
├── test_api.py                 # API测试
└── unit/                       # 单元测试
    └── test_models.py
```

### 测试命名约定

```python
# ✅ 好的命名
def test_user_login_success():
    """测试用户登录成功"""
    pass

def test_user_login_with_invalid_credentials():
    """测试用户登录失败（无效凭证）"""
    pass

# ❌ 不好的命名
def test_login():
    """不明确测试什么场景"""
    pass

def test1():
    """毫无意义的命名"""
    pass
```

### 测试类组织

```python
class TestUserService:
    """用户服务相关测试"""

    def test_create_user_success(self):
        pass

    def test_create_user_duplicate_email(self):
        pass

    def test_delete_user_not_found(self):
        pass
```

## 参数化测试

### 基本参数化

```python
@pytest.mark.parametrize("input,expected", [
    (1, 2),
    (2, 4),
    (3, 6),
])
def test_multiply_by_two(input, expected):
    assert input * 2 == expected
```

### 参数化与IDs

```python
@pytest.mark.parametrize("email,valid", [
    ("user@example.com", True),
    ("invalid", False),
    ("", False),
    ("@", False),
], ids=["valid_email", "invalid_format", "empty_string", "at_symbol_only"])
def test_email_validation(email, valid):
    assert is_valid_email(email) == valid
```

### 多参数参数化

```python
@pytest.mark.parametrize("x,y,expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
])
def test_addition(x, y, expected):
    assert add(x, y) == expected
```

### 参数化Classes

```python
@pytest.mark.parametrize("cls", [
    dict,
    list,
    set,
])
def test_common_iterable_methods(cls):
    instance = cls()
    assert hasattr(instance, '__len__')
```

### 参数化与Fixtures组合

```python
@pytest.fixture
def db_connection():
    return Database.connect(":memory:")

@pytest.mark.parametrize("data", [
    {"name": "Alice", "age": 30},
    {"name": "Bob", "age": 25},
    {"name": "Charlie", "age": 35},
])
def test_insert_user(db_connection, data):
    db_connection.insert("users", data)
    result = db_connection.get("users", name=data["name"])
    assert result == data
```

### 参数化测试用例

```python
test_cases = [
    # (input, expected, description)
    ("hello", "HELLO", "lowercase to uppercase"),
    ("WORLD", "WORLD", "already uppercase"),
    ("PyTest", "PYTEST", "mixed case"),
]

@pytest.mark.parametrize("input_str,expected,description", test_cases)
def test_to_upper(input_str, expected, description):
    assert to_upper(input_str) == expected
```

## 异常测试

### 测试异常抛出

```python
def test_zero_division():
    with pytest.raises(ZeroDivisionError):
        1 / 0
```

### 测试异常消息

```python
def test_value_error_message():
    with pytest.raises(ValueError, match="must be positive"):
        calculate_square_root(-1)
```

### 测试异常属性

```python
def test_custom_exception():
    with pytest.raises(CustomError) as exc_info:
        raise CustomError("Error message", code=42)

    assert exc_info.value.code == 42
    assert "Error message" in str(exc_info.value)
```

### 测试不抛出异常

```python
def test_no_exception():
    # 如果抛出异常，测试失败
    with pytest.raises(Exception):
        risky_operation()

    # 使用no_raise上下文（pytest-check插件）
    with pytest.no_exception():
        safe_operation()
```

### 测试特定异常类型

```python
@pytest.mark.parametrize("input,exception_type", [
    (None, AttributeError),
    ("not_a_number", ValueError),
    (-1, ValueError),
])
def test_validation_errors(input, exception_type):
    with pytest.raises(exception_type):
        validate_number(input)
```

## 标记和跳过

### 标记测试

```python
@pytest.mark.slow
def test_slow_operation():
    import time
    time.sleep(5)

@pytest.mark.integration
def test_database_integration():
    pass

@pytest.mark.unit
def test_unit_function():
    pass
```

### 运行特定标记的测试

```bash
# 只运行slow测试
pytest -m slow

# 不运行slow测试
pytest -m "not slow"

# 运行unit或integration
pytest -m "unit or integration"

# 运行integration且不是slow
pytest -m "integration and not slow"
```

### 跳过测试

```python
@pytest.mark.skip(reason="Not implemented yet")
def test_not_ready():
    pass

@pytest.mark.skipif(sys.version_info < (3, 8),
                    reason="Requires Python 3.8+")
def test_python38_feature():
    pass

@pytest.mark.skipif(not os.getenv("RUN_SLOW_TESTS"),
                    reason="Set RUN_SLOW_TESTS to run")
def test_slow():
    import time
    time.sleep(10)
```

### 条件跳过

```python
def test_if_platform_is_windows():
    if sys.platform != "win32":
        pytest.skip("Only runs on Windows")
    # 测试代码...

def test_if_module_available():
    pytest.importorskip("optional_module")
    # 测试代码...
```

### 预期失败

```python
@pytest.mark.xfail
def test_known_bug():
    """已知bug，预期会失败"""
    assert buggy_function() == expected

@pytest.mark.xfail(reason="Bug #123")
def test_specific_bug():
    pass

@pytest.mark.xfail(raises=ValueError)
def test_known_exception():
    """预期抛出特定异常"""
    might_raise_value_error()
```

## 测试配置

### pytest.ini

```ini
[pytest]
# 测试目录
testpaths = tests

# 测试文件模式
python_files = test_*.py

# 测试类模式
python_classes = Test*

# 测试函数模式
python_functions = test_*

# 默认标记
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
    unit: marks tests as unit tests

# 最小Python版本
minversion = 7.0

# 命令行选项
addopts =
    -v
    --strict-markers
    --tb=short
```

### pyproject.toml

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]
addopts = "-v --strict-markers --tb=short"
asyncio_mode = "auto"
```

## 常用模式

### Setup/Teardown模式

```python
class TestWithSetup:
    def setup_method(self):
        """每个测试方法前执行"""
        self.resource = Resource()

    def teardown_method(self):
        """每个测试方法后执行"""
        self.resource.cleanup()

    def setup_class(cls):
        """整个测试类前执行一次"""
        cls.shared_resource = SharedResource()

    def teardown_class(cls):
        """整个测试类后执行一次"""
        cls.shared_resource.cleanup()
```

### 数据驱动测试

```python
# 从文件读取测试数据
@pytest.fixture
def test_data():
    with open("test_cases.json") as f:
        return json.load(f)

@pytest.mark.parametrize("case", test_data()["cases"])
def test_with_data(case):
    result = process(case["input"])
    assert result == case["expected"]
```

### Mock外部依赖

```python
def test_with_mock(mocker):
    # Mock API调用
    mock_api = mocker.patch("module.api_call")
    mock_api.return_value = {"status": "success"}

    result = module.my_function()

    assert result["status"] == "success"
    mock_api.assert_called_once()
```

### 测试私有方法

```python
def test_private_method():
    """测试私有方法（必要时）"""
    from mymodule import MyClass
    obj = MyClass()

    # 直接测试私有方法
    result = obj._private_method("input")
    assert result == "expected"
```

### 测试生成器

```python
def test_generator():
    """测试生成器函数"""
    gen = my_generator()

    # 测试前几个值
    assert next(gen) == 1
    assert next(gen) == 2

    # 转换为列表测试
    gen = my_generator()
    assert list(gen) == [1, 2, 3, 4, 5]
```

### 测试上下文管理器

```python
def test_context_manager():
    """测试上下文管理器"""
    with MyContextManager() as context:
        assert context.is_active
        assert context.value == "expected"

    # 退出后验证清理
    assert not context.is_active
```

### 比较测试

```python
@pytest.mark.parametrize("implementation", [
    implementation_v1,
    implementation_v2,
    implementation_v3,
])
def test_all_implementations(implementation):
    """确保所有实现产生相同结果"""
    result = implementation("input")
    assert result == "expected_output"
```

### 边界条件测试

```python
@pytest.mark.parametrize("value", [
    0,           # 零
    -1,          # 负数
    1,           # 最小正数
    999999,      # 大数
    None,        # None
    "",          # 空字符串
    " ",         # 空白
])
def test_edge_cases(value):
    """测试各种边界条件"""
    result = process(value)
    assert result is not None
```

### 性能基准测试

```python
def test_performance_benchmark():
    """简单性能测试"""
    import time

    start_time = time.time()
    result = expensive_operation()
    elapsed = time.time() - start_time

    assert result is not None
    assert elapsed < 1.0  # 应在1秒内完成
```

### 集成测试模式

```python
@pytest.mark.integration
class TestDatabaseIntegration:
    """数据库集成测试"""

    @pytest.fixture(autouse=True)
    def setup_database(self):
        """自动使用的fixture"""
        self.db = TestDatabase()
        self.db.create_tables()
        yield
        self.db.cleanup()

    def test_user_crud(self):
        # 创建
        user = self.db.create_user(name="Alice")
        assert user.id is not None

        # 读取
        fetched = self.db.get_user(user.id)
        assert fetched.name == "Alice"

        # 更新
        self.db.update_user(user.id, name="Bob")
        updated = self.db.get_user(user.id)
        assert updated.name == "Bob"

        # 删除
        self.db.delete_user(user.id)
        deleted = self.db.get_user(user.id)
        assert deleted is None
```

### 快速失败模式

```python
@pytest.mark.fast
def test_quick_check():
    """快速验证核心功能"""
    assert core_function() is not None

@pytest.mark.slow
def test_comprehensive():
    """全面的测试（可能较慢）"""
    # 详细的测试逻辑...
    pass
```

### 测试变体

```python
# 使用不同的配置测试相同功能
@pytest.mark.parametrize("config_type", [
    "minimal",
    "default",
    "full",
])
def test_with_different_configs(config_type):
    config = load_config(config_type)
    result = process_with_config(config)
    assert result is not None
```

## 最佳实践

1. **一个测试验证一件事** - 保持测试简单专注
2. **使用描述性名称** - 测试名称应该说明它测试什么
3. **测试应该独立** - 不依赖执行顺序或其他测试
4. **使用fixtures** - 复用setup/teardown逻辑
5. **参数化测试** - 避免重复的测试代码
6. **合理使用标记** - 帮助组织和运行测试
7. **Mock外部依赖** - 保持测试快速和可靠
8. **测试边界条件** - 不要只测试正常路径
9. **保持测试快速** - 慢速测试标记为@slow
10. **定期运行测试** - 集成到CI/CD流程
