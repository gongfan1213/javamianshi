### 与 bluelab-opengraph 对比

#### 框架与编排
- 两者均使用 LangGraph：
  - opengraph：`app/graphs/graph.py` 使用 `StateGraph(AgentState)`，提供多种组合图（stage1/2/3 与会话图），MemorySaver 单例管理会话状态；
  - 本项目：在 `agents/graph/workflows/content_writing_graph.py` 使用 `StateGraph(ContentWritingState)`，并提供 Memory/Redis 双 checkpointer，配合 `graph_routes_v2` 提供 SSE 流式接口与人审中断/恢复。

#### 状态与阶段
- opengraph 的 `AgentState` 拆分为阶段字段（keywords/cards_map/insights_map/xiaohongshu_contents 等），并提供分阶段运行的图构造函数（`get_stage*_graph_runnable`）。
- 本项目的 `ContentWritingState` 定义 workflow_plan/current_stage/阶段产物，并由 `supervisor_node` 智能推进，适配多轮/人审场景。

#### 事件与接口
- opengraph：主要提供可编译的可运行图实例，未见统一的 SSE 事件接口；
- 本项目：`/graph_v2/chat/stream` 推送阶段/节点事件、人审提示、最终结果。

#### 可观测与检查点
- opengraph：示例中使用 `MemorySaver`，强调会话图单例与 `aget_state/aupdate_state`；
- 本项目：Memory/Redis 检查点可选，Graph 层与追踪（Langfuse）对齐，具备生产级恢复能力。

#### 结论
- opengraph 偏向于“模块化图构造与分段执行示例”；
- 本项目 偏向“生产可用的多阶段编排 + 中断恢复 + 流式接口”。


