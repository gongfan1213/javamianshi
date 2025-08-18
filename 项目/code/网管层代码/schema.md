### 这份文件做什么
`apis/schemas.py` 定义了整个后端 API 的“数据契约”（Pydantic 模型 + 枚举）。它约束了：
- 请求体结构（尤其是聊天请求 `ChatRequest` 及其 `RequestData`）
- 流式事件（SSE）的统一格式（`StreamEvent`、`EventPayload`、`EventType`、`ContentType`）
- 导出/内容创作等非流式接口的数据模型
- Nova3/Loomi 相关的选择、计划、类型映射，确保与前端协议兼容

### 结构总览
- 枚举类：`InteractionType`、`EventType`、`ContentType`、导出相关 `ExportContentType`、`ExportFormat`
- 基础信息与选择类：`BackgroundInfo`、`Selected*` 系列、`Nova3UserSelections`
- 主请求/事件模型：`RequestData`、`ChatRequest`、`EventPayload`、`StreamEvent`、错误/简单响应
- 导出模型：`ExportItem`、`ExportRequest`、`ExportResponse`
- 内嵌了完整示例，展示如何传 `nova3_selections`、`references` 等

### 关键枚举

- InteractionType（交互类型）
  - 完整分析、快速建议、文案创作、视觉诊断，以及 Nova3 的继续/选择模式
```10:19:apis/schemas.py
class InteractionType(str, Enum):
    FULL_ANALYSIS = "full_analysis"
    QUICK_SUGGESTION = "quick_suggestion"
    WRITE_COPY = "write_copy"
    DIAGNOSE_AVATAR = "diagnose_avatar"
    NOVA3_CONTINUE = "nova3_continue"
    NOVA3_SELECT = "nova3_select"
```

- EventType（流式事件类型）
  - system/user/tool_call/tool_return/llm_chunk/error
```20:28:apis/schemas.py
class EventType(str, Enum):
    SYSTEM = "system"
    USER = "user"
    TOOL_CALL = "tool_call"
    TOOL_RETURN = "tool_return"
    LLM_CHUNK = "llm_chunk"
    ERROR = "error"
```

- ContentType（语义内容类型）
  - 覆盖“思考/计划步骤/洞察/提案/画像/打点/文案/最终答案”等
  - BlueChat XML 标签类（planning/present_ideas/message_ask_user/quick_search/output_to_canvas）
  - Nova3 结果类型（insight/profile/hitpoint/facts/xhs_post/tiktok_script/wechat_article 等）
  - Loomi 专用类型全部“映射复用”为 `nova3_*`，保持前端兼容
```72:86:apis/schemas.py
# Loomi系统专用类型 - 🎯 更新为新的标签名称
LOOMI_RESONANT = "nova3_resonant"
...
LOOMI_POST_MESSAGE = "nova3_post_message"
CONCIERGE_WEBSEARCH = "nova3_concierge_websearch"
```

### 选择与背景信息模型
- `BackgroundInfo`：账号/平台/受众/行业/品牌风格等，可选
- `Selected*`：洞察/画像/打点/事实/小红书/抖音/公众号/思考等“用户勾选项”，字段统一为 `id/content/selected/user_comment`
- `Nova3UserSelections`：把上述各类选择聚合在一起，附带 `task_context`，供 Nova3/Supervisor/子Agent使用

### 主请求与事件模型

- RequestData（一次请求的具体数据）
  - 必填：`query` 字符串（min_length=1）
  - 可选：`background`、`interaction_type`、`interaction_context`
  - Nova3：`nova3_selections`
  - Loomi：`user_selections`（手动模式下按 notes 名称精准选取）
  - 通用：`references`（如 `["@insight1", "@profile2"]`）、`auto`（自动选择）、`file_ids`（多模态）
