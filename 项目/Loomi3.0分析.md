# Loomi 3.0 项目深度分析报告

## 项目概述

Loomi 3.0 是一个基于多Agent协作的内容创作流水线系统，采用面向上下文的编程（Context-Oriented Programming）设计哲学。系统通过Concierge、Orchestrator和Action三个核心Agent实现智能化的内容创作流程。

---

## 1️⃣ 设计模式理解与选择能力分析

### 1.1 核心设计模式

#### 1.1.1 面向上下文的编程模式
**设计理念**：系统不将Agent视为独立黑盒，而是作为共享、动态数据环境中的上下文消费者和信息生产者。

**实现方式**：
- **WorkSpace**：作为数据中心，管理所有结构化数据（notes、tactics）
- **ConversationManager**：作为记忆中心，记录所有交互流水
- **Context Builders**：为每个Agent量身定制上下文，精确控制信息流动

#### 1.1.2 分层架构模式
```
┌─────────────────┐
│   Agents Layer  │  ← 行为层
├─────────────────┤
│ Context Layer   │  ← 上下文构建层
├─────────────────┤
│   Core Layer    │  ← 数据层
└─────────────────┘
```

#### 1.1.3 策略模式（Strategy Pattern）
在`llm/model_base.py`中实现了模型提供商的策略模式：
```python
class ModelProvider(ABC):
    @abstractmethod
    def call_llm(self, system_prompt: str, user_prompt: str, **kwargs) -> str:
        pass
```

### 1.2 模块功能与联系

#### 1.2.1 Agents模块
- **Concierge**：前台接待，负责理解用户意图和任务分流
- **Orchestrator**：核心大脑，采用ReAct模式进行任务规划
- **Action**：执行单元，处理具体的原子化指令

#### 1.2.2 Core模块
- **WorkSpace**：数据中心，管理notes和tactics
- **ConversationManager**：记忆中心，分层存储对话历史
- **Context Builder**：上下文构建器，为不同Agent定制上下文
- **Notes Extractor**：鲁棒的notes提取器

#### 1.2.3 LLM模块
- **ModelManager**：模型管理器，支持多种模型提供商
- **Provider抽象**：统一的模型调用接口

### 1.3 设计模式选择原因

1. **面向上下文编程**：确保每个Agent在信息干扰最小、上下文最相关的环境中工作
2. **分层架构**：实现数据与行为的分离，提高系统可维护性
3. **策略模式**：支持多种LLM模型的无缝切换
4. **鲁棒性设计**：通过多级解析策略处理LLM输出的各种边界情况

---

## 2️⃣ 多Agent协作机制分析

### 2.1 Agent角色分工

#### 2.1.1 Concierge（前台）
**职责**：
- 接收用户输入，理解初步意图
- 处理简单问答，避免不必要的复杂处理
- 将复杂任务结构化传递给Orchestrator

**上下文设计**：
```python
def build_concierge_context(user_msg: str, workspace, conversation) -> str:
    # 包含：聊天历史、Orchestrator时间线、notes摘要
    # 特点：宽而浅，快速决策
```

#### 2.1.2 Orchestrator（编排员）
**职责**：
- 系统的核心大脑，采用ReAct模式
- 制定和调整高层策略（tactics）
- 调用Action完成具体步骤

**工作流程**：
```
Observe → Think → Act → Observe → ...
```

#### 2.1.3 Action（执行者）
**职责**：
- 执行原子化、具体的指令
- 纯粹的执行单元，高度专注

**上下文特点**：
```python
def build_action_context(step: Dict, workspace, conversation) -> str:
    # 极度精简：只包含指令和引用的note全文
    # 特点：窄而深，信息隔离
```

### 2.2 任务分配机制

#### 2.2.1 智能分流
- Concierge根据任务复杂度决定是否调用Orchestrator
- 简单任务直接回复，复杂任务进入编排流程

#### 2.2.2 动态配额管理
```python
# 动态计算命令配额：初始12条，每次反馈增加3条
base_commands = 12
feedback_bonus = max(0, call_count - 1) * 3
max_commands = base_commands + feedback_bonus
```

### 2.3 状态隔离与结果合并

