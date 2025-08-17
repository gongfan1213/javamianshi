
````markdown
# BluePlan Research: Gen-AI Agent for Social Media

> 📱 BluePlan 是一款**面向社交媒体内容创作者与营销运营人员的一站式生成式智能解决方案**，聚焦账号定位、内容策略、爆款推荐、视觉优化与文案生成，采用 `Supervisor + Researcher Agents` 架构，为创作与增长赋能。

---

## 🎯 项目定位

`blueplan_research` 旨在成为一款面向**社媒内容生产、策划、优化**场景的智能多 Agent 平台，具备：

- **多平台支持**：小红书 / 抖音 / 公众号（可拓展）。
- **生成式能力**：内容策略 + 文案 + 图片 + 封面建议一体化输出。
- **智能解耦**：采用 `Supervisor + Researcher Agents` 模式，分工明确。
- **架构现代**：支持分布式部署、支持高并发架构、链路日志追踪、LLM 模型灵活切换。
- **可集成性强**：提供标准化的 API 接口，可轻松嵌入第三方运营系统。
- **配置解耦**：系统配置、提示词、LLM 客户端完全解耦，支持热切换。

---

## 🧠 架构总览：Supervisor + Researcher Agents

       ┌────────────────────────┐
       │     SupervisorAgent    │
       │ (任务规划/意图识别 + 结果整合) │
       └───────────┬────────────┘
                   ↓
┌────────────────--─┬───────────────────┬──────────────────┐
│ Researcher:       │ Researcher:       │ Researcher:      │
│ UserInsight       │ ContentStrategy   │ FactCollection   │
│ (用户洞察/人设分析)  │ (内容策略/爆款挖掘) │ (事实搜集/案例分析)│
└────────┬──────────┴───┬───────────────┴─────────┬────────┘
         ↓              ↓                       ↓
┌────────────────┐ ┌────────────────┐ ┌───────────────────┐
│ DiagnoseAvatarAgent│ XHSWriterAgent     │ ... (Other Agents)│
│ (封面/头像优化建议)  │ (小红书文案生成)   │                   │
└────────────────┘ └────────────────┘ └───────────────────┘

---

## 🚀 核心交互：API 与流式协议

所有与 BluePlan 的交互都通过一个核心的 FastAPI 接口 `/chat` 完成。该接口接收用户请求，并以**流式（Streaming）** 的方式返回实时结果。

### 1. Chat API Endpoint

客户端通过调用此接口与 SupervisorAgent 开始一次交互。

**`POST /chat`**

#### 请求体 (Request Body)

请求体为一个 JSON 对象，包含了会话信息、用户身份和具体的任务指令。

