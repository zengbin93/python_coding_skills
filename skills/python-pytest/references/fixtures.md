# Pytest Fixtures 完全指南

## 目录
- [基础 Fixtures](#基础-fixtures)
- [Fixture 作用域](#fixture-作用域)
- [Fixture 参数化](#fixture-参数化)
- [Fixture 依赖](#fixture-依赖)
- [使用 conftest.py](#使用-conftestpy)
- [内置 Fixtures](#内置-fixtures)
- [Async Fixtures](#async-fixtures)

## 基础 Fixtures

### 创建简单 Fixture

```python
import pytest

@pytest.fixture
def sample_data():
    """返回测试数据"""
    return {"key": "value", "number": 42}

def test_with_fixture(sample_data):
    assert sample_data["key"] == "value"
    assert sample_data["number"] == 42
```

### 带Setup和Teardown的Fixture

```python
@pytest.fixture
def database():
    """Setup: 创建数据库连接"""
    db = Database.connect(":memory:")
    db.create_tables()

    yield db  # 提供给测试使用

    # Teardown: 清理
    db.close()
```

## Fixture 作用域

### Function作用域（默认）

每个测试函数都创建新的fixture实例：

```python
@pytest.fixture(scope="function")
def temp_file():
    with tempfile.NamedTemporaryFile() as f:
        yield f.name
    # 文件自动删除
```

### Class作用域

整个测试类共享一个fixture实例：

```python
@pytest.fixture(scope="class")
def shared_resource():
    resource = ExpensiveResource()
    yield resource
    resource.cleanup()

class TestResource:
    def test_method1(self, shared_resource):
        # shared_resource 在所有测试方法间共享
        pass

    def test_method2(self, shared_resource):
        # 相同的 shared_resource 实例
        pass
```

### Module作用域

整个模块共享一个fixture实例：

```python
@pytest.fixture(scope="module")
def module_cache():
    return Cache()

def test_1(module_cache):
    module_cache.set("key", "value")

def test_2(module_cache):
    # key 仍然存在
    assert module_cache.get("key") == "value"
```

### Session作用域

整个测试会话共享一个fixture：

```python
@pytest.fixture(scope="session")
def global_config():
    return load_config("config.yaml")
```

## Fixture 参数化

### 基本参数化

```python
@pytest.fixture(params=[1, 2, 3])
def number(request):
    return request.param

def test_with_parametrized_fixture(number):
    # 测试会运行3次，number 分别为 1, 2, 3
    assert number > 0
```

### 参数化与IDs

```python
@pytest.fixture(params=[
    ("valid@email.com", True),
    ("invalid", False),
    ("", False)
], ids=["valid_email", "invalid_email", "empty_email"])
def email_case(request):
    return request.param

def test_email_validation(email_case):
    email, expected = email_case
    assert is_valid_email(email) == expected
```

### 组合参数化

```python
@pytest.fixture(params=["mysql", "postgresql", "sqlite"])
def db_type(request):
    return request.param

@pytest.fixture(params=["read", "write"])
def operation(request):
    return request.param

def test_database_operations(db_type, operation):
    # 测试所有数据库类型和操作的组合
    # 3种数据库 × 2种操作 = 6个测试
    pass
```

## Fixture 依赖

### 一个Fixture依赖另一个

```python
@pytest.fixture
def config():
    return {"host": "localhost", "port": 5432}

@pytest.fixture
def database(config):
    # 依赖 config fixture
    return Database.connect(config["host"], config["port"])

def test_database_query(database):
    # database 会自动注入 config
    result = database.query("SELECT 1")
    assert result == 1
```

### 多个依赖

```python
@pytest.fixture
def auth_token():
    return "secret_token"

@pytest.fixture
def user_id():
    return 12345

@pytest.fixture
def authenticated_client(auth_token, user_id):
    return Client(token=auth_token, user=user_id)
```

## 使用 conftest.py

### 共享Fixtures

`conftest.py` 中的fixtures可以被其目录及子目录中的所有测试使用：

```python
# tests/conftest.py
@pytest.fixture
def test_client():
    return TestClient()

@pytest.fixture
def clean_database():
    db = get_test_db()
    db.cleanup()
    yield db
    db.close()
```

### 目录结构示例

```
tests/
├── conftest.py          # 全局 fixtures
├── test_auth.py         # 使用全局 fixtures
├── test_users.py        # 使用全局 fixtures
└── api/
    ├── conftest.py      # API 特定的 fixtures
    └── test_endpoints.py
```

### Fixtures 查找顺序

1. 测试文件所在的目录
2. 父目录
3. 直到项目根目录

## 内置 Fixtures

### tmp_path - 临时目录

```python
def test_with_temp_dir(tmp_path):
    # tmp_path 是一个 pathlib.Path 对象
    file_path = tmp_path / "test.txt"
    file_path.write_text("content")
    assert file_path.read_text() == "content"
    # 测试结束后自动清理
```

### tmpdir - 临时目录（py.path.local）

```python
def test_with_tmpdir(tmpdir):
    # tmpdir 是 py.path.local 对象
    file = tmpdir.join("test.txt")
    file.write("content")
    assert file.read() == "content"
```

### capsys - 捕获输出

```python
def test_print_output(capsys):
    print("Hello, World!")
    captured = capsys.readouterr()
    assert captured.out == "Hello, World!\n"
    assert captured.err == ""
```

### caplog - 捕获日志

```python
import logging

def test_logging(caplog):
    caplog.set_level(logging.INFO)
    logging.getLogger().info("Test message")

    assert "Test message" in caplog.text
    assert caplog.records[0].levelname == "INFO"
```

### monkeypatch - 打补丁

```python
def test_with_monkeypatch(monkeypatch):
    # 替换环境变量
    monkeypatch.setenv("API_KEY", "test_key")

    # 替换函数
    def mock_function():
        return "mocked"
    monkeypatch.setattr("module.function", mock_function)

    # 替换对象属性
    monkeypatch.setattr(SomeClass, "attribute", "new_value")
```

## Async Fixtures

### 基本Async Fixture

```python
import pytest_asyncio

@pytest_asyncio.fixture
async def async_client():
    client = AsyncClient()
    await client.connect()
    yield client
    await client.close()
```

### Async Fixture作用域

```python
@pytest_asyncio.fixture(scope="module")
async def module_async_resource():
    resource = await AsyncResource.create()
    yield resource
    await resource.cleanup()
```

### Loop Scope控制

```python
# 使用 session 的事件循环，但每次测试都创建新实例
@pytest_asyncio.fixture(loop_scope="session")
async def session_loop_function_scoped():
    return await setup_resource()

# 使用 module 的事件循环，缓存在 module 级别
@pytest_asyncio.fixture(loop_scope="module", scope="module")
async def module_loop_module_scoped():
    resource = await create_expensive_resource()
    yield resource
    await resource.cleanup()
```

### 混合同步和异步

```python
@pytest.fixture
def sync_config():
    return {"async_mode": True}

@pytest_asyncio.fixture
async def async_service(sync_config):
    service = await AsyncService.create(sync_config)
    yield service
    await service.shutdown()
```

## 最佳实践

1. **命名清晰**: fixture名称应该描述它提供的内容
2. **合理使用作用域**: 避免过度使用session作用域
3. **及时清理**: 使用yield后确保清理资源
4. **使用conftest.py**: 共享通用fixtures
5. **组合fixtures**: 通过依赖组合简单fixtures创建复杂资源
