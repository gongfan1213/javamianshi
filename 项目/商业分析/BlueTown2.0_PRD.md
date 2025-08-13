## 产品需求文档 (PRD): BlueTown 2.0 - 协作式 AI 分析平台

**版本:** 1.0
**日期:** 2024-07-31

**1. 引言**

BlueTown 2.0 是一个旨在利用多 AI 智能体协作，自动化完成复杂商业咨询和分析任务的平台。它模拟了企业内部多个不同部门（如竞争分析部、用户研究部、场景挖掘部、煽动力研究部等）**并行运作、协同分析**的模式。**其核心优势在于强大的"动态问题建模"能力，支持非线性的图谱式推理 (Graph Reasoning)，而非简单的线性任务执行。** 核心驱动逻辑摒弃了拟人化的角色设定，转而采用基于**状态 (State)** 和**条件触发器 (Condition Triggers)** 的功能模块化设计。本项目将选用 **CrewAI** 框架作为底层工程实现的基础，以利用其在智能体编排、任务管理和工具集成方面的能力，并着重通过自定义工具和流程设计来实现动态问题建模。

**2. 目标**

*   **自动化分析:** 接收用户输入的商业简报需求 (brief)，**分发给多个相关部门 (Crews)**，各自自动执行一系列研究、分析、总结任务，最终整合输出多维度的结构化分析结果。
*   **结构化知识沉淀:** 在各部门分析过程中，动态构建和维护一个**共享但区分来源**的结构化知识库 (`Wisebase`)，包含事实、假设和观点，并支持跨部门（`Crew`）高效共享。
*   **动态问题建模 (`Q_space`):** **将复杂问题实时构建为一个结构化的、动态演化的"问题模型/任务图谱" (`Q_space`)**，能反映问题的层级、变量、假设、潜在冲突和概念定义。基于当前的知识状态 (`Wisebase`)，**灵活地调整模型，进行深度、非线性的图谱式推理。**
*   **模块化与可扩展:** 设计清晰的模块边界（部门/`Crew`、执行单元/`Agent`、管理功能），易于扩展新的分析能力和部门。
*   **高效协作与冲突解决:** 实现不同 AI 智能体 (`Agent`) 之间以及不同部门 (`Crew`) 之间的有效信息同步和任务协同，**并能主动识别和处理 `Wisebase` 中的信息冲突，驱动更深层次的分析。**

**3. 核心用例 (更新版：多部门并行启动)**

**场景:** 用户需要对"亚一黄金"进行全面的市场机会分析。

1.  **输入:** 用户向 BlueTown 系统提交 `brief`："分析亚一黄金的市场机会，综合考虑竞争格局、用户需求、潜在应用场景及品牌沟通策略。"
2.  **分发与初步处理:**
    *   系统的**入口/分发机制**被触发。
    *   `brief` 被初步解析，识别出需要调动【竞争分析部】、【用户研究部】、【场景挖掘部】、【煽动力研究部】等多个部门 (`Crew`)。
    *   结构化的 `brief` 信息被推送给每个相关 `Crew`。
3.  **初始问题建模 (Q_space v1 构建 - 各部门并行):**
    *   对于每个被激活的 `Crew`，`Dispatcher` (或 `Crew` 自身) **创建一个特殊的初始 `Task`："根据用户 brief 和本部门职能，构建初始问题模型 (Q_space v1 YAML)"**。
    *   这个 `Task` 被分配给该 `Crew` 的 **Manager Agent** (或一个专用的 **Problem Decomposition Agent**)。
    *   该 Agent 执行此 `Task`，利用 LLM 的理解和分解能力，结合部门定位，生成初始的 `Q_space` YAML 结构，并将其存储（例如，与该 `Crew` 关联的文件或数据库记录）。这个 YAML 代表了对问题的初步结构化理解。