```json
{
  "user_id": "user_abc_123",
  "session_id": "session_xyz_789",
  "request_data": {
    "query": "我的小红书账号流量一直上不去，感觉笔记没人看，能帮我分析下原因并给出优化建议吗？",
    "background": {
      "account_name": "我的手工烘焙坊",
      "platform": "小红书",
      "target_audience": "20-30岁，喜欢DIY和甜品的女性",
      "previous_content": [
        "笔记标题：自制提拉米苏，一次成功！",
        "笔记标题：周末Vlog：我的烘焙日常"
      ]
    },
    "interaction_type": "full_analysis",
    "interaction_context": {}
  }
}
````

**字段说明:**

  - `user_id` (string, required): 唯一用户标识符。
  - `session_id` (string, required): 唯一会话标识符，用于追踪一次完整的对话流程。
  - `request_data` (object, required):
      - `query` (string, required): 用户的核心自然语言请求。
      - `background` (object, optional): 用户提供的背景信息，帮助 Agent 更好地理解上下文。
          - `account_name`: 账号名称。
          - `platform`: 目标平台（如 小红书, 抖音）。
          - `target_audience`: 目标受众描述。
          - `previous_content`: 用户过往发布的内容示例。
      - `interaction_type` (enum, optional): 交互类型，用于指导 SupervisorAgent 的行为。例如: `"full_analysis"`, `"quick_suggestion"`, `"write_copy"`. 默认为 `"full_analysis"`。
      - `interaction_context` (object, optional): 在多轮交互中，传递用户选择或补充的信息。例如 `{"selected_topic": "提拉米苏封面优化"}`。

#### 响应 (Response)

响应是一个基于 **Server-Sent Events (SSE)** 的流。每个事件都是一个独立的 JSON 对象，客户端可以实时解析并展示。这保证了用户可以立刻看到 Agent 的思考过程和初步结果，无需等待最终报告生成。

### 2\. 流式输出协议 (Streaming Protocol)

每个通过 SSE 发送的事件都遵循以下结构。通过 `event_type` 和 `content_type` 标签，前端可以清晰地知道当前这条信息是什么、来自哪里、应该如何展示。

#### 标准事件结构

```json
{
  "event_type": "llm_chunk",
  "agent_source": "SupervisorAgent",
  "timestamp": "2025-06-23T18:30:00.123Z",
  "payload": {
    "content_type": "plan_step",
    "data": "1. 分析用户账号'我的手工烘焙坊'的现有内容和人设定位。"
  }
}
```

#### 事件类型 (`event_type`)

| `event_type`   | 说明                                                              |
| :------------- | :---------------------------------------------------------------- |
| `system`       | 系统消息，如 "分析开始"、"任务完成" 等。                           |
| `user`         | 回显用户输入的消息，确认系统已接收。                              |
| `tool_call`    | Agent 正在调用一个工具（如搜索引擎、数据库查询）。                |
| `tool_return`  | 工具执行完毕后返回的结果（通常不对用户直接展示，用于调试）。      |
| `llm_chunk`    | **核心事件**: LLM 生成内容的实时片段。 payload 中包含具体内容类型。 |
| `error`        | 表示流程中发生了错误。                                            |

#### LLM 内容类型 (`payload.content_type` for `llm_chunk`)

这个标签用来区分 LLM 输出的具体内容属性，由产生内容的 Agent 决定。

| `content_type` | Agent 来源 (示例)              | 说明                                         |
| :------------- | :----------------------------- | :------------------------------------------- |
| `thought`      | 所有 Agent                     | Agent 的内心思考过程或推理链（Chain of Thought）。 |
| `plan_step`    | `SupervisorAgent`              | Supervisor 生成的执行计划步骤。              |
| `insight`      | `ContentStrategy`, `UserInsight` | 对市场、用户或内容的深刻洞察。               |
| `persona`      | `UserInsight`                  | 目标用户的画像分析或建议的人设。             |
| `case_study`   | `FactCollection`               | 分析的对标账号或爆款案例。                   |
| `suggestion`   | `DiagnoseAvatar`               | 具体的、可执行的优化建议（如封面、标题）。   |
| `copywriting`  | `XHSWriter`, `MPWriter`        | 生成的文案、标题、标签等。                   |
| `final_answer` | `SupervisorAgent`              | 由 Supervisor 整合所有结果后生成的最终报告。 |

-----

## 🤖 Agent 模块详解

每个 Agent 都是一个独立的专家，有明确的职责和输出内容类型。

| Agent 类型                 | 模块文件                  | 职责说明                                           | 核心输入                               | 核心输出 (流式 `content_type`)                  | 提示词配置文件 |
| -------------------------- | --------------------- | -------------------------------------------------- | -------------------------------------- | ----------------------------------------------- | ------------- |
| **SupervisorAgent** | `agents/supervisor.py`       | **意图识别与任务规划**：接收用户请求，生成 SmartPlan，调度子 Agent，并整合最终报告。 | 用户 `request_data`                    | `plan_step`, `thought`, `final_answer`          | `supervisor_system.txt`, `supervisor_user.txt` |
| **Researcher:UserInsight** | `agents/user_insight.py`     | **用户洞察**：分析用户背景信息，定义账号人设和目标受众画像。 | 账号背景, 历史内容                     | `persona`, `insight`, `thought`                 | `user_insight_system.txt`, `user_insight_user.txt` |
| **Researcher:ContentStrategy** | `agents/content_strategy.py` | **内容策略**：研究平台热门趋势，挖掘爆款选题和对标账号。 | 平台, 领域, 受众画像                   | `insight`, `case_study`, `suggestion`, `thought` | `content_strategy_system.txt`, `content_strategy_user.txt` |
| **Researcher:FactCollection** | `agents/fact_collection.py` | **事实搜集**：通过搜索等工具，查找具体的案例、数据来支撑策略。 | Supervisor下发的具体问题（如"近期热门烘焙笔记"） | `case_study`, `tool_return`（原始数据）         | `fact_collection_system.txt`, `fact_collection_user.txt` |
| **Specialist:DiagnoseAvatar**| `agents/diagnose_avatar.py`| **视觉诊断**：分析账号的头像、封面图，并结合爆款案例提出优化建议。 | 用户封面图, 对标案例                  | `suggestion`, `insight`, `thought`              | `diagnose_avatar_system.txt`, `diagnose_avatar_user.txt` |
| **Specialist:XHSWriter** | `agents/writer_xhs.py`       | **文案创作（小红书）**：根据策略和人设，生成小红书风格的标题、正文和标签。 | 选题, 人设, 关键词                     | `copywriting`, `thought`                        | `writer_xhs_system.txt`, `writer_xhs_user.txt` |
| **Specialist:MPWriter** | `agents/writer_mp_dy.py`     | **文案创作（公众号/抖音）**：生成适用于其他平台的文案。 | 选题, 人设, 平台特性                   | `copywriting`, `thought`                        | `writer_mp_dy_system.txt`, `writer_mp_dy_user.txt` |

-----

## 🧠 Smart Plan 执行示例

**输入 (`/chat` Request):**

```json
{
  "query": "小红书封面转化率低怎么办？",
  "background": { "platform": "小红书", "account_name": "我的手工烘焙坊" }
}
```

**流式输出 (Streamed Response Preview):**

```json
// 1. SupervisorAgent 开始规划
{ "event_type": "llm_chunk", "agent_source": "SupervisorAgent", "payload": { "content_type": "plan_step", "data": "1. 分析'我的手工烘焙坊'当前封面风格与近期同类爆款的差异。" } }
{ "event_type": "llm_chunk", "agent_source": "SupervisorAgent", "payload": { "content_type": "plan_step", "data": "2. 调用内容策略Agent，查找3个烘焙类爆款笔记进行案例分析。" } }
{ "event_type": "llm_chunk", "agent_source": "SupervisorAgent", "payload": { "content_type": "plan_step", "data": "3. 调用视觉诊断Agent，基于案例生成3种封面优化建议。" } }
{ "event_type": "llm_chunk", "agent_source": "SupervisorAgent", "payload": { "content_type": "plan_step", "data": "4. 汇总建议，生成最终报告。" } }

