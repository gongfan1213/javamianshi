### API 路由与端点对比（dev vs prod）

#### 核心差异
- **dev 新增 LangGraph 流式接口**：`/graph_v2/chat/stream`（SSE），封装三阶段 StateGraph 流程，包含阶段事件、节点事件、人审中断提示与恢复；并提供获取/恢复/状态接口雏形。
- **prod 保持传统聊天接口为主**：`/chat`、`/loomichat` 等；未发现 `graph_v2` 路由与 `StateGraph` 流式执行端点。

#### 证据文件
- dev：`apis/graph_routes_v2.py` 定义 `APIRouter(prefix="/graph_v2")` 与 `POST /graph_v2/chat/stream`，使用 `app.astream(input_state, config)` 推送事件；并通过 `utils.llm_client_factory.get_traced_llm_client` 开启/结束会话级追踪。
- dev：`apis/app.py` 显式 `include_router(graph_routes_v2)`，白名单含 `/graph/chat`、`/graph/resume`、`/graph/status`。
- prod：`apis/app.py`、`apis/routes.py` 未包含 `graph_routes_v2`，以传统 `/chat` 路由为核心。

#### 流式事件设计（dev）
- 事件类型：`graph_start`、`system`（trace 开始/结束）、`stage_change`、`node_execution`、`node_event`、`human_review_required`、`final_result`、`graph_end`、`error`。
- 输入状态：累积 `messages` 历史；以 `user_id:session_id` 作为 `thread_id` 接入检查点；递归深度提升到 50。
- 人审：当节点置位 `need_human_review`，服务端中断流，客户端经 `/graph/resume` 继续。

#### 影响
- 客户端：基于 SSE 实时渲染阶段/节点级事件，交互与可观测性更强。
- 服务端：需维护 Graph 生命周期、检查点恢复、追踪统一化。
- 兼容：可与传统 `/chat` 并存，逐步引流到 `/graph_v2/chat/stream`。