4.  **任务规划与执行启动 (基于 Q_space v1):**
    *   各 `Crew` 的 **Manager Agent** 读取其生成的 `Q_space` v1 YAML。
    *   Manager Agent **解析 `Q_space` 结构，识别出初始可行动的节点** (例如，`status: open` 的 `sub_question` 或 `hypothesis`)。
    *   Manager Agent **将这些 `Q_space` 节点转化为一个或多个具体的、可执行的 CrewAI `Tasks`** (定义 `description`, `expected_output`, 分配 `agent`)。
    *   这些初始的 CrewAI `Tasks` 被添加到 `Crew` 的任务队列中，由 CrewAI 的 `Process` 开始调度执行。
    *   相应的 `Q_space` 节点状态更新为 `in_progress`。
5.  **迭代执行、知识共享与动态调整 (循环):**
    *   `Agents` 执行 CrewAI `Tasks`，调用 `Tools`。
    *   执行结果通过**知识状态管理**模块更新到共享的 `Wisebase` (标记来源 `Crew`)。
    *   `Wisebase` 的更新**触发 Manager Agent 重新评估 `Q_space`**。
    *   Manager Agent 更新 `Q_space` 节点状态 (e.g., `answered`, `conflicted`)，链接 `Wisebase` 证据。
    *   Manager Agent 基于更新后的 `Q_space` 和 `Wisebase` **持续识别新的可行动节点，生成新的 CrewAI `Tasks`** (包括处理冲突的任务)，或**扩展 `Q_space`** (添加新节点/分支)。
    *   新生成的 `Tasks` 加入执行队列。此循环持续进行，直到 `Q_space` 中的主要问题得到解答或达到预设停止条件。
6.  **整合与输出:**
    *   当所有相关 `Crew` 的关键任务完成或达到某个汇聚点时，一个**顶层整合机制**（可以是另一个 `Crew` 或特定流程）被触发。
    *   该机制从 `Wisebase` 中提取各部门的关键 `facts` 和 `points`。
    *   整合不同维度的分析结果，形成一份全面的市场机会分析报告。

**4. 功能需求 (基于状态与触发器)**

| 功能模块                  | 触发条件                                   | 主要功能                                                                 | 状态影响/输出                                                                 |
| :------------------------ | :----------------------------------------- | :----------------------------------------------------------------------- | :-------------------------------------------------------------------------- |
| **入口/分发机制**       | 接收到用户 `brief`                        | 解析 `brief` 意图；确定需要激活哪些部门 (`Crew`)；分发结构化 `brief`          | 触发多个目标 `Crew` 的【输入处理与结构化】流程                                |
| **输入处理与结构化**      | (被分发机制触发) 接收到结构化 `brief`     | (部门内) 解析、清洗、提取与本部门职能相关的关键信息                         | 初始化本 `Crew` 的 `Wisebase` 相关条目                                          |
| **知识状态管理 (`Wisebase`)** | 接收到本 `Crew` 或其他 `Crew` 的新信息输入 | 对信息进行分类、验证、去重、处理冲突；**记录信息来源 (`Crew` ID)**；更新知识库 | 维护共享 `Wisebase` 的动态一致性；支持按来源查询                                |
| **任务规划与生成 (`Q_space`)** | 本 `Crew` 的 `Wisebase` 状态发生显著变化  | 基于 `Wisebase` 信息 **和本部门职能定位**，生成/调整任务计划；分配任务给 `Agents` | 创建/更新本 `Crew` 的 `Q_space` 结构                                        |
| **执行单元 (`Agent`)**    | 被分配到 `Task`                           | 根据 `role`, `goal`, `backstory`, 使用 `Tools` 执行 `Task`           | 返回结构化的执行结果给**知识状态管理**模块                                      |
| **跨部门信息同步**        | 任务规划/知识分析中识别到跨部门信息需求     | 查询共享 `Wisebase` (可按来源过滤)；提取并整合相关信息                     | 将获取的信息写入本 `Crew` 的 `Wisebase` (触发本 Crew 的**知识状态管理**)       |
| **顶层结果整合**          | 关键 `Crews` 完成主要目标或达到汇聚条件    | 从 `Wisebase` 提取各部门关键结论；整合、去重、结构化输出                       | 输出最终的多维度分析报告                                                      |

**5. 非功能需求**