#### 2.3.1 状态隔离
- **WorkSpace**：管理全局状态（notes、tactics）
- **ConversationManager**：管理对话状态和历史
- **Context Builders**：为每个Agent提供隔离的上下文视图

#### 2.3.2 结果合并
- **Notes系统**：通过结构化数据实现Agent间的异步通信
- **执行记录**：详细记录每个Action的执行结果和产出

### 2.4 Planner-Subagent结构运行细节

#### 2.4.1 Orchestrator作为Planner
```python
def process_task(self, task_message: str, execution_log: Optional[List[Dict]] = None) -> Dict[str, Any]:
    # 1. 构建上下文
    context = build_orchestrator_context(...)
    
    # 2. 调用LLM进行规划
    response = call_llm(...)
    
    # 3. 解析execute命令
    execute_commands = self._extract_execute_commands(response)
    
    # 4. 记录执行信息
    self._record_execution(execute_commands, observe, think)
```

#### 2.4.2 Action作为Subagent
```python
def execute(action_type: str, instruction: str, workspace, conversation, orchestrator=None) -> Dict[str, Any]:
    # 1. 获取Action特定的prompt
    system_prompt = ACTION_PROMPTS[action_type]
    
    # 2. 构建精简上下文
    user_prompt = build_action_context(step, workspace, conversation)
    
    # 3. 执行Action
    output = call_llm(...)
    
    # 4. 提取并创建notes
    notes_created = extract_and_create_notes(output, step_id, workspace)
```

---

## 3️⃣ 上下文与记忆架构设计分析

### 3.1 记忆架构层次

#### 3.1.1 短期记忆（ConversationManager）
```python
class ConversationManager:
    def __init__(self):
        self.history: List[Dict[str, Any]] = []  # 完整对话历史
        self._orchestrator_calls: List[Dict[str, Any]] = []  # Orchestrator调用历史
        self._active_orchestrator_index: Optional[int] = None  # 当前活跃调用
```

**特点**：
- 线性存储，按时间顺序记录
- 分层访问，提供不同粒度的历史数据
- 支持快速检索和状态管理

#### 3.1.2 长期记忆（WorkSpace）
```python
class WorkSpace:
    def __init__(self):
        self.notes: Dict[str, Dict[str, str]] = {}  # 结构化知识库
        self._note_counters: Dict[str, int] = {}  # 自动编号
        self.tactics: Optional[str] = None  # Orchestrator思考空间
```

**特点**：
- 结构化存储，支持类型分类
- 自动编号，确保唯一性
- 支持引用和关联

### 3.2 记忆控制机制

#### 3.2.1 写入控制
- **自动写入**：Action执行后自动提取notes并写入WorkSpace
- **手动写入**：Concierge可以手动创建material类型的notes
- **状态管理**：支持notes的反思评审状态（star、archive、trash）

#### 3.2.2 清理机制
```python
def get_referenceable_notes(self) -> Dict[str, Dict[str, str]]:
    """获取可引用的notes（排除trash状态）"""
    return {
        note_id: note_data 
        for note_id, note_data in self.notes.items() 
        if note_data.get("review_status") != "trash"
    }
```

### 3.3 上下文构建策略

#### 3.3.1 精确性控制
- **Concierge上下文**：宽而浅，快速决策
- **Orchestrator上下文**：深而战略性，支持规划
- **Action上下文**：窄而深，高度专注

#### 3.3.2 动态性控制
- 上下文不是静态的，每次ReAct循环都会更新
- 支持实时补充新需求
- 动态调整信息粒度

### 3.4 新型记忆组织方式

#### 3.4.1 Agentic Memory
系统实现了类似Agentic Memory的设计：
- **WorkSpace**：作为Agent的"记忆宫殿"
- **Notes系统**：实现Agent间的知识共享
- **Context Builders**：实现记忆的按需访问

#### 3.4.2 KV Memory
通过notes的键值对结构实现：
```python
# 键：note_id（如resonant1）
# 值：包含type、content、source的结构化数据
self.notes[note_id] = {
    "type": note_type,
    "content": content,
    "source": source
}
```

---

## 4️⃣ 工具调用与API注入机制分析

### 4.1 工具动态注册机制