// 2. ContentStrategyAgent 开始执行
{ "event_type": "system", "agent_source": "ContentStrategy", "payload": { "data": "正在分析烘焙类爆款案例..." } }
{ "event_type": "llm_chunk", "agent_source": "ContentStrategy", "payload": { "content_type": "case_study", "data": "案例1: '@烘焙少女喵'，特点：高饱和度色彩，成品特写，手写体标题..." } }

// 3. DiagnoseAvatarAgent 开始执行
{ "event_type": "system", "agent_source": "DiagnoseAvatar", "payload": { "data": "正在生成封面优化建议..." } }
{ "event_type": "llm_chunk", "agent_source": "DiagnoseAvatar", "payload": { "content_type": "suggestion", "data": "建议风格A（食欲风）: 采用顶光拍摄，突出蛋糕的光泽感，放大细节..." } }
{ "event_type": "llm_chunk", "agent_source": "DiagnoseAvatar", "payload": { "content_type": "suggestion", "data": "建议风格B（故事感）: 加入人物互动元素，如递出一块蛋糕的手，增加场景氛围..." } }

// 4. SupervisorAgent 整合并输出最终结果
{ "event_type": "llm_chunk", "agent_source": "SupervisorAgent", "payload": { "content_type": "final_answer", "data": "综合分析，我们为您提供以下优化方案：..." } }
{ "event_type": "system", "agent_source": "SupervisorAgent", "payload": { "data": "分析完成。" } }
```

-----

## 📁 项目结构（简化架构）

```
blueplan_research/
├── main.py                    # 主启动类：BluePlanApp，支持本地调试和API启动
├── agents/                    # 所有智能体模块（核心业务逻辑）
│   ├── __init__.py
│   ├── base_agent.py          # Agent基类，定义通用接口和流式输出逻辑
│   ├── supervisor.py          # SupervisorAgent：任务规划、子任务调度
│   ├── user_insight.py        # UserInsightAgent：用户洞察分析
│   ├── content_strategy.py    # ContentStrategyAgent：内容策略研究
│   ├── fact_collection.py     # FactCollectionAgent：事实搜集
│   ├── diagnose_avatar.py     # DiagnoseAvatarAgent：视觉诊断
│   ├── writer_xhs.py          # XHSWriterAgent：小红书文案创作
│   └── writer_mp_dy.py        # MPWriterAgent：公众号/抖音文案创作
├── apis/                      # API接口模块（对外服务层）
│   ├── __init__.py
│   ├── app.py                 # FastAPI应用实例和配置
│   ├── routes.py              # API路由定义（包含 /chat 接口）
│   ├── schemas.py             # Pydantic数据模型（请求/响应结构）
│   └── middleware.py          # 中间件（日志、CORS、错误处理等）
├── config/                    # 配置管理（完全解耦）
│   ├── __init__.py
│   ├── settings.py            # 系统配置类：LLM配置、API配置等
│   ├── llm_config.yaml        # LLM模型配置：OpenAI, Claude, Gemini等
│   └── app_config.yaml        # 应用配置：端口、日志级别、缓存等
├── prompts/                   # 提示词库（系统+用户提示词分离）
│   ├── __init__.py
│   ├── supervisor_system.txt     # SupervisorAgent系统提示词
│   ├── supervisor_user.txt       # SupervisorAgent用户提示词模板
│   ├── user_insight_system.txt   # UserInsightAgent系统提示词
│   ├── user_insight_user.txt     # UserInsightAgent用户提示词模板
│   ├── content_strategy_system.txt  # ContentStrategyAgent系统提示词
│   ├── content_strategy_user.txt    # ContentStrategyAgent用户提示词模板
│   ├── fact_collection_system.txt   # FactCollectionAgent系统提示词
│   ├── fact_collection_user.txt     # FactCollectionAgent用户提示词模板
│   ├── diagnose_avatar_system.txt   # DiagnoseAvatarAgent系统提示词
│   ├── diagnose_avatar_user.txt     # DiagnoseAvatarAgent用户提示词模板
│   ├── writer_xhs_system.txt        # XHSWriterAgent系统提示词
│   ├── writer_xhs_user.txt          # XHSWriterAgent用户提示词模板
│   ├── writer_mp_dy_system.txt      # MPWriterAgent系统提示词
│   └── writer_mp_dy_user.txt        # MPWriterAgent用户提示词模板
├── utils/                     # 工具和客户端（解耦组件）
│   ├── __init__.py
│   ├── llm_client.py          # LLM客户端封装（支持多模型切换）
│   ├── logger.py              # 日志工具（统一日志格式）
│   ├── memory_store.py        # 会话记忆存储（上下文管理）
│   ├── search_tools.py        # 搜索工具集成
│   └── streaming.py           # 流式响应工具
├── requirements.txt           # Python依赖
├── Dockerfile                 # Docker部署配置
└── README.md                  # 项目说明文档
```

---

## 🔧 核心设计原则

### 1. 架构简化原则
- **双目录结构**：仅 `agents/` 和 `apis/` 两个核心目录，业务逻辑清晰分离。
- **单一职责**：每个 Agent 专注一个特定领域，避免功能重叠。
- **最小化接口**：只提供必要的 API 端点，避免过度设计。

### 2. 配置解耦设计

#### LLM 客户端解耦
```python
# utils/llm_client.py 支持多模型切换
class LLMClient:
    def __init__(self, provider: str = "openai"):
        """
        初始化LLM客户端
        Args:
            provider: 模型提供商 (openai, claude, gemini, qwen等)
        """
        self.provider = provider
        self.client = self._init_client()
    
    def _init_client(self):
        # 根据配置文件动态初始化不同的LLM客户端
        pass
    
    async def stream_chat(self, messages: List[dict]) -> AsyncGenerator:
        # 统一的流式聊天接口，支持所有模型
        pass