*   **性能:** LLM 调用需优化以控制成本和延迟。`Wisebase` 的读写操作需高效。
*   **可扩展性:** 方便添加新的部门 (`Crew`)、执行者 (`Agent`) 和工具 (`Tool`)。
*   **可靠性:** 能处理 `Agent` 执行失败或 `Tool` 不可用的情况 (e.g., 重试、备用方案)。
*   **可维护性:** 代码结构清晰，配置（`Agent` 定义、`Tool` 配置）易于管理。
*   **状态持久化:** `Wisebase` 和 `Q_space` 的状态应能持久化，支持任务中断和恢复 (可选)。
*   **可观测性:** 提供日志记录和追踪，方便调试 `Agent` 交互和任务流程。

**6. 系统架构与 CrewAI 实现**

*   **核心框架:** CrewAI
*   **关键概念区分:**
    *   **问题模型 (`Q_space`):** (YAML 结构) 代表对问题的**结构化理解**。非线性，图状，动态演化。存储问题的分解、假设、概念、冲突等。**是思考和规划的基础。**
    *   **任务图谱/队列 (CrewAI `Tasks` & `Process`):** 代表解决问题的**具体执行步骤**。通常是 DAG 或队列，管理执行顺序和依赖。**是行动的载体。**
*   **入口/分发器:**
    *   **实现方式:** 可以是一个简单的 Python 脚本/函数，或者一个专门的轻量级 "Dispatcher Crew"。
    *   **职责:** 接收原始 `brief`，调用 LLM (或规则) 判断需要哪些部门 `Crew` 参与，然后实例化并启动这些 `Crew`，传入初始 `brief` 信息。
*   **部门 (`Crew`):** 每个业务部门实现为一个独立的 `Crew` 实例。
    *   `Crew(agents=[...], tasks=[...], process=Process.hierarchical, memory=False)`
    *   `Process.hierarchical` 的 Manager Agent 的 **Prompt/Role 需要特别配置**，以反映该部门的独特分析视角和规划偏好 (e.g., 竞争分析部Manager更关注市场份额，用户研究部Manager更关注用户痛点)。
*   **执行者 (`Agent`):** 与之前设计一致，专注于具体执行任务。
*   **知识状态 (`Wisebase`):**
    *   **实现方式:** 仍然通过**自定义 `Tool` (`WisebaseTool`)**，但需要强调其**共享性**和**来源追踪**。
    *   **`WisebaseTool` (增强版):**
        *   后端存储需要支持多 `Crew` 访问（可以是同一个数据库表/集合，通过 `crew_id` 字段区分）。
        *   `write(data: Dict, classification: str, crew_id: str)`: 写入时必须包含来源 `Crew` ID。
        *   `read(query: str, filter_crew_id: Optional[str] = None) -> List[Dict]`: 读取时可以按来源 `Crew` ID 过滤，或读取所有 `Crew` 的信息。
        *   `query_other_crew_wisebase` 方法可能不再需要，因为所有信息都在共享库中，通过 `read` 配合 `filter_crew_id` 实现类似效果。
*   **动态问题模型/任务图谱 (`Q_space`):**
    *   **核心定位:** `Q_space` 不是简单的任务列表，而是对当前问题的结构化理解和探索路径图，是进行图谱式推理的基础。 它动态地反映了问题的分解、关键变量、待验假设、概念定义和潜在冲突点。
    *   **实现方式:** 每个 `Crew` 内部维护自己的 `Q_space` 实例。其状态 **以结构化的 YAML 格式 (或 JSON) 存储和管理**，以提升可读性和便于维护 (具体存储后端可选，如内存、文件、数据库)。YAML 利用缩进清晰地表示层级关系。
    *   **结构核心要素:**
        *   `id`: 节点的唯一标识符。
        *   `text`: 节点表示的具体内容 (问题、假设、概念等)。
        *   `type`: 节点的性质 (e.g., `main_question`, `sub_question`, `hypothesis`, `concept_definition`, `potential_factor`...)，指导 Agent 如何处理该节点。
        *   `status`: 节点的当前状态 (e.g., `open`, `in_progress`, `answered`, `conflicted`, `closed`...)，驱动任务规划和状态更新。
        *   `children`: 包含子节点的列表，体现问题的分解和发散性思考。
        *   (可选/运行时关联) `related_wisebase_ids`: 关联到 `Wisebase` 中支撑或反驳该节点的知识条目 ID。
    *   **动态演化:** `Q_space` 不是静态的。当 `Agent` 的研究在 `Wisebase` 中产生新信息、验证了假设或发现了冲突时，Manager Agent (或特定工具) 会更新对应 `Q_space` 节点的 `status` (以及可能的 `related_wisebase_ids`)。更重要的是，如果研究引出了全新的问题或假设，可以在 `Q_space` 结构中添加新的节点/分支。