#### 4.1.1 模型提供商注册
```python
class ModelManager:
    def __init__(self):
        self._providers: Dict[str, ModelProvider] = {}
        self._current_provider: Optional[ModelProvider] = None
    
    def register_provider(self, provider: ModelProvider):
        self._providers[provider.name] = provider
```

#### 4.1.2 Action工具注册
```python
# prompts/action_prompts.py
ACTION_PROMPTS = {
    "resonant": resonant_prompt,
    "persona": persona_prompt,
    "hitpoint": hitpoint_prompt,
    # ... 更多Action类型
}
```

### 4.2 Schema解析机制

#### 4.2.1 XML标签解析
系统使用XML标签作为工具调用的schema：
```python
def _extract_execute_commands(self, response: str) -> List[Dict[str, str]]:
    patterns = [
        r'<execute\s+action="([^"]+)"\s+instruction="([^"]+)"\s*/?>',
        r'<execute\s+action=([^\s]+)\s+instruction=(.*?)(?=\s*/?>)',
        # ... 多种格式支持
    ]
```

#### 4.2.2 鲁棒性解析
```python
class RobustNotesExtractor:
    def extract_notes(self, response: str) -> Dict[str, List[Tuple[str, str]]]:
        # 多级解析策略：完美匹配 → 修复匹配 → 模糊匹配
        for tag_type in self.known_types:
            extracted_notes = self._extract_notes_for_type(response, tag_type)
```

### 4.3 工具选择与参数对齐

#### 4.3.1 智能工具选择
- Orchestrator根据任务需求选择合适的Action
- 支持多种Action类型的组合使用
- 动态调整Action的执行参数

#### 4.3.2 参数对齐机制
```python
def build_action_context(step: Dict, workspace, conversation) -> str:
    # 1. 解析@引用
    refs = re.findall(r'@([a-zA-Z_]+\d+)', step["instruction"])
    
    # 2. 自动添加material引用（写作类Action）
    if step["action"] in writing_actions:
        material_notes = [note_id for note_id in referenceable_notes.keys() 
                         if note_id.startswith('material')]
    
    # 3. 展开所有参考资料
    for ref in refs:
        note_data = referenceable_notes.get(ref)
        if note_data:
            parts.append(f"@{ref}:")
            parts.append(note_data["content"])
```

### 4.4 调用Fallback机制

#### 4.4.1 多级解析策略
```python
def _extract_notes_for_type(self, response: str, tag_type: str) -> List[Tuple[str, str]]:
    # 第一级：完美匹配
    perfect_notes = self._extract_perfect_matches(response, tag_type)
    
    # 第二级：修复匹配（只有在完美匹配失败时才尝试）
    if not perfect_notes:
        repaired_notes = self._extract_with_repair(response, tag_type)
    
    # 第三级：模糊匹配（最后手段）
    if not notes:
        fuzzy_notes = self._extract_fuzzy_matches(response, tag_type)
```

#### 4.4.2 错误处理机制
```python
def execute(action_type: str, instruction: str, workspace, conversation, orchestrator=None) -> Dict[str, Any]:
    try:
        # 检查action类型是否有效
        if action_type not in ACTION_PROMPTS:
            raise ValueError(f"未知的Action类型: {action_type}")
        
        # 执行Action
        output = call_llm(**llm_kwargs)
        
        return {
            "success": True,
            "output": output,
            "notes_created": notes_created
        }
    except Exception as e:
        return {
            "success": False,
            "error": str(e),
            "notes_created": []
        }
```

### 4.5 统一工具执行接口

#### 4.5.1 抽象接口设计
```python
def call_llm(system_prompt: str, user_prompt: str, **kwargs) -> str:
    """统一的LLM调用接口"""
    current_provider = model_manager.get_current_provider()
    if not current_provider:
        raise ValueError("未设置当前模型提供商")
    
    return current_provider.call_llm(system_prompt, user_prompt, **kwargs)
```

#### 4.5.2 参数标准化
- 支持temperature、max_output_tokens等标准参数
- 支持thinking_budget等特殊参数
- 支持enable_websearch等功能开关

---

## 5️⃣ 控制流与调度能力分析

### 5.1 DAG工作流设计能力

