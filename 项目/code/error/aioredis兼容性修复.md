- 遇到的问题：在 Python 3.13 中，asyncio.TimeoutError 与内置的 builtins.TimeoutError 被合并为同一个类，旧版 aioredis 的 `aioredis.exceptions` 里定义的
  `class TimeoutError(asyncio.TimeoutError, builtins.TimeoutError, RedisError)` 等多继承会触发 TypeError: duplicate base class TimeoutError。

- 修复方式：用自定义 MetaPathFinder/Loader 拦截模块导入，在导入 `aioredis.exceptions` 时注入“修复后的实现”，把 TimeoutError 改为只继承一个 TimeoutError 基类，避免重复基类冲突；其他异常类保持不变。仅在 Python >= 3.13 时启用。

- 使用要点：务必在任何导入 aioredis 之前先 `import utils.aioredis_fix`（例如应用入口处），这样拦截器会提前挂载到 `sys.meta_path`。

关键代码片段：
```17:31:utils/aioredis_fix.py
if sys.version_info >= (3, 13):
    import importlib.abc
    ...
    class AioredisExceptionsFixer(importlib.abc.MetaPathFinder, importlib.abc.Loader):
        ...
        def find_spec(self, fullname, path, target=None):
            if fullname == 'aioredis.exceptions':
                return importlib.machinery.ModuleSpec(fullname, self)
            return None
```

```60:66:utils/aioredis_fix.py
# 修复后的TimeoutError - 只继承一个TimeoutError基类
class TimeoutError(asyncio.TimeoutError, RedisError):
    """Fixed TimeoutError for Python 3.13 compatibility"""
    pass
```

- 效果：避免在 Python 3.13 环境中导入 aioredis 时抛出“duplicate base class TimeoutError”，保证 aioredis 正常使用。
