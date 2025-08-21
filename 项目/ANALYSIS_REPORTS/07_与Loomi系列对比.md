### 与 Loomi 系列对比（Loomi2.0/3.0/3.1）

#### 框架与组织
- Loomi 系列：未检索到 `langgraph/StateGraph/create_react_agent` 明确使用痕迹，流程以类/函数组织的 orchestrator/concierge 等为核心；
- 本项目：显式采用 LangGraph DAG + `@tool` + `create_react_agent`，将编排与工具层收敛在 `agents/graph/**`。

#### 能力差异
- 本项目：
  - 可恢复：检查点（Memory/Redis）+ 中断/恢复；
  - 事件：SSE 推送阶段/节点级事件，前端更好集成；
  - 观测：Langfuse 会话级追踪统一 `/chat` 与 `/graph_v2`；
  - 工具：`@tool` 自动 schema，create_react_agent 动态注入；
- Loomi：
  - 成熟的场景化 prompts 与多 Agent 协作范式；
  - 更贴近业务的 orchestration，但恢复/事件语义主要由业务逻辑承担。

#### 迁移建议
- 从 Loomi 中抽取稳定功能为 `@tool`，以最小侵入纳入本项目的子图；
- 保留 Loomi 的提示词资产与模式，逐步迁移到基于状态/阶段的 DAG 表达；
- 建一套对照测试（输入/输出/事件）保证迁移质量。