```

#### 配置文件结构
```yaml
# config/llm_config.yaml
llm:
  default_provider: "openai"
  providers:
    openai:
      api_key: "${OPENAI_API_KEY}"
      model: "gpt-4"
      base_url: "https://api.openai.com/v1"
    claude:
      api_key: "${CLAUDE_API_KEY}"
      model: "claude-3-sonnet-20240229"
    qwen:
      api_key: "${QWEN_API_KEY}"
      model: "qwen-max"
      base_url: "https://dashscope.aliyuncs.com/api/v1"

# config/app_config.yaml
app:
  name: "BluePlan Research"
  version: "1.0.0"
  debug: true
  log_level: "INFO"
api:
  host: "0.0.0.0"
  port: 8000
  cors_origins: ["*"]
memory:
  store_type: "memory"  # memory, redis, file
  max_sessions: 1000
```

### 3. 提示词双组设计

每个 Agent 的提示词分为两个文件：

#### 系统提示词（固定）
```txt
# prompts/supervisor_system.txt
你是BluePlan的SupervisorAgent，负责任务规划和Agent调度。

## 核心职责
1. 解析用户请求，识别意图和需求
2. 制定Smart Plan执行计划
3. 调度相应的子Agent执行任务
4. 整合所有Agent的结果，生成最终报告

## 输出规范
- 使用content_type标签标记输出内容
- plan_step: 执行计划步骤
- thought: 思考过程
- final_answer: 最终整合报告

## 调度规则
- 用户洞察问题 -> UserInsightAgent
- 内容策略问题 -> ContentStrategyAgent  
- 视觉优化问题 -> DiagnoseAvatarAgent
- 文案创作问题 -> XHSWriter/MPWriter
- 事实查证问题 -> FactCollectionAgent
```

#### 用户提示词（动态参数）
```txt
# prompts/supervisor_user.txt
## 用户请求分析
用户ID: {user_id}
会话ID: {session_id}

## 用户查询
{query}

## 背景信息
- 账号名称: {account_name}
- 目标平台: {platform}  
- 目标受众: {target_audience}
- 历史内容: {previous_content}

## 交互类型
{interaction_type}

## 上下文信息
{interaction_context}

请分析这个请求，制定Smart Plan并开始执行。
```

### 4. 主启动类设计

```python
# main.py
class BluePlanApp:
    """
    BluePlan主应用类
    支持本地调试模式和API服务模式
    """
    
    def __init__(self):
        """初始化BluePlan应用"""
        self.logger = self._setup_logger()
        self.config = self._load_config()
        self.supervisor = SupervisorAgent()
        
    def _setup_logger(self):
        """配置日志系统"""
        pass
        
    def _load_config(self):
        """加载系统配置"""
        pass
    
    async def chat_debug(self, query: str, background: dict = None):
        """
        本地调试模式：直接与SupervisorAgent对话
        Args:
            query: 用户查询
            background: 背景信息
        """
        self.logger.info(f"🚀 开始处理查询: {query}")
        
        # 构造请求数据
        request_data = {
            "query": query,
            "background": background or {},
            "interaction_type": "full_analysis"
        }
        
        # 流式输出结果
        async for event in self.supervisor.process_request(request_data):
            self._print_event(event)
            
    def _print_event(self, event: dict):
        """格式化打印事件信息"""
        agent = event.get("agent_source", "System")
        content_type = event.get("payload", {}).get("content_type", "unknown")
        data = event.get("payload", {}).get("data", "")
        
        print(f"[{agent}] ({content_type}): {data}")
        
    def start_api_server(self):
        """启动FastAPI服务器"""
        import uvicorn
        from apis.app import app
        
        self.logger.info("🌐 启动BluePlan API服务器...")
        uvicorn.run(
            app, 
            host=self.config.api.host,
            port=self.config.api.port,
            log_level=self.config.app.log_level.lower()
        )

if __name__ == "__main__":
    import asyncio
    import sys
    
    app = BluePlanApp()
    
    if len(sys.argv) > 1 and sys.argv[1] == "debug":
        # 本地调试模式
        query = "我的小红书账号流量上不去，帮我分析下原因"
        background = {
            "account_name": "我的手工烘焙坊",
            "platform": "小红书"
        }
        
        asyncio.run(app.chat_debug(query, background))
    else:
        # 启动API服务器
        app.start_api_server()
