# Pytest Mocking 和 Patching 完全指南

## 目录
- [使用 monkeypatch](#使用-monkeypatch)
- [使用 pytest-mock](#使用-pytest-mock)
- [Mock 对象](#mock-对象)
- [常见Mock模式](#常见mock模式)
- [Patch策略](#patch策略)
- [避免常见陷阱](#避免常见陷阱)

## 使用 monkeypatch

### monkeypatch.setattr - 替换属性/方法

```python
def test_replace_function(monkeypatch):
    # 替换模块函数
    def mock_get_data():
        return {"mocked": True}

    monkeypatch.setattr("myapp.data.get_data", mock_get_data)
    result = get_data()
    assert result == {"mocked": True}
```

```python
def test_replace_class_method(monkeypatch):
    # 替换类方法
    class API:
        def fetch(self):
            return "real data"

    def mock_fetch(self):
        return "mocked data"

    monkeypatch.setattr(API, "fetch", mock_fetch)

    api = API()
    assert api.fetch() == "mocked data"
```

### monkeypatch.setitem/delitem - 修改字典

```python
def test_with_mock_config(monkeypatch):
    # 添加配置项
    monkeypatch.setitem(app.config, "TEST_MODE", True)

    # 删除配置项
    monkeypatch.delitem(app.config, "PRODUCTION_DB", raising=False)

    # 修改配置
    monkeypatch.setitem(app.config, "API_KEY", "test_key")
```

### monkeypatch.setenv/delenv - 环境变量

```python
def test_with_environment(monkeypatch):
    # 设置环境变量
    monkeypatch.setenv("DATABASE_URL", "sqlite:///:memory:")

    # 删除环境变量
    monkeypatch.delenv("PRODUCTION_ENV", raising=False)

    # 验证
    assert os.getenv("DATABASE_URL") == "sqlite:///:memory:"
```

### monkeypatch.syspath_prepend - 修改路径

```python
def test_with_custom_path(monkeypatch, tmp_path):
    # 添加自定义模块路径
    custom_module = tmp_path / "custom_module"
    custom_module.mkdir()
    monkeypatch.syspath_prepend(str(custom_module))

    import custom_module  # 现在可以导入了
```

### monkeypatch.chdir - 改变工作目录

```python
def test_with_working_dir(monkeypatch, tmp_path):
    # 临时改变工作目录
    monkeypatch.chdir(tmp_path)

    # 测试依赖于当前目录的代码
    assert os.getcwd() == str(tmp_path)
    # 测试结束后自动恢复原目录
```

## 使用 pytest-mock

### 安装

```bash
pip install pytest-mock
```

### mocker.patch - 基本Mock

```python
def test_mocker_patch(mocker):
    # 创建mock对象
    mock_func = mocker.patch("module.function")

    # 调用被patch的函数
    module.function("arg1", "arg2")

    # 验证调用
    mock_func.assert_called_once_with("arg1", "arg2")
```

### 设置返回值

```python
def test_mock_return_value(mocker):
    mock_func = mocker.patch("module.external_api_call")
    mock_func.return_value = {"status": "success", "data": [1, 2, 3]}

    result = module.external_api_call()
    assert result == {"status": "success", "data": [1, 2, 3]}
```

### 设置副作用

```python
def test_mock_side_effect(mocker):
    # 抛出异常
    mock_func = mocker.patch("module.risky_operation")
    mock_func.side_effect = ValueError("Invalid input")

    with pytest.raises(ValueError, match="Invalid input"):
        module.risky_operation()
```

```python
def test_mock_side_effect_sequence(mocker):
    # 多次调用返回不同值
    mock_func = mocker.patch("module.retry_operation")
    mock_func.side_effect = [None, None, "success"]

    assert module.retry_operation() is None
    assert module.retry_operation() is None
    assert module.retry_operation() == "success"
```

### mocker.patch.object - 对象方法

```python
def test_patch_object(mocker):
    class Service:
        def method(self):
            return "original"

    service = Service()
    mock_method = mocker.patch.object(service, "method", return_value="mocked")

    assert service.method() == "mocked"
    mock_method.assert_called_once()
```

### mocker.patch.multiple - 多个属性

```python
def test_patch_multiple(mocker):
    class API:
        def method1(self): pass
        def method2(self): pass

    # 同时mock多个方法
    mocks = mocker.patch.multiple(API,
                                   method1=mocker.DEFAULT,
                                   method2=mocker.DEFAULT)

    API.method1()
    API.method2()

    mocks["method1"].assert_called_once()
    mocks["method2"].assert_called_once()
```

### autospec - 自动签名检查

```python
def test_autospec(mocker):
    # autospec 确保mock有相同的函数签名
    mocker.patch("module.process_data", autospec=True)

    # 正确调用
    module.process_data(data=[1, 2, 3], option=True)

    # 错误调用会抛出TypeError（因为签名不匹配）
    # module.process_data()  # TypeError: missing required arguments
```

## Mock 对象

### Mock的基本用法

```python
from unittest.mock import Mock

def test_mock_object():
    mock = Mock()

    # 任何属性访问都返回新的Mock
    mock.any_attribute.another_attribute

    # 可以设置返回值
    mock.method.return_value = "result"

    # 可以验证调用
    mock.method()
    assert mock.method.called
    assert mock.method.call_count == 1
```

### 配置Mock行为

```python
def test_configure_mock():
    mock = Mock()

    # 设置属性
    mock.name = "TestMock"
    mock.value = 42

    # 设置方法返回值
    mock.calculate.return_value = 100

    # 设置方法副作用
    mock.process.side_effect = [1, 2, 3]

    assert mock.calculate() == 100
    assert mock.process() == 1
    assert mock.process() == 2
```

### 验证调用

```python
def test_assertions():
    mock = Mock()

    mock.method("arg1", "arg2", kwarg="value")

    # 验证是否被调用
    assert mock.method.called

    # 验证调用次数
    assert mock.method.call_count == 1

    # 验证调用参数
    mock.method.assert_called_once_with("arg1", "arg2", kwarg="value")

    # 验证至少调用一次
    mock.method.assert_called_with("arg1", "arg2", kwarg="value")

    # 验证从未调用
    mock.other_method.assert_not_called()
```

### 访问调用历史

```python
def test_call_history():
    mock = Mock()

    mock.method(1, 2)
    mock.method(3, 4)

    # 查看所有调用
    print(mock.method.call_args_list)
    # [call(1, 2), call(3, 4)]

    # 查看第一次调用
    print(mock.method.call_args)
    # call(3, 4)

    # 按索引访问调用
    print(mock.method.call_args_list[0])
    # call(1, 2)
```

## 常见Mock模式

### Mock外部API调用

```python
import requests

def test_api_call(mocker):
    # Mock requests.get
    mock_response = mocker.Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"result": "success"}

    mocker.patch("requests.get", return_value=mock_response)

    # 测试代码
    result = fetch_from_api("https://api.example.com/data")
    assert result == {"result": "success"}

    # 验证调用
    requests.get.assert_called_once_with("https://api.example.com/data")
```

### Mock数据库操作

```python
def test_database_save(mocker):
    # Mock数据库session
    mock_session = mocker.Mock()
    mocker.patch("myapp.db.get_session", return_value=mock_session)

    # 执行操作
    save_user("John", "john@example.com")

    # 验证数据库操作
    mock_session.add.assert_called_once()
    mock_session.commit.assert_called_once()
```

### Mock文件系统

```python
def test_file_operations(mocker):
    # Mock文件读取
    mocker.patch("builtins.open", mocker.mock_open(read_data="file content"))

    result = read_file("test.txt")
    assert result == "file content"
```

### Mock时间相关

```python
from unittest.mock import patch
from datetime import datetime

def test_time_dependent_code():
    # 固定时间
    fixed_time = datetime(2025, 1, 1, 12, 0, 0)
    with patch("myapp.module.datetime") as mock_datetime:
        mock_datetime.now.return_value = fixed_time

        result = get_timestamp()
        assert result == "2025-01-01 12:00:00"
```

## Patch策略

### Patch的位置很重要

```python
# ❌ 错误：patch定义的位置
from myapp import utils

def test_wrong(mocker):
    mocker.patch("utils.external_call")  # 不会生效

# ✅ 正确：patch使用的地方
def test_correct(mocker):
    mocker.patch("myapp.utils.external_call")  # 正确
```

### 使用字符串路径

```python
# ✅ 推荐：使用字符串路径
def test_with_string_path(mocker):
    mocker.patch("myapp.module.function_name")

# ❌ 避免：直接导入对象
def test_with_direct_import(mocker):
    from myapp.module import function_name
    mocker.patch.object(function_name)  # 可能不生效
```

### 类方法的Patch

```python
def test_patch_class_method(mocker):
    # Patch类方法
    mocker.patch("myapp.models.User.save")

    user = User(username="test")
    user.save()  # 使用mock

    # 验证
    User.save.assert_called_once()
```

### 实例方法的Patch

```python
def test_patch_instance_method(mocker):
    user = User(username="test")

    # Patch特定实例的方法
    mocker.patch.object(user, "save")

    user.save()  # 使用mock

    user.save.assert_called_once()
```

## 避免常见陷阱

### 陷阱1：过度Mock

```python
# ❌ 过度mock - 测试mock本身而非实际逻辑
def test_over_mocked(mocker):
    mocker.patch("module.function_a")
    mocker.patch("module.function_b")
    mocker.patch("module.function_c")
    # 没有实际代码被测试！

# ✅ 适度mock - 只mock外部依赖
def test_appropriate_mock(mocker):
    mocker.patch("module.external_api")  # 只mock外部API
    result = module.internal_logic()  # 测试内部逻辑
    assert result == expected
```

### 陷阱2：Patch错误的路径

```python
# ❌ 错误：在定义处patch
from myapp.utils import helper

def test_wrong_patch(mocker):
    mocker.patch("helper.some_function")  # 错误路径

# ✅ 正确：在使用处patch
def test_correct_patch(mocker):
    mocker.patch("myapp.utils.helper.some_function")  # 正确路径
```

### 陷阱3：忘记设置autospec

```python
# ❌ 没有autospec - 调用签名错误也不会报错
def test_without_autospec(mocker):
    mock = mocker.patch("module.process")
    module.process()  # 本来需要参数，但不会报错

# ✅ 使用autospec - 调用错误会立即发现
def test_with_autospec(mocker):
    mocker.patch("module.process", autospec=True)
    # module.process()  # TypeError: missing required arguments
```

### 陷阱4：在fixture中mock但未返回

```python
# ❌ fixture中mock但测试无法访问
@pytest.fixture
def mock_api(mocker):
    mocker.patch("myapp.api.call")

# ✅ 返回mock对象以便测试使用
@pytest.fixture
def mock_api(mocker):
    return mocker.patch("myapp.api.call")

def test_api(mock_api):
    # 现在可以配置和验证mock
    mock_api.return_value = {"data": "test"}
    # ...
```

## 最佳实践

1. **优先使用monkeypatch** - 对于简单的替换，monkeypatch更简洁
2. **使用pytest-mock** - 对于复杂的mock场景，pytest-mock更强大
3. **Patch正确的位置** - 在使用的地方patch，不是定义的地方
4. **使用autospec** - 确保mock与原对象签名一致
5. **验证调用** - 不仅设置mock，还要验证它被正确调用
6. **避免过度mock** - 只mock外部依赖，测试真实逻辑
7. **使用fixture** - 将常用mock封装成fixture
