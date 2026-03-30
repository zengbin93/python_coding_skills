# Pytest 异步测试完全指南

## 目录
- [pytest-asyncio 配置](#pytest-asyncio-配置)
- [基本异步测试](#基本异步测试)
- [异步 Fixtures](#异步-fixtures)
- [测试模式](#测试模式)
- [常见场景](#常见场景)
- [最佳实践](#最佳实践)

## pytest-asyncio 配置

### 安装

```bash
pip install pytest-asyncio
```

### 配置文件设置

**pyproject.toml**:
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

**pytest.ini**:
```ini
[pytest]
asyncio_mode = auto
```

**setup.cfg**:
```ini
[tool:pytest]
asyncio_mode = auto
```

### 测试模式

#### Auto模式（推荐）

```toml
asyncio_mode = "auto"
```

- 自动为所有async测试函数添加 `@pytest.mark.asyncio`
- 自动接管所有async fixtures
- 最简单的配置
- 适用于只使用asyncio的项目

#### Strict模式

```toml
asyncio_mode = "strict"
```

- 需要显式使用 `@pytest.mark.asyncio` 装饰器
- async fixtures需要使用 `@pytest_asyncio.fixture`
- 适用于使用多种异步库的项目（asyncio + trio等）

## 基本异步测试

### Auto模式下的异步测试

```python
# pytest.ini: asyncio_mode = auto

async def test_basic_async():
    """在auto模式下，自动标记为asyncio测试"""
    result = await async_function()
    assert result == "expected"
```

### Strict模式下的异步测试

```python
import pytest

@pytest.mark.asyncio
async def test_marked_async():
    """在strict模式下，需要显式标记"""
    result = await async_function()
    assert result == "expected"
```

### 异步断言

```python
async def test_async_assertions():
    # 异步操作后断言
    value = await fetch_value()
    assert value is not None
    assert value > 0

    # 异步上下文管理器
    async with AsyncResource() as resource:
        assert resource.is_ready

    # 多个异步操作
    results = await asyncio.gather(
        operation1(),
        operation2(),
        operation3()
    )
    assert len(results) == 3
```

## 异步 Fixtures

### 基本异步Fixture

```python
import pytest_asyncio

@pytest_asyncio.fixture
async def async_client():
    """创建异步客户端"""
    client = AsyncClient()
    await client.connect()
    yield client
    await client.close()

async def test_with_client(async_client):
    result = await async_client.fetch_data()
    assert result is not None
```

### 异步Fixture作用域

```python
@pytest_asyncio.fixture(scope="function")
async def function_scoped():
    """每个测试函数创建新实例"""
    return await AsyncResource.create()

@pytest_asyncio.fixture(scope="module")
async def module_scoped():
    """模块级别共享"""
    resource = await AsyncResource.create()
    yield resource
    await resource.cleanup()

@pytest_asyncio.fixture(scope="session")
async def session_scoped():
    """会话级别共享"""
    resource = await AsyncResource.create()
    yield resource
    await resource.cleanup()
```

### Loop Scope控制

```python
# Function-scoped loop（默认）
@pytest_asyncio.fixture(loop_scope="function")
async def default_loop():
    """每个测试使用新的事件循环"""
    return await setup()

# Module-scoped loop
@pytest_asyncio.fixture(loop_scope="module")
async def module_loop():
    """整个模块共享一个事件循环"""
    return await setup()

# Session-scoped loop
@pytest_asyncio.fixture(loop_scope="session")
async def session_loop():
    """整个测试会话共享一个事件循环"""
    return await setup()
```

### 组合作用域和Loop Scope

```python
# Module作用域，Module loop
@pytest_asyncio.fixture(scope="module", loop_scope="module")
async def module_scoped_with_module_loop():
    """缓存在module级别，使用module的事件循环"""
    resource = await ExpensiveResource.create()
    yield resource
    await resource.cleanup()

# Function作用域，Module loop
@pytest_asyncio.fixture(loop_scope="module")
async def function_scoped_with_module_loop():
    """每次创建新实例，但使用module的事件循环"""
    return await setup()
```

### 依赖其他Fixture的异步Fixture

```python
@pytest.fixture
def config():
    return {"host": "localhost", "port": 8080}

@pytest_asyncio.fixture
async def async_service(config):
    """依赖同步fixture"""
    service = await AsyncService.create(config["host"], config["port"])
    yield service
    await service.shutdown()
```

### 参数化异步Fixture

```python
@pytest_asyncio.fixture(params=["mysql", "postgresql", "sqlite"])
async def async_database(request):
    """参数化的异步数据库fixture"""
    db = await connect_to_database(request.param)
    yield db
    await db.close()

async def test_with_parametrized_db(async_database):
    # 对每个数据库类型运行测试
    result = await async_database.query("SELECT 1")
    assert result == 1
```

## 测试模式

### 测试异步函数

```python
async def test_async_function():
    """测试纯异步函数"""
    result = await async_add(1, 2)
    assert result == 3
```

### 测试异步方法

```python
async def test_async_method():
    """测试异步类方法"""
    service = AsyncService()
    result = await service.process_data("input")
    assert result == "processed: input"
```

### 测试异步上下文管理器

```python
async def test_async_context_manager():
    """测试异步上下文管理器"""
    async with AsyncConnection() as conn:
        assert conn.is_connected
        result = await conn.execute("SELECT 1")
        assert result == 1
    # conn 自动关闭
```

### 测试异步迭代器

```python
async def test_async_iterator():
    """测试异步迭代器"""
    async_iter = async_range(5)

    results = []
    async for value in async_iter:
        results.append(value)

    assert results == [0, 1, 2, 3, 4]
```

### 测试异步生成器

```python
async def test_async_generator():
    """测试异步生成器"""
    results = [item async for item in async_generator()]

    assert len(results) > 0
    assert all(isinstance(item, int) for item in results)
```

## 常见场景

### 测试异步HTTP客户端

```python
import aiohttp

@pytest_asyncio.fixture
async def http_client():
    """创建HTTP客户端fixture"""
    session = aiohttp.ClientSession()
    yield session
    await session.close()

async def test_http_request(http_client, mocker):
    """Mock HTTP响应"""
    # Mock响应
    mock_response = mocker.Mock()
    mock_response.status = 200
    mock_response.json = AsyncMock(return_value={"result": "success"})

    # 使用真实客户端
    async with http_client.get("http://example.com/api") as resp:
        assert resp.status == 200
        data = await resp.json()
        assert data["result"] == "success"
```

### 测试异步数据库操作

```python
@pytest_asyncio.fixture
async def test_db():
    """创建测试数据库"""
    db = AsyncDatabase(":memory:")
    await db.create_table("users", ["id", "name"])
    yield db
    await db.close()

async def test_database_operations(test_db):
    """测试数据库CRUD操作"""
    # 插入
    await test_db.insert("users", {"id": 1, "name": "Alice"})

    # 查询
    user = await test_db.get("users", 1)
    assert user["name"] == "Alice"

    # 更新
    await test_db.update("users", 1, {"name": "Bob"})
    user = await test_db.get("users", 1)
    assert user["name"] == "Bob"

    # 删除
    await test_db.delete("users", 1)
    user = await test_db.get("users", 1)
    assert user is None
```

### 测试并发任务

```python
async def test_concurrent_tasks():
    """测试并发执行"""
    async def task(name, delay):
        await asyncio.sleep(delay)
        return f"{name} completed"

    # 并发执行
    results = await asyncio.gather(
        task("task1", 0.1),
        task("task2", 0.2),
        task("task3", 0.1)
    )

    assert len(results) == 3
    assert "completed" in results[0]
```

### 测试超时处理

```python
async def test_timeout_handling():
    """测试异步超时"""
    async def slow_operation():
        await asyncio.sleep(5)
        return "done"

    # 使用asyncio.wait_for
    with pytest.raises(asyncio.TimeoutError):
        await asyncio.wait_for(slow_operation(), timeout=1.0)
```

### 测试异常处理

```python
async def test_async_exception():
    """测试异步异常"""
    async def failing_operation():
        raise ValueError("Async error")

    with pytest.raises(ValueError, match="Async error"):
        await failing_operation()
```

### 测试异步资源清理

```python
@pytest_asyncio.fixture
async def resource_with_cleanup():
    """测试资源清理"""
    resource = AsyncResource()
    await resource.initialize()
    yield resource

    # cleanup验证
    assert resource.is_initialized
    await resource.cleanup()
    assert not resource.is_initialized

async def test_cleanup(resource_with_cleanup):
    """测试后自动清理"""
    assert resource_with_cleanup.is_initialized
    # 测试结束后自动调用cleanup
```

## Mock异步操作

### Mock异步函数

```python
from unittest.mock import AsyncMock

async def test_mock_async_function(mocker):
    """Mock异步函数"""
    mock_func = AsyncMock(return_value="mocked result")

    result = await mock_func("arg1", "arg2")
    assert result == "mocked result"

    mock_func.assert_called_once_with("arg1", "arg2")
```

### Mock异步方法

```python
async def test_mock_async_method(mocker):
    """Mock异步方法"""
    class AsyncService:
        async def fetch_data(self):
            return "real data"

    service = AsyncService()

    # Mock异步方法
    mocker.patch.object(service, "fetch_data",
                        new=AsyncMock(return_value="mocked data"))

    result = await service.fetch_data()
    assert result == "mocked data"
```

### Mock异步上下文管理器

```python
async def test_mock_async_context_manager(mocker):
    """Mock异步上下文管理器"""
    mock_manager = AsyncMock()
    mock_manager.__aenter__.return_value = mock_manager
    mock_manager.__aexit__ = AsyncMock(return_value=None)

    # 使用mock
    async with mock_manager as context:
        assert context is mock_manager
```

## 性能测试

### 测量异步操作时间

```python
import time

async def test_async_performance():
    """测试异步操作性能"""
    start_time = time.time()

    await async_operation()

    elapsed = time.time() - start_time
    assert elapsed < 1.0  # 应在1秒内完成
```

### 测试并发效率

```python
async def test_concurrent_efficiency():
    """测试并发执行效率"""
    start_time = time.time()

    # 顺序执行
    for i in range(10):
        await asyncio.sleep(0.1)

    sequential_time = time.time() - start_time

    start_time = time.time()

    # 并发执行
    await asyncio.gather(*[
        asyncio.sleep(0.1) for _ in range(10)
    ])

    concurrent_time = time.time() - start_time

    # 并发应该快得多
    assert concurrent_time < sequential_time / 2
```

## 最佳实践

1. **使用auto模式** - 除非需要支持多种异步库
2. **合理配置loop_scope** - 避免不必要的事件循环创建
3. **及时清理资源** - 使用yield后的cleanup代码
4. **避免过度并发** - 控制并发数量避免资源耗尽
5. **测试异常场景** - 不仅测试成功路径，也要测试失败路径
6. **使用fixture** - 将异步资源封装为fixture复用
7. **Mock外部依赖** - Mock网络、数据库等外部服务
8. **测试超时处理** - 确保合理的超时设置
9. **验证并发安全** - 测试共享资源的并发访问
10. **保持测试简单** - 每个测试只验证一个行为