#### 5.1.1 ReAct循环模式
```python
# Orchestrator的ReAct工作流程
def process_task(self, task_message: str, execution_log: Optional[List[Dict]] = None) -> Dict[str, Any]:
    # Observe: 构建上下文观察全局状态
    context = build_orchestrator_context(...)
    
    # Think: LLM进行思考决策
    response = call_llm(...)
    
    # Act: 执行具体Action
    execute_commands = self._extract_execute_commands(response)
```

#### 5.1.2 任务依赖关系管理
- **材料依赖**：Action通过@引用建立依赖关系
- **执行顺序**：Orchestrator控制Action的执行顺序
- **结果传递**：通过notes系统实现结果传递

### 5.2 幂等执行能力

#### 5.2.1 状态管理
```python
class ConversationManager:
    def __init__(self):
        self._orchestrator_running: bool = False  # 运行状态
        self._active_orchestrator_index: Optional[int] = None  # 活跃调用索引
```

#### 5.2.2 执行记录
```python
def add_execution_record(self, step: str, notes_created: Optional[List[str]] = None, 
                        observe: Optional[str] = None, think: Optional[str] = None, 
                        actions_detail: Optional[List[Dict]] = None):
    """记录执行信息，支持幂等性检查"""
    entry = {
        "type": "execution",
        "step": step,
        "notes_created": notes_created or [],
        "observe": observe,
        "think": think,
        "actions_detail": actions_detail
    }
```

### 5.3 失败回滚机制

#### 5.3.1 错误捕获与处理
```python
def execute(action_type: str, instruction: str, workspace, conversation, orchestrator=None) -> Dict[str, Any]:
    try:
        # 执行Action
        output = call_llm(**llm_kwargs)
        notes_created = extract_and_create_notes(output, step_id, workspace)
        
        return {"success": True, "output": output, "notes_created": notes_created}
    except Exception as e:
        logger.error(f"执行步骤失败: {str(e)}")
        return {"success": False, "error": str(e), "notes_created": []}
```

#### 5.3.2 状态恢复
- 通过execution_log重建执行历史
- 支持从断点继续执行
- 保持数据一致性

### 5.4 优先级调度机制

#### 5.4.1 动态配额管理
```python
def _calculate_dynamic_command_quota(self):
    """计算动态命令配额：初始12条，每次反馈增加3条"""
    orchestrator_calls = self.conversation.get_recent_orchestrator_calls(limit=None)
    call_count = len(orchestrator_calls)
    
    base_commands = 12
    feedback_bonus = max(0, call_count - 1) * 3
    max_commands = base_commands + feedback_bonus
```

#### 5.4.2 资源优化
```python
if remaining_commands <= 3:
    parts.append("⚠️  剩余命令数较少，请优先执行最关键的Action")
```

### 5.5 异步协同机制

#### 5.5.1 异步执行
```python
def run_orchestrator_async(self, task_message: str):
    """异步运行Orchestrator"""
    def run():
        # 在后台线程中执行Orchestrator
        pass
    
    thread = threading.Thread(target=run)
    thread.start()
```

#### 5.5.2 状态同步
- 通过ConversationManager维护全局状态
- 支持实时状态查询和更新
- 确保多线程环境下的数据一致性

---

## 6️⃣ 性能与系统监控能力分析

### 6.1 响应延迟评估

#### 6.1.1 Token使用统计
```python
_session_stats = {
    "total_calls": 0,
    "total_input_tokens": 0,
    "total_output_tokens": 0,
    "total_thinking_tokens": 0,
    "total_cost": 0.0
}

def update_session_stats(input_tokens: int, output_tokens: int, thinking_tokens: int, cost: float):
    """更新会话统计"""
    global _session_stats
    _session_stats["total_calls"] += 1
    _session_stats["total_input_tokens"] += input_tokens
    _session_stats["total_output_tokens"] += output_tokens
    _session_stats["total_thinking_tokens"] += thinking_tokens
    _session_stats["total_cost"] += cost
```

