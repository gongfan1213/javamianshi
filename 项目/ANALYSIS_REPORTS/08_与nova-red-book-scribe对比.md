### 与 nova-red-book-scribe 对比

#### 框架与栈
- scribe（前端工具型项目）：以 React/TS + 组件为主，辅助内容生产/编辑工作流；代码中未检索到 `langgraph/StateGraph` 使用。
- 本项目：后端 Agent 编排为主，LangGraph 多阶段流程，SSE 事件输出与可观测追踪。

#### 能力互补
- scribe 偏 UI/生产力工具，适合作为前端界面与交互；
- 本项目 提供可恢复的 Agent 后端流程与 API，可为 scribe 提供后端服务能力；
- 结合方式：scribe 以 SSE 消费 `/graph_v2/chat/stream`，可视化阶段/节点事件与人审交互。


