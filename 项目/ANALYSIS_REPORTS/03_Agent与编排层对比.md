### Agent 与编排层对比

#### dev：以 LangGraph 为核心的可恢复编排
- 工作流声明：`agents/graph/workflows/content_writing_graph.py`
  - `StateGraph(ContentWritingState)`，节点包括 `supervisor`、`requirement_alignment`、`content_search`、`content_writing`、`human_confirmation`；
  - `add_conditional_edges("human_confirmation", route_after_confirmation, {...})` 进行分支路由；
  - 提供 `compile_content_writing_with_memory/with_redis`，注入 `MemorySaver` 或自研 `AsyncRedisSaver` 检查点；
  - `debug=True` 便于开发期定位。
- 状态模型：`agents/graph/state/content_writing_state.py`
  - `TypedDict` 定义工作流阶段、消息、阶段产物、是否人审、最终输出；
  - 明确 `current_stage` 与 `workflow_plan`，便于阶段驱动与 UI 呈现。
- 节点逻辑：`agents/graph/node/supervisor_node.py` 等
  - `content_supervisor_node` 动态生成或推进 `workflow_plan`，根据阶段产物决定流向（`Send("content_search")` 等）；
  - 当 `need_human_review` 为真时，触发人审分支；
  - 使用 `Command`/`Send`/`END` 使控制流显式可视。
- 工具与代理：`agents/graph/agent/content_writing_agent.py`
  - `@tool` 定义工具，`create_react_agent(model, tools=[...])` 注入；
  - 工具职责清晰（规划/创作/评审/打包），提升组合与复用能力。

#### prod：以 Nova3/Loomi 传统流程为主
- 以 `agents/nova3/`、`agents/loomi/` 的类/函数编排为主，未见 `StateGraph`；
- 多模块串接通过自定义调度/队列与 LangSmith 追踪；
- 工具接入以直接函数/类组合为主，缺少统一的 `@tool` schema 与 create_react_agent 注入路径。

#### 差异影响
- 确定性与可调试性：dev 的 DAG 与显式状态更利于定位与复现。
- 扩展性：dev 的工具注入与节点拆分降低新能力接入成本。
- 恢复能力：dev 原生支持检查点与中断恢复，适配长链路与多轮协作。
- 迁移成本：可将 prod 的 agent 能力逐步改造成 `@tool` 并纳入子图，分阶段推进。


