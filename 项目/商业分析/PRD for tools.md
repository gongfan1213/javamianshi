0409
当前的几个工具：
1.  **Wisebase 工具 (`wisebase_tool.py`)**:
    *   **用途**: 用于读写一个**共享知识库 (Wisebase)**，不同 Crew 可以共享信息，也可以按来源读取。
    *   **工具 1: `Wisebase Write Tool` (写入工具)**
        *   **职能**: 让 Agent 将一条知识（事实、假设、观点等）写入 Wisebase。写入时会自动标记来源 Crew。
        *   **Agent 使用时需提供的参数**:
            *   `content` (字符串, 字典 或 列表): 要存入的知识内容本身。
            *   `classification` (字符串): 对这条知识的分类 (例如: 'fact', 'hypothesis', 'point', 'context')。
            *   `crew_id` (字符串): 标识是哪个 Crew 写入的这条知识。
            *   `metadata` (可选的字典): 一个包含额外信息的字典，例如：
                *   `source_task_id` (字符串): 来源任务的ID。
                *   `source_agent_role` (字符串): 来源 Agent 的角色。
                *   `confidence` (浮点数): 置信度 (0.0-1.0)。
                *   `tags` (字符串列表): 相关标签。
                *   `related_qspace_node_ids` (字符串列表): 关联的 Q_space 节点ID。
    *   **工具 2: `Wisebase Read Tool` (读取工具)**
        *   **职能**: 让 Agent 从 Wisebase 读取知识。可以进行关键词搜索，也可以按写入的 Crew 进行过滤。
        *   **Agent 使用时需提供的参数**:
            *   `query` (字符串): 搜索关键词。如果想获取所有相关条目，请使用 'all'；否则将进行简单的内容匹配。
            *   `filter_crew_id` (可选的字符串): 如果提供，则只读取指定 `crew_id` 写入的条目。如果省略，则读取所有 Crew 的相关条目。

2.  **Q_space 工具 (`q_space_tool.py`)**:
    *   **用途**: 用于读取和更新**特定 Crew 的动态问题模型 (Q_space)**。每个 Crew 都有自己独立的 Q_space，通常是一个 JSON 文件。
    *   **工具 1: `Q_space Get Tool` (获取工具)**
        *   **职能**: 让 Agent 获取指定 Crew 的完整 Q_space 结构。结果是一个包含整个 Q_space 的 JSON 字符串。
        *   **Agent 使用时需提供的参数**:
            *   `crew_id` (字符串): 需要获取哪个 Crew 的 Q_space。
    *   **工具 2: `Q_space Update Tool` (更新工具)**
        *   **职能**: 让 Agent 用一个新的、完整的 JSON 结构**覆盖**指定 Crew 的 Q_space。工具会先验证 JSON 的有效性。
        *   **Agent 使用时需提供的参数**:
            *   `crew_id` (字符串): 需要更新哪个 Crew 的 Q_space。
            *   `full_json_structure` (字符串): 一个**完整且有效**的 JSON 字符串，代表新的 Q_space 结构。

3.  **Jina 搜索工具 (`jina_search_tool.py`)**:
    *   **用途**: 提供**深度网络搜索**能力。
    *   **工具: `Jina AI DeepSearch Tool`**
        *   **职能**: 让 Agent 使用 Jina AI 的 DeepSearch API 进行网络搜索，以获取最新的信息、文章或特定细节。
        *   **Agent 使用时需提供的参数**:
            *   `query` (字符串): 要搜索的问题或关键词。

总结来说：
*   `Wisebase` 工具让 Agents 能跨 Crew 共享和检索结构化的知识片段。
*   `Q_space` 工具让 Agents 能管理和迭代各自 Crew 内部的、表示问题结构的动态模型。
*   `Jina Search` 工具让 Agents 能进行实时的网络搜索。

这些工具都设计为接收简单的参数（主要是字符串、字典、列表），Agent 可以根据任务需求调用相应的工具并提供必要的输入。

————————————————————

修改计划：

Wisebase写入工具
1. 不应该需要metadata，而是应该在写入时自动记录来源任务的ID，来源Agent的角色，related_qspace_node_ids
2. 不需要tags和置信度参数

manager只负责读取和监控Wisebase的变化，



使用 JSON Knowledge Source 来为agent挂载知识