#### 6.1.2 性能监控面板
```python
def create_llm_panel(prompt_type: str, user_prompt: str, response: str, 
                     input_tokens: int, output_tokens: int, thinking_tokens: int, 
                     call_cost: float, total_cost: float) -> None:
    """创建并显示LLM调用面板"""
    # 显示详细的性能统计信息
    token_stats = f"Tokens: {input_tokens:,} in + {output_tokens:,} out + {thinking_tokens:,} thinking = {total_tokens:,} total"
    cost_stats = f"Cost: ${call_cost:.4f} (${total_cost:.4f} session)"
```

### 6.2 Token成本管理

#### 6.2.1 成本计算
```python
def calculate_cost(self, input_tokens: int, output_tokens: int, thinking_tokens: int = 0, **kwargs) -> float:
    """计算调用成本"""
    # 不同模型提供商有不同的定价策略
    pass
```

#### 6.2.2 成本优化策略
- 根据Action类型调整参数
- 使用thinking_budget控制思考深度
- 动态调整max_output_tokens

### 6.3 Memory检索效率

#### 6.3.1 分层检索
```python
def get_recent_chat_history(self, limit: int = 10) -> List[Dict[str, Any]]:
    """获取最近的用户对话历史（仅user和concierge类型）"""
    chat_history = []
    for entry in reversed(self.history):
        if entry["type"] in ["user", "concierge"]:
            chat_history.append(entry)
            if len(chat_history) >= limit * 2:  # user和concierge成对
                break
    return list(reversed(chat_history))
```

#### 6.3.2 智能缓存
- 通过notes系统实现知识缓存
- 支持按类型和状态快速检索
- 避免重复计算和存储

### 6.4 优化工具使用

#### 6.4.1 日志系统
```python
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[logging.FileHandler('loomi.log')]
)
```

#### 6.4.2 性能分析
- 详细的执行记录和统计
- 支持性能瓶颈分析
- 提供优化建议

---

## 7️⃣ 安全性与合规意识分析

### 7.1 权限边界设计

#### 7.1.1 上下文隔离
```python
def build_action_context(step: Dict, workspace, conversation) -> str:
    """Action的上下文极度精简，只包含必要信息"""
    # 只包含：任务指令、引用的note全文
    # 不包含：聊天历史、其他notes、系统状态等敏感信息
```

#### 7.1.2 数据访问控制
- WorkSpace提供get_referenceable_notes()方法
- 支持notes的状态管理（star、archive、trash）
- 防止敏感信息泄露

### 7.2 Prompt注入防护

#### 7.2.1 输入验证
```python
def _clean_response(self, response: str) -> str:
    """清理响应文本，移除XML标签"""
    patterns = [
        r'<call_orchestrator>.*?</call_orchestrator>',
        r'<save_material>.*?</save_material>'
    ]
    
    clean = response
    for pattern in patterns:
        clean = re.sub(pattern, '', clean, flags=re.DOTALL)
```

#### 7.2.2 输出过滤
- 使用RobustNotesExtractor处理LLM输出
- 多级解析策略防止恶意标签
- 内容清理和验证

### 7.3 数据越权防护

#### 7.3.1 引用验证
```python
def build_action_context(step: Dict, workspace, conversation) -> str:
    # 验证@引用的有效性
    refs = re.findall(r'@([a-zA-Z_]+\d+)', step["instruction"])
    referenceable_notes = workspace.get_referenceable_notes()
    
    for ref in refs:
        note_data = referenceable_notes.get(ref)
        if note_data:
            # 只提供可引用的notes
            parts.append(f"@{ref}:")
            parts.append(note_data["content"])
```

#### 7.3.2 状态隔离
- 每个Agent只能访问其权限范围内的数据
- 通过Context Builders控制信息流动
- 防止跨Agent的数据泄露

### 7.4 敏感信息保护

#### 7.4.1 日志脱敏
```python
logger.info(f"添加用户消息: {content[:50]}...")  # 只记录前50字符
logger.info(f"添加Concierge响应: {content[:50]}...")  # 避免记录完整敏感信息
```

#### 7.4.2 数据加密
- 敏感信息不直接存储在日志中
- 使用摘要和预览替代完整内容
- 支持数据清理和归档

### 7.5 审计与合规回溯