```234:272:apis/schemas.py
class RequestData(BaseModel):
    query: str = Field(..., min_length=1, max_length=2000)
    background: Optional[BackgroundInfo] = None
    interaction_type: InteractionType = Field(default=InteractionType.FULL_ANALYSIS)
    interaction_context: Optional[Dict[str, Any]] = Field(default={})
    nova3_selections: Optional[Nova3UserSelections] = None
    references: Optional[List[str]] = Field(default=[])
    auto: Optional[bool] = Field(default=False)
    user_selections: Optional[List[str]] = Field(default=[])
    file_ids: Optional[List[str]] = Field(default=[])
```
- 注意：`query` 是必填且长度≥1；即便主要依赖 `references` 或 `user_selections`，也需要提供非空 `query` 以通过校验（否则 FastAPI 会返回 422）。

- ChatRequest（外层聊天请求）
  - 顶层携带 `user_id`、`session_id`（正则限制 `^[a-zA-Z0-9_-]+$`），以及 `request_data`
```274:279:apis/schemas.py
class ChatRequest(BaseModel):
    user_id: str = Field(..., pattern=r"^[a-zA-Z0-9_-]+$")
    session_id: str = Field(..., pattern=r"^[a-zA-Z0-9_-]+$")
    request_data: RequestData
```

- StreamEvent / EventPayload（SSE 统一事件协议）
  - `StreamEvent`：`event_type`（见上）、`agent_source`（谁产生的事件）、`timestamp`（ISO）、`payload`
  - `EventPayload`：`id`、`content_type`（见上）、`data`（文本或结构化列表）、`metadata`
  - 发送至前端时，前端据 `event_type + content_type` 分类渲染
```287:293:apis/schemas.py
class StreamEvent(BaseModel):
    event_type: EventType
    agent_source: str
    timestamp: str = Field(default_factory=lambda: datetime.now().isoformat())
    payload: EventPayload
```

- 错误与简单响应
  - `ErrorResponse`：标准错误载体（message/code/details）
  - `SimpleResponse`：非流式接口的轻量响应

### 导出/内容创作相关
- `ExportContentType`（xhs_post/tiktok_script/wechat_article/all）、`ExportFormat`（txt/json/csv）
- `ExportItem`：单条导出项（id/title/content/type/user_comment）
- `ExportRequest`：三种导出来源（来自 `nova3_selections`、直接给 `export_items`、或指定 `round_num` 从历史轮次导出），可选是否包含评论/元数据/自定义分隔符
- `ExportResponse`：导出结果的汇总与原文内容

### 内嵌示例
文件底部提供了完整示例，展示首轮请求与“用户选择后”的二次请求如何携带 `nova3_selections` 与 `references`：
```307:366:apis/schemas.py
# Nova3用户选择字段使用示例：...
# 1. 首次请求
# 2. 用户选择后的后续请求（含 nova3_selections 与 references）
# 3. 数据流转过程
# 4. 引用机制
```

### 与路由的协作
- 入参：`/novachat`、`/loomichat` 等端点直接接收 `ChatRequest`
- 出参（流式）：服务端将中间过程/结果包装为 `StreamEvent` 逐条发送
- 出参（非流式）：导出接口返回 `ExportResponse`

### 设计要点与注意
- Query 作为“驱动字段”被设计为必填，保持接口统一性；若主要依赖 `references`/`user_selections`，仍需传入一个最小非空字符串。
- Loomi 的 `LOOMI_*` 类型被映射到 `nova3_*`，避免前端分支爆炸，复用渲染逻辑。
- 流式事件的 `event_type` 表示“数据流阶段/性质”，`content_type` 表示“语义类别”，二者组合让前端稳定解码多 Agent 的细粒度输出。
- `timestamp` 一律 ISO 字符串，便于日志与审计。

- 可扩展性：新增能力时，优先扩展 `ContentType` 与对应前端渲染；若引入 Planner/并行证据（ReWOO 风格），建议新增 plan/evidence/citation 等 `content_type` 与结构化 `data`。

小结
- 该文件是 API 的“协议中枢”，规范了聊天请求、流式事件、导出与用户选择的完整数据结构。
- Loomi/Nova3 的语义类型在这里统一，保证后端多 Agent 输出与前端展示的强一致与可演进性。