*   **Manager Agent (Hierarchical Process):**
    *   **核心职责:**
        1.  **(初始)** 执行"构建初始 `Q_space` v1"的任务 (如果选择此方案)。
        2.  **持续地** 作为 `Crew` 内的"问题建模师"和"推理引擎"：**将 `Q_space` (问题理解) 转化为可执行的 CrewAI `Tasks` (行动计划)**。
    *   **工作方式:**
        1.  **解析 `Q_space`:** 读取并理解当前的 `Q_space` YAML 结构。
        2.  **识别可行动节点:** 查找 `status` 为 `open`, `conflicted` 等需要处理的节点。
        3.  **结合 `Wisebase`:** 查询相关知识。
        4.  **生成 CrewAI `Tasks`:**
            *   **判断映射:** 决定一个 `Q_space` 节点是映射到单个 `Task` 还是需要拆分为多个 `Tasks`。
            *   **定义 `Task`:** 精心设计 CrewAI `Task` 的 `description`, `expected_output`, 选择 `agent`, 设置必要的 `context` (依赖)。
            *   **加入队列:** 将生成的 `Task`(s) 提交给 CrewAI `Process` 管理的任务队列。
        5.  **更新 `Q_space` 状态:** 将对应节点的 `status` 标记为 `in_progress`。
        6.  **(后续循环)** 监控 `Task` 完成和 `Wisebase` 更新。
        7.  **处理结果:** 更新 `Q_space` 节点状态 (`answered` 等) 和关联 `Wisebase` 信息。
        8.  **冲突检测与解决:** 主动检查冲突，标记 `Q_space` 节点为 `conflicted`，并生成专门的冲突解决 `Tasks`。
        9.  **模型扩展:** 按需在 `Q_space` 中添加新节点/分支。
    *   **Prompt 定制:** Manager Agent 的 Prompt 至关重要，需要明确指导其执行上述问题建模、图谱推理、冲突处理的逻辑，并融入该 `Crew` (部门) 的特定视角和分析偏好。

*   **`Wisebase` 与 `Q_space` 的相互作用:**
    *   `Q_space` 指导信息采集方向 (生成 `Tasks`)。
    *   `Tasks` 的执行结果填充和更新 `Wisebase`。
    *   `Wisebase` 的状态变化（新知识、冲突发现）反馈给 `Q_space`，更新节点状态并驱动 Manager Agent 调整模型和规划后续步骤。
    *   这种**闭环反馈**是实现动态问题建模和深度推理的关键。

*   **管理功能实现映射 (更新):**
    *   **任务规划:** **这是一个持续的、由 Manager Agent 驱动的过程，核心是将 `Q_space` 的状态变化翻译成具体的 CrewAI `Tasks` 并加入执行队列。**
    *   其他映射更新，明确初始 Q_space 构建步骤。

**7. 数据模型**

*   **Wisebase 条目 (增加来源和关联Q_space节点ID - 可选但推荐):**
    ```json
    {
      "id": "unique_wisebase_entry_id",
      "crew_id": "competition_analysis",
      "related_qspace_node_ids": ["q_jzh_vs_others", "hyp_jzh_econ"], // 关联的Q_space节点
      "content": "...",
      "classification": "fact",
      "source_task_id": "...",
      "source_agent_role": "...",
      "timestamp": "...",
      "confidence": 0.9,
      "tags": [...]
    }
    ```

