### apis/app.py 做什么
- 创建并配置 FastAPI 应用（标题、描述、文档路径、CORS、模板/静态资源、中间件）。
- 注入请求级上下文与鉴权/限流（仅对白名单中的接口启用）。
- 统一注册业务路由（核心 `router`、`/health`、恢复流 `recovery_router`）。
- 生命周期钩子：启动时初始化持久化；优雅关闭。
- 开发模式下可直接 `uvicorn` 启动。

### 关键点解读
- Python 3.13 兼容处理
  - 启动早期注入 `utils.aioredis_fix`，解决 aioredis 在 3.13 的兼容问题（必须先于其他导入）。

- 应用与基础设施
  - 开放式 CORS（生产建议收敛域名）。
  - 模板目录 `templates`；若存在 `static/` 则挂载 `/static`。
  - 加载 `apis.middleware.setup_middleware`，提供请求日志与耗时头。

- 请求中间件：上下文、鉴权、限流
```85:112:apis/app.py
@app.middleware("http")
async def require_id_middleware(request: Request, call_next):
    request_id = str(uuid.uuid4()).replace("-", "")
    require_id.set(request_id)
    ...
    if (request.url.path in API_WHITELIST and ...):
        user_token = headers.get("authorization")
        current_user_id = supabase_check_token(user_token)
        if not current_user_id:
            return JSONResponse(status_code=401, content=...)
        user_id.set(current_user_id)
        # 基于用户ID的限流（Redis 配置驱动）
        rate_limiter = await get_rate_limiter()
        is_allowed, rate_info = await rate_limiter.is_allowed(identifier=current_user_id)
        ...
```
  - “白名单”语义为受保护列表：仅名单内的接口进行 Token 校验与限流（如 `/loomichat`、上传接口）。不在名单内则跳过鉴权/限流。
  - 触发限流返回 429，并附带标准限流响应头；同时异步发送告警。
  - 成功通过后，为响应附加剩余额度头：`X-RateLimit-*`。
  - 使用 `core.context` 的 `require_id`、`user_id` 作为 contextvar，在 `finally` 中清理，避免串话。

- 路由注册
```229:233:apis/app.py
app.include_router(router)                 # 业务主路由，根挂载
app.include_router(health_router, prefix="/health")
app.include_router(recovery_router)        # 断线恢复/回放
```

- 生命周期
```203:219:apis/app.py
@app.on_event("startup")
async def startup_event():
    success = await startup_persistence()  # 初始化持久化（失败则退回内存模式）
```

- 根路由与开发启动
  - GET `/` 返回服务元信息和常用端点提示。
  - 直接运行文件时：`uvicorn apis.app:app --reload`（监听 `0.0.0.0:8000`）。

### 常见操作
- 想让新接口也强制鉴权限流：把路径加入 `API_WHITELIST`。
- 想全局开放：移出名单或调整中间件逻辑。
- 想自定义限流策略：在 Redis 中调整限流配置或扩展 `utils.rate_limiter`。
- 生产安全建议：收敛 CORS、扩展非白名单的鉴权策略、完善 `API_WHITELIST`。

- 小结
  - 该文件负责应用装配与请求“守门员”（上下文、鉴权、限流）。将核心业务分发到 `apis/routes.py`、健康与恢复路由分别独立，启动时确保存储层可用并提供清晰的可观测性与保护措施。
