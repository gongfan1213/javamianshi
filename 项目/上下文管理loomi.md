我来详细分析Loomi 3.0项目的上下文管理机制，结合源码进行深入解析。

Read file: core/context_builder.py
Read file: agents/concierge.py
Read file: agents/orchestrator.py
Read file: agents/orchestrator.py
Read file: core/conversation.py
现在我来详细分析Loomi 3.0项目的上下文管理机制，结合源码进行深入解析：

# Loomi 3.0 上下文管理机制深度分析

## 1️⃣ 上下文管理架构概览

### 1.1 核心设计哲学：面向上下文的编程

Loomi 3.0采用了**面向上下文的编程（Context-Oriented Programming）**设计哲学，其核心思想是：

- **精确性控制**：为每个Agent提供恰到好处的上下文
- **分层管理**：不同Agent使用不同粒度的上下文信息
- **动态构建**：上下文不是静态的，随ReAct循环动态更新

### 1.2 上下文管理架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Context Builders                        │
│  ┌─────────────────┐ ┌─────────────────┐ ┌──────────────┐ │
│  │ build_concierge │ │build_orchestrator│ │build_action  │ │
│  │   _context      │ │   _context      │ │  _context    │ │
│  └─────────────────┘ └─────────────────┘ └──────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Data Sources                            │
│  ┌─────────────────┐ ┌─────────────────┐ ┌──────────────┐ │
│  │  WorkSpace      │ │ConversationMgr  │ │  Notes       │ │
│  │  (Structured)   │ │  (Linear)       │ │  (KV Cache)  │ │
│  └─────────────────┘ └─────────────────┘ └──────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 2️⃣ 三层上下文构建器详解

### 2.1 Concierge上下文构建器

**源码分析**：
```python
def build_concierge_context(user_msg: str, workspace, conversation) -> str:
    """
    构建Concierge的上下文
    
    包含:
    - chat_history（聊天历史，最新消息会标记）
    - orchestrator_timeline（Orchestrator统一时间线）
    - created_notes（所有生成的笔记）
    """
```

**上下文特点**：**宽而浅，快速决策**

#### 2.1.1 聊天历史管理
```python
# 获取最近100条聊天历史
chat_history = conversation.get_recent_chat_history(limit=100)

# 智能标记新消息
is_new_message = True
if chat_history and chat_history[-1]["type"] == "user" and chat_history[-1]["content"] == user_msg:
    is_new_message = False

# 构建带编号的聊天历史
for entry in chat_history:
    message_count += 1
    if entry["type"] == "user":
        parts.append(f"#{message_count} **用户**: {entry['content']}")
    elif entry["type"] == "concierge":
        original_response = entry.get('original_response', entry['content'])
        parts.append(f"#{message_count} Loomi（你）: {original_response}")
```

#### 2.1.2 Orchestrator时间线
```python
# 获取所有Orchestrator调用历史
orchestrator_calls = conversation.get_recent_orchestrator_calls(limit=None)

# 按时间顺序显示执行记录
for msg_idx, msg in enumerate(orchestrator_calls):
    parts.append(f"# 用户消息{msg_idx + 1}: {msg['message']}")
    
    # 显示每个Round的执行详情（简化版）
    for record in msg['execution_records']:
        round_match = re.match(r'Round(\d+):', record['step'])
        if round_match:
            round_num = round_match.group(1)
            parts.append(f"## Round{round_num}:")
            
            # 显示observe和think（80字符预览）
            if 'observe' in record and record['observe']:
                observe_clean = record['observe'].replace('\n', ' ').replace('\r', ' ')
                observe_clean = ' '.join(observe_clean.split())
                observe_preview = observe_clean[:80] + "..." if len(observe_clean) > 80 else observe_clean
                parts.append(f"🔍 观察: {observe_preview}")
```

#### 2.1.3 Notes摘要
```python
# 显示所有notes的前50字符摘要
for note_id, note_data in workspace.notes.items():
    raw_content = note_data["content"]
    
    # 对特定类型进行换行清理
    note_type = note_id.split('_')[0] if '_' in note_id else note_id.rstrip('0123456789')
    if note_type in ['hitpoint', 'xhs', 'wechat', 'tiktok', 'websearch', 'knowledge', 'persona', 'resonant', 'brand', 'content']:
        clean_content = raw_content.replace('\n', ' ').replace('\r', ' ')
        clean_content = ' '.join(clean_content.split())
        content_preview = clean_content[:50] + "..." if len(clean_content) > 50 else clean_content
    else:
        content_preview = raw_content[:50] + "..." if len(raw_content) > 50 else raw_content
    
    parts.append(f"@{note_id}: {content_preview}")
```