```

### 5. 日志和调试规范

#### 日志设计
```python
# utils/logger.py
import logging
from datetime import datetime

class BluePlanLogger:
    """统一的日志管理类"""
    
    def __init__(self, name: str, level: str = "INFO"):
        self.logger = logging.getLogger(name)
        self.logger.setLevel(getattr(logging, level.upper()))
        
        # 控制台输出格式
        console_handler = logging.StreamHandler()
        formatter = logging.Formatter(
            '[%(asctime)s] %(name)s [%(levelname)s] %(message)s',
            datefmt='%Y-%m-%d %H:%M:%S'
        )
        console_handler.setFormatter(formatter)
        self.logger.addHandler(console_handler)
    
    def info(self, message: str):
        """信息日志"""
        self.logger.info(f"📋 {message}")
        
    def debug(self, message: str):
        """调试日志"""
        self.logger.debug(f"🔍 {message}")
        
    def warning(self, message: str):
        """警告日志"""
        self.logger.warning(f"⚠️ {message}")
        
    def error(self, message: str):
        """错误日志"""
        self.logger.error(f"❌ {message}")
        
    def agent_start(self, agent_name: str, task: str):
        """Agent开始执行日志"""
        self.logger.info(f"🤖 {agent_name} 开始执行: {task}")
        
    def agent_complete(self, agent_name: str, result_type: str):
        """Agent完成执行日志"""
        self.logger.info(f"✅ {agent_name} 完成执行，输出类型: {result_type}")
```

### 6. 中文注释规范

#### 类和方法注释
```python
class SupervisorAgent(BaseAgent):
    """
    监督者智能体：负责任务规划和子Agent调度
    
    主要功能：
    1. 解析用户请求，识别意图和需求类型
    2. 制定Smart Plan执行计划
    3. 根据计划调度相应的子Agent
    4. 整合所有Agent的输出结果
    5. 生成最终的综合报告
    
    Attributes:
        config: 系统配置对象
        logger: 日志记录器
        sub_agents: 子Agent实例字典
        memory_store: 会话记忆存储
    """
    
    def __init__(self, config: Settings):
        """
        初始化监督者Agent
        
        Args:
            config: 系统配置对象，包含LLM配置、提示词路径等
        """
        super().__init__(config)
        self.logger.info("正在初始化SupervisorAgent...")
        
    async def process_request(self, request_data: dict) -> AsyncGenerator[dict, None]:
        """
        处理用户请求，生成流式响应
        
        Args:
            request_data: 用户请求数据，包含query、background等字段
            
        Yields:
            dict: 流式事件，包含event_type、agent_source、payload等字段
            
        流程：
        1. 解析用户意图
        2. 制定执行计划
        3. 调度子Agent执行
        4. 整合结果输出
        """
        self.logger.agent_start("SupervisorAgent", f"处理查询: {request_data.get('query', '')}")
        
        # 步骤1: 意图识别和计划制定
        async for event in self._create_smart_plan(request_data):
            yield event
            
        # 步骤2: 执行计划
        async for event in self._execute_plan(request_data):
            yield event
            
        self.logger.agent_complete("SupervisorAgent", "final_answer")
```

#### 关键逻辑注释
```python
async def _create_smart_plan(self, request_data: dict) -> AsyncGenerator[dict, None]:
    """
    创建Smart Plan执行计划
    
    根据用户查询的类型和复杂度，智能决定需要调用哪些子Agent：
    - 账号分析问题 -> UserInsightAgent
    - 内容策略问题 -> ContentStrategyAgent + FactCollectionAgent  
    - 视觉优化问题 -> DiagnoseAvatarAgent
    - 文案创作问题 -> XHSWriter/MPWriter
    """
    
    # 构造提示词：系统提示词 + 用户动态参数
    system_prompt = self._load_system_prompt("supervisor")
    user_prompt = self._build_user_prompt("supervisor", request_data)
    
    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_prompt}
    ]
    
    # 流式生成计划步骤
    async for chunk in self.llm_client.stream_chat(messages):
        # 解析内容类型并发送事件
        event = {
            "event_type": "llm_chunk",
            "agent_source": "SupervisorAgent",
            "timestamp": datetime.now().isoformat(),
            "payload": {
                "content_type": "plan_step",  # 计划步骤类型
                "data": chunk
            }
        }
        yield event
```

---

## 📦 安装与启动

### 安装依赖

```bash
pip install -r requirements.txt
```

### 配置环境变量

```bash
# 复制配置文件模板
cp config/llm_config.yaml.example config/llm_config.yaml
cp config/app_config.yaml.example config/app_config.yaml

# 设置LLM API密钥
export OPENAI_API_KEY="your_openai_api_key"
export CLAUDE_API_KEY="your_claude_api_key"
```

### 启动方式

#### 方式1: 本地调试模式
```bash
# 直接与SupervisorAgent对话，便于开发调试
python main.py debug
```

#### 方式2: API服务模式
```bash
# 启动FastAPI服务器，对外提供API接口
python main.py