*   **Q_space (动态问题模型/任务图谱 - YAML 结构示例):**
    ```yaml
    # Q_space Root: JZH Gold Consumption Logic
    id: q_root_jzh_gold
    text: 中国江浙沪的三线下沉市场的黄金珠宝消费逻辑有什么特殊点？
    type: main_question
    status: open
    children:
      - id: q_jzh_context
        text: 江浙沪地域特征分析
        type: concept_definition_group # 概念定义组
        status: open
        children:
          - id: q_jzh_vs_others
            text: 江浙沪与中国其他地区的主要区别是什么？
            type: sub_question
            status: open
            children:
              - id: hyp_jzh_econ
                text: 消费水平：经济发达，人均可支配收入高
                type: hypothesis
                status: open
              - id: hyp_jzh_family
                text: 家庭结构：独生子女家庭多
                type: hypothesis
                status: open
              - id: q_jzh_culture
                text: 文化习俗有何特点？
                type: sub_question
                status: open
                children:
                  - id: hyp_jzh_culture_biz
                    text: 改革开放早，重商主义和创业精神足
                    type: hypothesis
                    status: open
                  - id: hyp_jzh_culture_intl
                    text: 国际化程度高，对西方文化/品牌认同更高
                    type: hypothesis
                    status: open
      - id: q_3rd_tier_context
        text: 三线下沉市场定义与江浙沪特殊性
        type: concept_definition_group
        status: open
        children:
          - id: def_3rd_tier_general
            text: 一般定义：可能是县城、农村等，相对落后
            type: potential_definition
            status: open
          - id: q_3rd_tier_jzh_special
            text: 江浙沪语境下的三线市场有何不同？
            type: sub_question
            status: open
            children:
              - id: hyp_3rd_jzh_rich_village
                text: 可能是富裕示范村，创业氛围浓
                type: hypothesis
                status: open
              - id: hyp_3rd_jzh_migrant_biz
                text: 家庭成员可能在外经商，过年回来消费
                type: hypothesis
                status: open
              - id: q_3rd_jzh_class_diff
                text: 不同阶层（中产/小康/穷人）在江浙沪三线市场的具体表现有何不同？
                type: sub_question
                status: open
      - id: q_gold_logic_context
        text: 黄金珠宝消费逻辑要素拆解
        type: concept_definition_group
        status: open
        children:
          - id: q_gold_industry
            text: 行业品牌和产品形态有哪些？
            type: sub_question
            status: open
            children:
              - id: data_gold_brand_jewelry
                text: 品牌金饰 (周大福等): 日常佩戴、兼顾保值
                type: data_point_category # 需要调研具体品牌/产品
                status: open
              - id: data_gold_investment
                text: 投资金 (金条/金豆子): 投资保值为主
                type: data_point_category
                status: open
              - id: data_luxury_metal
                text: 奢侈品贵金属 (卡地亚等): 看重品牌/设计
                type: data_point_category
                status: open
          - id: q_gold_experience
            text: 消费体验流程有哪些？
            type: sub_question
            status: open
            children:
              - id: data_channel_offline
                text: 线下渠道
                type: data_point_category
                status: open
              - id: data_channel_online
                text: 线上渠道
                type: data_point_category
                status: open
          - id: q_gold_behavior_types
            text: 具体的消费行为有哪些类型？
            type: sub_question
            status: open
            children:
              - id: data_behavior_brand
                text: 买品牌金饰
                type: data_point_category
                status: open
              - id: data_behavior_beans
                text: 攒金豆子
                type: data_point_category
                status: open
              - id: data_behavior_luxury
                text: 买奢侈品贵金属
                type: data_point_category
                status: open
          - id: q_gold_behavior_reason
            text: 这些不同行为背后的真实原因/动机可能是什么？(可能是杂糅)
            type: sub_question
            status: open
            children:
              - id: hyp_reason_investment
                text: 投资保值
                type: potential_factor
                status: open
              - id: hyp_reason_life_event
                text: 生辰婚嫁
                type: potential_factor
                status: open
              - id: hyp_reason_social
                text: 社交炫耀
                type: potential_factor
                status: open
              - id: hyp_reason_self_pleasure
                text: 悦己娱乐
                type: potential_factor
                status: open
              - id: hyp_reason_luxury_bundle
                text: 奢侈品配货
                type: potential_factor
                status: open
      - id: q_logic_diff_focus
        text: 最终研究焦点：消费逻辑的异同
        type: research_focus_definition # 研究焦点定义
        status: open
        children:
          - id: q_define_logic
            text: 什么是"消费逻辑"？
            type: concept_definition
            status: open
            children:
              - id: def_logic_process
                text: 视角1: 完整的信息输入和决策流程 (行为主体)
                type: potential_definition
                status: open
              - id: def_logic_subjective
                text: 视角2: 消费者主观内心世界 (个体主体)
                type: potential_definition
                status: open
          - id: q_factors_influencing_logic
            text: 什么因素会导致消费逻辑不同？(决策影响因素)
            type: sub_question
            status: open
            children:
              - id: factor_family_structure
                text: 例：家庭结构 (大家族 vs 三口之家) -> 对他人评价的考量不同
                type: potential_factor_example
                status: open
              # 这里可以继续添加其他潜在因素
    ```