### 2.2 Orchestrator上下文构建器

**源码分析**：
```python
def build_orchestrator_context(task_message: str, workspace, conversation, execution_log: Optional[List[Dict]] = None) -> str:
    """
    构建Orchestrator的上下文
    
    包含:
    - 工作状态总览（材料使用关系和当前状态）
    - 完整的用户消息队列和执行时间线
    - 每个Round的产出Notes（显示在对应Round执行记录后）
    """
```

**上下文特点**：**深而战略性，支持规划**

#### 2.2.1 工作状态总览
```python
# 动态计算命令配额
orchestrator_calls = conversation.get_recent_orchestrator_calls(limit=None)
call_count = len(orchestrator_calls)

# 动态计算命令配额：初始12条，每次反馈增加3条
base_commands = 12
feedback_bonus = max(0, call_count - 1) * 3  # 首次不算反馈，从第2次开始每次+3
max_commands = base_commands + feedback_bonus

executed_commands = len(execution_log) if execution_log else 0
remaining_commands = max_commands - executed_commands

# 显示命令执行状态
parts.append(f"🎯 命令执行状态: 已执行 {executed_commands}/{max_commands} 条命令，剩余 {remaining_commands} 条")
```

#### 2.2.2 材料状态分析
```python
# 按类型分类材料
materials = {}  # 原材料库
hitpoints = {}  # hitpoint库
writings = {}   # 写作内容库

for note_id, note_data in all_notes.items():
    note_type = note_id.split('_')[0] if '_' in note_id else note_id.rstrip('0123456789')
    
    if note_type in ['websearch', 'knowledge', 'persona', 'resonant', 'brand', 'content', 'material']:
        materials[note_id] = note_data
    elif note_type == 'hitpoint':
        hitpoints[note_id] = note_data
    elif note_type in ['xhs', 'wechat', 'tiktok']:
        writings[note_id] = note_data

# 分析材料使用状态
used_materials = set()  # 被Action引用的材料
used_hitpoints = set()  # 被写作类Action使用的hitpoint

# 从execution_log中获取材料使用信息
if execution_log:
    for log_entry in execution_log:
        action = log_entry.get('action', '')
        instruction = log_entry.get('instruction', '')
        
        # 从instruction中提取@引用
        if '@' in instruction:
            refs = re.findall(r'@([a-zA-Z_]+\d+)', instruction)
            
            # 根据action类型分类处理
            if action == 'hitpoint':
                # hitpoint Action使用的是原材料
                for ref in refs:
                    if ref in materials:
                        used_materials.add(ref)
            elif action in ['xhs_post', 'wechat_article', 'tiktok_script']:
                # 写作类Action使用的是hitpoint和材料
                for ref in refs:
                    if ref in hitpoints:
                        used_hitpoints.add(ref)
                    elif ref in materials:
                        used_materials.add(ref)
```

#### 2.2.3 完整执行时间线
```python
# 显示每个Round的详细执行记录
for record in msg['execution_records']:
    round_match = re.match(r'Round(\d+):', record['step'])
    if round_match:
        round_num = round_match.group(1)
        parts.append(f"## Round{round_num}:")
        
        # 显示observe（完整内容）
        if 'observe' in record and record['observe']:
            observe_clean = record['observe'].replace('\n', ' ').replace('\r', ' ')
            observe_clean = ' '.join(observe_clean.split())
            parts.append(f"�� 观察: {observe_clean}")
        
        # 显示think（完整内容）
        if 'think' in record and record['think']:
            think_clean = record['think'].replace('\n', ' ').replace('\r', ' ')
            think_clean = ' '.join(think_clean.split())
            parts.append(f"💭 思考: {think_clean}")
        
        # 显示actions详细信息
        if 'actions_detail' in record and record['actions_detail']:
            for action_detail in record['actions_detail']:
                action = action_detail['action']
                instruction = action_detail['instruction']
                action_notes = action_detail.get('notes_created', [])
                
                parts.append(f"⚡ <call agent={action} instruction={instruction}/>")
                
                # 显示该action的产出
                if action_notes:
                    for note_id in action_notes:
                        note = workspace.get_note(note_id)
                        if note:
                            raw_content = note['content']
                            clean_content = raw_content.replace('\n', ' ').replace('\r', ' ')
                            clean_content = ' '.join(clean_content.split())
                            content_preview = clean_content[:100] + "..." if len(clean_content) > 100 else clean_content
                            parts.append(f"   产出 @{note_id}: {content_preview}")
```