# 或使用uvicorn直接启动
uvicorn apis.app:app --reload --host 0.0.0.0 --port 8000
```

#### 方式3: Docker部署
```bash
docker build -t blueplan-research .
docker run -p 8000:8000 -e OPENAI_API_KEY=your_key blueplan-research
```

-----

## 🏗️ 核心技术实现细节

### 1. 数据模型定义 (Pydantic Schemas)

```python
# apis/schemas.py
from typing import List, Dict, Any, Optional, Union
from pydantic import BaseModel, Field
from enum import Enum

class InteractionType(str, Enum):
    """交互类型枚举"""
    FULL_ANALYSIS = "full_analysis"      # 完整分析
    QUICK_SUGGESTION = "quick_suggestion" # 快速建议
    WRITE_COPY = "write_copy"            # 文案创作
    DIAGNOSE_AVATAR = "diagnose_avatar"   # 视觉诊断

class ChatRequest(BaseModel):
    """聊天请求模型"""
    user_id: str = Field(..., description="用户ID", regex=r"^[a-zA-Z0-9_-]+$")
    session_id: str = Field(..., description="会话ID", regex=r"^[a-zA-Z0-9_-]+$")
    request_data: RequestData = Field(..., description="请求数据")

class StreamEvent(BaseModel):
    """流式事件模型"""
    event_type: EventType = Field(..., description="事件类型")
    agent_source: str = Field(..., description="Agent来源")
    timestamp: str = Field(..., description="时间戳")
    payload: EventPayload = Field(..., description="事件负载")
```

### 2. Base Agent 基类设计

```python
# agents/base_agent.py
from abc import ABC, abstractmethod
from typing import AsyncGenerator, Dict, Any, List, Optional

class BaseAgent(ABC):
    """
    Agent基类：定义所有智能体的通用接口和行为
    
    核心功能：
    1. 统一的流式输出接口
    2. 提示词加载和渲染
    3. LLM客户端管理
    4. 错误处理和重试机制
    """
    
    def __init__(self, config: 'Settings', agent_name: str):
        """初始化Agent基类"""
        self.config = config
        self.agent_name = agent_name
        self.logger = BluePlanLogger(f"Agent.{agent_name}")
        self.llm_client = LLMClient(config.llm.default_provider)
        
    @abstractmethod
    async def process_request(
        self, 
        request_data: Dict[str, Any],
        context: Optional[Dict[str, Any]] = None
    ) -> AsyncGenerator[StreamEvent, None]:
        """处理请求的抽象方法，子类必须实现"""
        pass
    
    async def _emit_event(
        self, 
        event_type: EventType, 
        content_type: ContentType = None,
        data: str = ""
    ) -> StreamEvent:
        """发送流式事件的通用方法"""
        return StreamEvent(
            event_type=event_type,
            agent_source=self.agent_name,
            timestamp=datetime.now().isoformat(),
            payload=EventPayload(content_type=content_type, data=data)
        )
```

### 3. 错误处理和容错机制

```python
# utils/error_handler.py
class ErrorCode(str, Enum):
    """错误代码枚举"""
    AGENT_TIMEOUT = "AGENT_TIMEOUT"
    LLM_RATE_LIMIT = "LLM_RATE_LIMIT"
    LLM_API_ERROR = "LLM_API_ERROR"
    INVALID_REQUEST = "INVALID_REQUEST"

class BluePlanException(Exception):
    """BluePlan自定义异常基类"""
    def __init__(self, error_code: ErrorCode, message: str, details: Optional[Dict] = None):
        self.error_code = error_code
        self.message = message
        self.details = details or {}

class ErrorHandler:
    """统一错误处理器"""
    def handle_exception(self, e: Exception, context: str = "") -> Dict[str, Any]:
        """处理异常并返回错误信息"""
        # 错误分类和处理逻辑
        pass
```

### 4. 性能和并发控制

```python
# utils/concurrency.py
class ConcurrencyManager:
    """并发控制管理器"""
    
    def __init__(self, max_concurrent_users: int = 100):
        self.max_concurrent_users = max_concurrent_users
        self.user_semaphore = asyncio.Semaphore(max_concurrent_users)
        self.active_sessions: Dict[str, Dict[str, Any]] = {}
    
    async def acquire_user_slot(self, user_id: str, session_id: str) -> bool:
        """获取用户槽位，防止系统过载"""
        pass
    
    def get_system_status(self) -> Dict[str, Any]:
        """获取系统负载状态"""
        return {
            "active_sessions": len(self.active_sessions),
            "available_slots": self.user_semaphore._value
        }
```

### 5. 测试策略和用例设计

```python
# tests/test_agents.py
class TestSupervisorAgent:
    """SupervisorAgent测试用例"""
    
    async def test_process_request_flow(self):
        """测试完整的请求处理流程"""
        # 测试意图识别 -> 计划生成 -> Agent调度 -> 结果整合
        pass
    
    async def test_content_type_labeling(self):
        """测试内容类型标签的正确性"""
        # 验证输出的content_type标签符合规范
        pass
    
    async def test_error_recovery(self):
        """测试错误恢复机制"""
        # 测试Agent执行失败时的恢复策略
        pass