**8. 未来考虑**

*   `Q_space` JSON 结构的图形化展示与交互界面。
*   更复杂的 `Q_space` 节点关系（例如，支持节点间的非父子依赖）。
*   Manager Agent 动态规划能力的持续优化和评估。
*   `Wisebase` 与 `Q_space` 状态同步的鲁棒性设计。
*   更精细的 `Wisebase` 冲突检测与解决机制。
*   引入向量数据库支持更智能的 `Wisebase` 语义查询。
*   为 `Agent` 提供更复杂的长期记忆能力。
*   开发用户界面 (UI) 以便用户提交 `brief`、监控进度和查看结果。
*   增加更多类型的 `Agent` 和 `Tool` 以支持更广泛的分析领域。
*   评估 Sequential Process 在某些线性任务场景下的适用性。










# Crew 架构设计

单个部门 `Crew` 内部的架构设计，特别是 `Manager Agent` 如何与我们设计的动态问题模型 (`Q_space`) 和 CrewAI 的任务队列 (`Tasks`/`Process`) 进行交互。

参考 CrewAI 的文档，特别是关于 Hierarchical Process 的部分，我们可以勾勒出部门 `Crew` 内部的运作蓝图：

**部门 Crew 内部架构组件:**

1.  **`Crew` 实例:** 代表特定部门 (e.g., `competition_analysis_crew`)。
    *   `agents`: 包含该部门的所有执行者 `Agent`。
    *   `tasks`: **初始为空或只包含一个“构建初始 Q_space”的任务。** 后续的任务由 Manager Agent 动态生成和委派。
    *   `process`: **`Process.hierarchical`** (必须)。
    *   `manager_llm`: **必须指定**，这是驱动 Manager Agent 思考和决策的核心 LLM。
    *   `manager_agent`: **可选**，可以指定一个自定义的 Agent 作为 Manager，赋予其更复杂的内置逻辑或特定工具，否则 CrewAI 会基于 `manager_llm` 创建一个默认 Manager。对于我们的复杂逻辑，**定义一个定制的 `manager_agent` 并赋予其特定工具可能是更好的选择**。
    *   `memory`: 可能需要启用 (`True`)，以便 Manager Agent 更好地跟踪其自身的规划过程和与 Agent 的交互历史，但这不同于结构化的 `Wisebase`。
    *   `planning`: 可以启用 (`True`)，允许 Manager 在执行前制定更高层次的计划（这可能与我们的 `Q_space` 有重叠，需要仔细协调）。

2.  **执行者 `Agents`:** 部门内的具体工作人员 (e.g., `ResearcherAgent`, `DataAnalystAgent`)。
    *   拥有明确的 `role`, `goal`, `backstory`。
    *   配备完成其职能所需的 `Tools`，**必须包含 `WisebaseTool`** 以便读写共享知识库。