### 2.3 Action上下文构建器

**源码分析**：
```python
def build_action_context(step: Dict, workspace, conversation) -> str:
    """
    构建Action的上下文
    
    包含:
    - 写作任务背景（部分Action需要，如写作类）
    - Orchestrator提供的instruction
    - instruction中@引用的完整内容
    """
```

**上下文特点**：**窄而深，高度专注**

#### 2.3.1 智能背景添加
```python
# 定义写作类Action列表（用于自动material引用）
writing_actions = ["xhs_post", "wechat_article", "tiktok_script", "hitpoint"]

# 定义不需要写作任务背景的Action类型
no_user_need_actions = ["websearch", "knowledge", "resonant"]

# 根据Action类型决定是否添加写作任务背景
if step["action"] not in no_user_need_actions:
    parts.append("[写作任务背景]（注意这是背景，并非你的任务）")
    # 获取所有Concierge传递给Orchestrator的任务消息
    orchestrator_calls = conversation.get_recent_orchestrator_calls()
    if orchestrator_calls:
        # 将所有任务消息按时间顺序展示
        for i, call in enumerate(orchestrator_calls):
            if i == 0:
                # 第一条是主要需求
                parts.append(call['message'])
            else:
                # 后续的是补充和反馈
                parts.append(f"补充: {call['message']}")
```

#### 2.3.2 智能引用展开
```python
# 查找instruction中的@引用
if '@' in step["instruction"]:
    refs = re.findall(r'@([a-zA-Z_]+\d+)', step["instruction"])
    refs = list(set(refs))  # 去重

# 为hitpoint和创作类Action自动添加@material引用
if step["action"] in writing_actions:
    # 查找所有material类型的notes
    referenceable_notes = workspace.get_referenceable_notes()
    material_notes = [note_id for note_id in referenceable_notes.keys() if note_id.startswith('material')]
    # 添加到引用列表，但避免重复
    for material_id in material_notes:
        if material_id not in refs:
            refs.append(material_id)

# 展开所有@参考资料
if refs:
    parts.append("[参考资料]")
    referenceable_notes = workspace.get_referenceable_notes()
    for ref in refs:
        note_data = referenceable_notes.get(ref)
        if note_data:
            parts.append(f"@{ref}:")
            parts.append(note_data["content"])  # 完整内容，不是摘要
            parts.append("")
        else:
            parts.append(f"@{ref}: [未找到或不可引用]")
            parts.append("")
```

## 3️⃣ 上下文数据源管理

### 3.1 WorkSpace（结构化数据源）

**源码分析**：
```python
class WorkSpace:
    def __init__(self):
        # Notes存储 - 使用类型+编号作为key
        self.notes: Dict[str, Dict[str, str]] = {}
        
        # 编号计数器 - 用于自动编号
        self._note_counters: Dict[str, int] = {}
        
        # tactics - Orchestrator的工作备忘
        self.tactics: Optional[str] = None
```

**KV Cache设计**：
```python
# 键：note_id（如resonant1, persona2, hitpoint3）
# 值：结构化数据字典
self.notes[note_id] = {
    "type": note_type,        # 类型标识
    "content": content,       # 实际内容
    "source": source,         # 来源标识
    "review_status": status   # 状态管理（可选）
}
```

### 3.2 ConversationManager（线性数据源）

**源码分析**：
```python
class ConversationManager:
    def __init__(self):
        # 完整的对话历史
        self.history: List[Dict[str, Any]] = []
        
        # Orchestrator调用历史（用于快速访问）
        self._orchestrator_calls: List[Dict[str, Any]] = []
        
        # 当前活跃的orchestrator调用索引
        self._active_orchestrator_index: Optional[int] = None
        
        # Orchestrator运行状态
        self._orchestrator_running: bool = False
```