```

### 6. 安全和鉴权机制

```python
# apis/security.py
class SecurityManager:
    """API安全管理器"""
    
    def __init__(self, secret_key: str, rate_limit_per_minute: int = 60):
        self.secret_key = secret_key
        self.rate_limit_per_minute = rate_limit_per_minute
        self.rate_limit_store: Dict[str, list] = {}
    
    async def verify_token(self, credentials) -> Dict[str, Any]:
        """验证JWT令牌"""
        pass
    
    def check_rate_limit(self, user_id: str) -> bool:
        """检查API调用频率限制"""
        pass
    
    def validate_request_data(self, request_data: Dict[str, Any]) -> bool:
        """验证请求数据安全性，防止注入攻击"""
        pass
```

### 7. 监控和指标系统

```python
# utils/metrics.py
class MetricsCollector:
    """业务指标收集器"""
    
    def __init__(self):
        self.request_count = defaultdict(int)
        self.response_times = defaultdict(list)
        self.agent_success_rate = defaultdict(lambda: {"success": 0, "failure": 0})
        self.content_type_stats = defaultdict(int)
    
    def record_agent_execution(self, agent_name: str, success: bool, duration: float):
        """记录Agent执行指标"""
        pass
    
    def get_metrics_summary(self) -> Dict[str, Any]:
        """获取系统运行指标摘要"""
        return {
            "avg_response_times": {...},
            "agent_success_rates": {...},
            "content_generation_stats": {...}
        }
```

### 8. 工具集成框架

```python
# utils/search_tools.py
class SearchToolManager:
    """搜索工具管理器"""
    
    def __init__(self, config: dict):
        self.config = config
        self.tools = self._init_tools()
    
    async def search_xiaohongshu_cases(self, query: str) -> List[Dict[str, Any]]:
        """搜索小红书爆款案例"""
        pass
    
    async def search_trend_topics(self, platform: str, category: str) -> List[str]:
        """搜索平台热门话题"""
        pass
    
    async def analyze_competitor_content(self, account_name: str) -> Dict[str, Any]:
        """分析竞品账号内容"""
        pass
```

### 9. 会话状态管理

```python
# utils/memory_store.py
class MemoryStore:
    """会话记忆存储管理器"""
    
    def __init__(self, config: dict):
        self.store_type = config.get("store_type", "memory")
        self.max_sessions = config.get("max_sessions", 1000)
        self.session_timeout = config.get("session_timeout", 3600)  # 1小时
        
        if self.store_type == "redis":
            self.redis_client = self._init_redis()
        else:
            self.memory_cache: Dict[str, Dict[str, Any]] = {}
    
    async def save_session_context(
        self, 
        session_id: str, 
        context: Dict[str, Any]
    ) -> bool:
        """保存会话上下文"""
        pass
    
    async def get_session_context(self, session_id: str) -> Optional[Dict[str, Any]]:
        """获取会话上下文"""
        pass
    
    async def update_agent_history(
        self, 
        session_id: str, 
        agent_name: str, 
        result: Dict[str, Any]
    ):
        """更新Agent执行历史"""
        pass
    
    def cleanup_expired_sessions(self):
        """清理过期会话"""
        pass
```

### 10. 依赖和配置管理

```python
# requirements.txt - 详细版本控制
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic[email]==2.5.0
openai==1.6.1
anthropic==0.8.1
redis==5.0.1
PyYAML==6.0.1
python-multipart==0.0.6
python-jose[cryptography]==3.3.0
aiofiles==23.2.1
httpx==0.25.2
pytest==7.4.3
pytest-asyncio==0.21.1
pytest-cov==4.1.0
```

```yaml
# config/app_config.yaml - 完整配置示例
app:
  name: "BluePlan Research"
  version: "1.0.0"
  debug: false
  log_level: "INFO"
  
api:
  host: "0.0.0.0"
  port: 8000
  cors_origins: ["*"]
  max_request_size: 10485760  # 10MB
  timeout: 300  # 5分钟
  
security:
  jwt_secret_key: "${JWT_SECRET_KEY}"
  access_token_expire_minutes: 30
  rate_limit_per_minute: 60
  enable_auth: true
  
memory:
  store_type: "redis"  # memory, redis, file
  redis_url: "${REDIS_URL}"
  max_sessions: 10000
  session_timeout: 7200  # 2小时
  
performance:
  max_concurrent_users: 200
  max_concurrent_agents: 100
  response_timeout: 180
  llm_timeout: 60
  
monitoring:
  enable_metrics: true
  metrics_port: 9090
  log_file: "/app/logs/blueplan.log"
  log_rotation: "1 day"
```

### 11. OpenAPI文档生成

```python
# apis/app.py - FastAPI配置增强
from fastapi import FastAPI, Depends, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.openapi.docs import get_swagger_ui_html
from fastapi.openapi.utils import get_openapi

def create_app() -> FastAPI:
    """创建FastAPI应用实例"""
    app = FastAPI(
        title="BluePlan Research API",
        description="面向社交媒体内容创作的Gen-AI Agent系统",
        version="1.0.0",
        docs_url="/docs",
        redoc_url="/redoc",
        openapi_url="/openapi.json"
    )
    
    # 自定义OpenAPI schema
    def custom_openapi():
        if app.openapi_schema:
            return app.openapi_schema
        
        openapi_schema = get_openapi(
            title="BluePlan Research API",
            version="1.0.0",
            description="智能社媒内容创作平台API文档",
            routes=app.routes,
        )
        
        # 添加认证信息
        openapi_schema["components"]["securitySchemes"] = {
            "BearerAuth": {
                "type": "http",
                "scheme": "bearer",
                "bearerFormat": "JWT"
            }
        }
        
        app.openapi_schema = openapi_schema
        return app.openapi_schema
    
    app.openapi = custom_openapi
    return app