3.  **Manager `Agent` (核心):** 由 `Process.hierarchical` 提供 (最好是自定义的 `manager_agent`)。
    *   **职责:** 问题建模师、推理引擎、任务规划器和协调者。
    *   **所需关键工具:**
        *   **`WisebaseTool`:** 访问共享知识库，获取全局信息和兄弟部门的发现。
        *   **`Q_spaceTool` (自定义):** 这是**至关重要的自定义工具**，用于与本 `Crew` 的 `Q_space` YAML 状态进行交互。至少需要提供：
            *   `read_qspace()`: 读取并返回当前 `Q_space` 的 YAML 内容或解析后的结构。
            *   `find_nodes(status: str, type: Optional[str] = None) -> List[Node]`: 查找指定状态和类型的节点。
            *   `update_node_status(node_id: str, new_status: str)`: 更新指定节点的状态。
            *   `add_node(parent_id: str, node_data: Dict)`: 在指定父节点下添加新的子节点（用于模型扩展）。
            *   `link_wisebase_entry(node_id: str, wisebase_id: str)`: 将 `Wisebase` 条目关联到 `Q_space` 节点。

4.  **`Q_space` 状态 (YAML):** 每个 `Crew` 独立维护的问题模型，通过 `Q_spaceTool` 访问和修改。

5.  **CrewAI `Tasks` / `Process`:** CrewAI 内置的任务对象和执行流程 (由 Hierarchical Process 管理)。Manager Agent 的输出（规划和委派）会转化为具体的 CrewAI `Tasks` 并放入这个流程中执行。

**Manager Agent 与各组件的配合流程 (迭代循环):**

1.  **触发:** Manager Agent 被 CrewAI 的 Hierarchical Process 激活以进行下一步规划。
2.  **状态感知:**
    *   `Manager` 使用 `Q_spaceTool.read_qspace()` 获取当前问题模型的完整状态。
    *   `Manager` 使用 `Q_spaceTool.find_nodes(status='open')` (以及可能的 `conflicted` 等) 识别出需要处理的节点。
    *   对于识别出的节点，`Manager` 使用 `WisebaseTool.read(query=...)` 查询 `Wisebase` 中与之相关的背景信息、现有证据或冲突信息（查询内容可以基于节点 `text` 和其在 `Q_space` 中的上下文）。
3.  **决策与规划 (LLM 核心能力):**
    *   基于其 Prompt 中定义的部门视角、问题建模逻辑和冲突处理策略，结合从 `Q_space` 和 `Wisebase` 获取的状态信息，`Manager` 的 `manager_llm` 进行思考：
        *   **优先级判断:** 哪些 `open` 节点最需要优先处理？
        *   **任务转化:** 如何将选定的 `Q_space` 节点 (一个或多个) 转化为具体的、可执行的 CrewAI `Task`？（如前所述，可能一对一或一对多拆分）。需要明确 `description`, `expected_output`。
        *   **Agent 分配:** 哪个执行者 `Agent` 最适合执行这个新生成的 `Task`？
        *   **冲突处理:** 如果发现 `conflicted` 节点，规划解决冲突的 `Task`。
        *   **模型扩展:** 是否需要通过 `Q_spaceTool.add_node()` 来扩展问题模型？
4.  **任务委派 (通过 CrewAI 机制):**
    *   Manager Agent 的输出 (其 LLM 的响应) **会被 CrewAI 的 Hierarchical Process 解析为任务分配指令**。这个输出需要符合 CrewAI 对 Manager 输出的预期格式（可能需要通过 Prompt Engineering 来确保）。
    *   CrewAI 框架负责将这些指令转化为实际的 `Task` 对象，并将其分配给指定的执行者 `Agent`。
5.  **状态更新 (Q_space):**
    *   在任务被委派后，`Manager` (或通过回调) 使用 `Q_spaceTool.update_node_status()` 将对应节点的 `status` 更新为 `in_progress`。
6.  **执行与反馈:**
    *   执行者 `Agent` 执行 CrewAI 分配的 `Task`。
    *   `Agent` 使用 `WisebaseTool` 读取所需信息，并在任务结束时将结果写入 `Wisebase` (通过 `WisebaseTool.write()`)。
    *   任务完成后，结果会反馈给 Hierarchical Process。