**分层访问机制**：
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

def get_recent_orchestrator_calls(self, limit: Optional[int] = 10) -> List[Dict[str, str]]:
    """获取最近的Orchestrator调用历史"""
    recent_calls = []
    
    # 如果limit为None，获取所有历史；否则获取最近的limit条
    calls_to_process = self._orchestrator_calls if limit is None else self._orchestrator_calls[-limit:]
    
    for call in reversed(calls_to_process):
        result = {
            "message": call["message"]
        }
        
        # 如果有响应，也包含进去
        if "response_index" in call:
            response = self.history[call["response_index"]]
            result["response"] = response["content"]
        
        # 包含执行记录（新增）
        if "execution_records" in call:
            result["execution_records"] = call["execution_records"]
        
        recent_calls.append(result)
    
    return list(reversed(recent_calls))
```

## 4️⃣ 上下文使用流程

### 4.1 Concierge上下文使用

**源码分析**：
```python
def process_user_input(self, user_input: str) -> str:
    # 记录用户消息
    self.conversation.add_user_message(user_input, notes_created)
    
    # 构建上下文并调用LLM
    context = build_concierge_context(user_input, self.workspace, self.conversation)
    
    # 使用用户选择的LLM模型
    response = call_llm(
        system_prompt=self.system_prompt,
        user_prompt=context,
        temperature=0.4,
        max_output_tokens=4000
    )
```

### 4.2 Orchestrator上下文使用

**源码分析**：
```python
def process_task(self, task_message: str, execution_log: Optional[List[Dict]] = None) -> Dict[str, Any]:
    # 构建上下文
    context = build_orchestrator_context(
        task_message,
        self.workspace,
        self.conversation,
        execution_log=execution_log or []
    )
    
    # 调用LLM
    response = call_llm(
        system_prompt=self.system_prompt,
        user_prompt=context,
        temperature=0.5,
        max_output_tokens=4000
    )
```

### 4.3 Action上下文使用

**源码分析**：
```python
def execute(action_type: str, instruction: str, workspace, conversation, orchestrator=None) -> Dict[str, Any]:
    # 构建简单的步骤对象（仅用于build_action_context）
    step = {
        "action": action_type,
        "instruction": instruction
    }
    
    # 构建上下文
    user_prompt = build_action_context(step, workspace, conversation)
    
    # 根据action类型设置不同的参数
    if action_type == "knowledge":
        temperature = 0.3
        max_tokens = 8000
        thinking_budget = None
    elif action_type in ["xhs_post", "wechat_article", "tiktok_script", "hitpoint"]:
        temperature = 0.6
        max_tokens = 8000
        thinking_budget = 1200
    else:
        temperature = 0.4
        max_tokens = 8000
        thinking_budget = 500
    
    # 调用LLM
    output = call_llm(**llm_kwargs)
```

## 5️⃣ 上下文管理创新点

### 5.1 精确性控制
- **Concierge**：宽而浅，快速决策（摘要级别）
- **Orchestrator**：深而战略性，支持规划（完整内容）
- **Action**：窄而深，高度专注（精确引用）

### 5.2 动态性控制
- 上下文不是静态的，每次ReAct循环都会更新
- 支持实时补充新需求
- 动态调整信息粒度

### 5.3 智能引用机制
- 自动解析@引用并展开完整内容
- 智能添加material引用（写作类Action）
- 支持引用验证和错误处理

### 5.4 分层数据管理
- **WorkSpace**：结构化知识库（KV Cache）
- **ConversationManager**：线性对话历史
- **Context Builders**：按需上下文构建

## 6️⃣ 总结

Loomi 3.0的上下文管理机制体现了以下核心特点：

1. **面向上下文的编程哲学**：将Agent视为上下文消费者，实现精确的信息流动控制
2. **三层上下文构建器**：为不同Agent量身定制上下文，确保信息干扰最小
3. **智能数据管理**：通过WorkSpace和ConversationManager实现高效的数据存储和检索
4. **动态上下文更新**：支持实时状态更新和ReAct循环
5. **鲁棒的引用机制**：智能解析和展开@引用，确保数据完整性

这种设计使得Loomi 3.0能够在复杂的多Agent协作环境中保持高效、精确的上下文管理，为AI驱动的创作工作流提供了强有力的技术支撑。