```

### 12. 健康检查和监控端点

```python
# apis/health.py
from fastapi import APIRouter, Depends
from typing import Dict, Any
from utils.metrics import MetricsCollector
from utils.concurrency import ConcurrencyManager

router = APIRouter(prefix="/health", tags=["健康检查"])

@router.get("/")
async def health_check() -> Dict[str, str]:
    """基础健康检查"""
    return {"status": "healthy", "service": "BluePlan Research"}

@router.get("/ready")
async def readiness_check() -> Dict[str, Any]:
    """就绪检查 - K8s使用"""
    # 检查关键服务是否就绪
    checks = {
        "llm_client": await _check_llm_connection(),
        "memory_store": await _check_memory_store(),
        "config": _check_config_validity()
    }
    
    all_ready = all(checks.values())
    
    return {
        "ready": all_ready,
        "checks": checks,
        "timestamp": datetime.now().isoformat()
    }

@router.get("/metrics")
async def get_metrics(
    metrics: MetricsCollector = Depends(),
    concurrency: ConcurrencyManager = Depends()
) -> Dict[str, Any]:
    """获取系统指标"""
    return {
        "business_metrics": metrics.get_metrics_summary(),
        "system_status": concurrency.get_system_status(),
        "timestamp": datetime.now().isoformat()
    }
```

### 13. 生产环境部署优化

```dockerfile
# Dockerfile - 生产级优化
FROM python:3.11-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-slim AS runtime

# 安全配置
RUN groupadd -r blueplan && useradd -r -g blueplan blueplan
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /root/.local /home/blueplan/.local
COPY . .

# 设置权限
RUN chown -R blueplan:blueplan /app
USER blueplan

# 环境变量
ENV PATH=/home/blueplan/.local/bin:$PATH
ENV PYTHONPATH=/app
ENV PYTHONUNBUFFERED=1

# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000
CMD ["python", "main.py"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  blueplan-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET_KEY=${JWT_SECRET_KEY}
    depends_on:
      - redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '0.5'
        reservations:
          memory: 512M
          cpus: '0.25'
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

volumes:
  redis_data:
```

-----

## 🔧 可拓展功能

- **[开发中] LangGraph 图节点可视化**: 实时监控 Agent 状态和任务流转。
- **多模型兼容**: 已支持 OpenAI, Claude, Gemini, 通义千问等主流 LLM。
- **[选配] 企业级部署**: 提供完整的 Dockerfile 和 K8s 配置，支持水平扩展。
- **记忆存储**: 支持 Memory, Redis, File 多种会话存储方式。
- **插件系统**: 支持自定义 Agent 和工具的热插拔。

-----

## 📄 License

MIT License

-----

**🧠 BluePlan Research：让每一次内容创作都更智能、更精准。**

```

### 主要优化点总结：

1.  **明确的 API 契约**：
    * `POST /chat` 的请求体 (`Request Body`) 被详细定义，包含了 `user_id`, `session_id` 和结构化的 `request_data`，这为代码生成提供了清晰的 Pydantic 模型（`schemas.py`）基础。
    * 明确指出响应是**流式**的，避免了生成一个一次性返回所有内容的函数。

2.  **详细的流式协议**：
    * 创建了 `流式输出协议` 章节，这是本次优化的核心。
    * 定义了统一的事件结构 (`event_type`, `agent_source`, `payload` 等)，让前端或任何客户端都能简单一致地处理信息。
    * **关键的 `content_type` 标签**：我将你提到的 "打点消息、洞察消息、人设消息" 等概念，具体化为 `llm_chunk` 事件负载中的 `content_type` 标签（如 `plan_step`, `insight`, `persona`, `case_study`），并清晰地将其与各个 Agent 的职责关联起来。这使得代码生成工具可以为每个 Agent 的输出逻辑创建带有特定标签的流式 `yield` 语句。

3.  **强化的 Agent 职责说明**：
    * `Agent 模块详解` 表格被扩充，增加了 "核心输入" 和 "核心输出（流式 `content_type`）" 两列。
    * 这直接告诉了代码生成器：`UserInsight` Agent 在编码时，其输出应该被标记为 `persona` 或 `insight`；`SupervisorAgent` 的输出应标记为 `plan_step` 或 `final_answer`。这种关联性极大地提高了代码生成的准确性。

4.  **动态的执行示例**：
    * `Smart Plan 执行示例` 不再是静态的步骤列表，而是模拟了真实的、带标签的流式 JSON 输出序列。这不仅是一个功能示例，更是一个**实现模板**，直观地展示了整个系统如何工作。

5.  **结构微调**：
    * 更新了项目结构图，加入了 `schemas.py` 等关键文件。
    * 调整了标题和部分描述，使其更具技术文档的专业性和清晰度。

这份优化后的文档为 AI 代码生成工具提供了足够详尽、结构化的信息，能够引导它生成更符合你设想的、包含复杂流式处理和 Agent 交互逻辑的代码框架。
```