#### 7.5.1 完整执行记录
```python
def add_execution_record(self, step: str, notes_created: Optional[List[str]] = None, 
                        observe: Optional[str] = None, think: Optional[str] = None, 
                        actions_detail: Optional[List[Dict]] = None):
    """记录完整的执行信息，支持审计"""
    entry = {
        "type": "execution",
        "step": step,
        "notes_created": notes_created or [],
        "observe": observe,
        "think": think,
        "actions_detail": actions_detail
    }
```

#### 7.5.2 合规性支持
- 完整的操作日志记录
- 支持操作回溯和审计
- 符合数据保护法规要求

---

## 8️⃣ 项目细节和创新点分析

### 8.1 技术创新点

#### 8.1.1 面向上下文的编程
**创新性**：将Agent视为上下文消费者，实现精确的信息流动控制
**实现**：
```python
# 为不同Agent定制精确的上下文
def build_concierge_context(user_msg: str, workspace, conversation) -> str:
    # Concierge：宽而浅，快速决策
    
def build_orchestrator_context(task_message: str, workspace, conversation) -> str:
    # Orchestrator：深而战略性，支持规划
    
def build_action_context(step: Dict, workspace, conversation) -> str:
    # Action：窄而深，高度专注
```

#### 8.1.2 鲁棒的Notes提取系统
**创新性**：多级解析策略处理LLM输出的各种边界情况
**实现**：
```python
class RobustNotesExtractor:
    def extract_notes(self, response: str) -> Dict[str, List[Tuple[str, str]]]:
        # 第一级：完美匹配
        # 第二级：修复匹配
        # 第三级：模糊匹配
```

#### 8.1.3 动态配额管理
**创新性**：根据用户反馈动态调整执行配额
**实现**：
```python
# 初始12条，每次反馈增加3条
base_commands = 12
feedback_bonus = max(0, call_count - 1) * 3
max_commands = base_commands + feedback_bonus
```

### 8.2 架构设计亮点

#### 8.2.1 分层数据管理
- **WorkSpace**：结构化知识库
- **ConversationManager**：线性对话历史
- **Context Builders**：按需上下文构建

#### 8.2.2 多模型支持
```python
class ModelManager:
    def register_provider(self, provider: ModelProvider):
        self._providers[provider.name] = provider
    
    def set_current_provider(self, name: str):
        if name in self._providers:
            self._current_provider = self._providers[name]
```

#### 8.2.3 智能任务分流
- Concierge根据任务复杂度智能分流
- 避免不必要的复杂处理
- 提高系统响应效率

### 8.3 用户体验创新

#### 8.3.1 动态加载效果
```python
def show_dynamic_loading():
    """显示动态加载效果，提升用户体验"""
    loading_steps = [
        {"text": "Access granted, pending...", "color": "white"},
        {"text": "[7%]Initializing MetaCognitive Core...", "color": "grey70"},
        # ... 更多步骤
    ]
```

#### 8.3.2 实时状态反馈
- 显示Orchestrator运行状态
- 提供执行进度和配额信息
- 支持实时补充需求

### 8.4 系统健壮性

#### 8.4.1 错误处理机制
- 多级解析策略处理LLM输出异常
- 完善的异常捕获和恢复机制
- 支持从断点继续执行

#### 8.4.2 性能优化
- 智能缓存和检索机制
- 动态参数调整
- 详细的性能监控

### 8.5 扩展性设计

#### 8.5.1 模块化架构
- 清晰的模块边界和接口
- 支持新Action类型的快速添加
- 支持新模型提供商的无缝集成

#### 8.5.2 配置化设计
- 通过配置文件管理系统参数
- 支持不同场景的定制化配置
- 便于部署和维护

### 8.6 总结

Loomi 3.0项目展现了以下核心创新：

1. **面向上下文的编程哲学**：将Agent视为上下文消费者，实现精确的信息流动控制
2. **鲁棒的多级解析系统**：能够处理LLM输出的各种边界情况
3. **智能的任务分流机制**：根据任务复杂度动态调整处理策略
4. **动态配额管理**：根据用户反馈优化执行效率
5. **完善的监控和审计**：支持性能分析和合规回溯
6. **模块化和可扩展设计**：便于功能扩展和维护

这些创新点使得Loomi 3.0成为一个高效、健壮、可扩展的多Agent内容创作系统，为AI驱动的创作工作流提供了新的范式。 