7.  **结果处理与模型更新:**
    *   `Manager Agent` (通过 Hierarchical Process 机制) 接收到任务完成的信号和结果 (可能需要验证)。
    *   `Manager` 使用 `Q_spaceTool.update_node_status()` 更新对应节点的 `status` (e.g., `answered`)。
    *   `Manager` 使用 `Q_spaceTool.link_wisebase_entry()` 将生成的 `Wisebase` 条目 ID 关联到 `Q_space` 节点。
    *   Manager Agent 进入下一个规划循环 (返回步骤 1)。

**关键实现要点:**

*   **`Q_spaceTool` 是核心粘合剂:** 它是 Manager Agent 理解和操作问题模型的关键接口。
*   **Manager Agent Prompt 是灵魂:** 需要精心设计，包含问题建模策略、任务转化逻辑、冲突处理规则以及部门特有的分析视角。
*   **CrewAI Hierarchical Process 是引擎:** 它提供了 Manager-Worker 的基本交互框架和任务调度执行机制。我们需要在其基础上，通过自定义 Manager Agent 和工具来注入我们复杂的动态建模逻辑。

这个设计明确了 Manager Agent 在 Crew 内部的核心作用：**持续读取和更新结构化的问题模型 (`Q_space`) 和知识库 (`Wisebase`)，并基于此动态地生成和委派具体的执行任务 (CrewAI `Tasks`) 给下属 Agents。**



# 项目架构

bluetown_core/
├── .env                     # Environment variables: API Keys, DB connections, etc.
├── config/                  # Global configurations
│   └── settings.yaml        # LLM settings (default model, temp), Wisebase config (type, connection)
├── data/                    # Persistent data storage (optional, if not using external DBs)
│   └── wisebase_store/      # Example: Storage for Wisebase if file-based or local DB
│   └── q_space_archive/     # Optional: Archive completed Q_space YAMLs
├── docs/                    # Project documentation
│   └── BlueTown2.0_PRD.md
├── src/                     # Main source code
│   ├── __init__.py
│   ├── crews/               # Each department is a Crew Python package
│   │   ├── __init__.py
│   │   ├── competition_analysis/ # Example: Competition Analysis Crew
│   │   │   ├── __init__.py
│   │   │   ├── config/         # Configuration specific to this Crew
│   │   │   │   ├── agents.yaml # Defines executor agents & custom manager_agent specifics
│   │   │   │   └── tasks.yaml  # Defines the initial Q_space build task template
│   │   │   ├── agents.py     # Logic to load/define Agents for this Crew (imports from YAML)
│   │   │   ├── tasks.py      # Logic to load/define Tasks (imports from YAML, includes Q_space build logic)
│   │   │   └── crew.py       # Defines & configures the Crew instance (agents, initial task, process, manager)
│   │   ├── user_research/    # Example: User Research Crew
│   │   │   └── ... (similar structure)
│   │   ├── scene_discovery/  # Example: Scene Discovery Crew
│   │   │   └── ...
│   │   ├── persuasion_research/ # Example: Persuasion Research Crew
│   │   │   └── ...
│   │   └── reporting/        # Example: Crew responsible for final report integration
│   │       └── ...
│   ├── tools/               # Shared custom tools used across Crews
│   │   ├── __init__.py
│   │   ├── base_tool.py    # Optional: Base class for custom tools
│   │   ├── wisebase_tool.py # Defines WisebaseTool class (interacts with Wisebase store)
│   │   └── q_space_tool.py  # Defines Q_spaceTool class (interacts with Q_space YAML state)
│   ├── state/               # Runtime state management
│   │   └── q_space/         # Directory holding the current Q_space YAML for each active Crew run
│   │       ├── competition_analysis.yaml # Dynamically created/updated during runtime
│   │       ├── user_research.yaml
│   │       └── ...
│   └── utils/               # Shared utility functions (e.g., YAML parsing, logging setup)
│       └── __init__.py
│       └── parsing.py
│       └── logging_config.py
├── dispatcher.py            # Main entry point: Handles brief input, selects/initializes Crews, runs them, manages overall flow
├── requirements.txt         # Python package dependencies (crewai, langchain, pyyaml, etc.)
└── README.md                # Project overview, setup instructions, architecture summary