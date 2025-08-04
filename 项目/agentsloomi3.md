基于对Loomi 3.0系统的深入分析，我来详细说明这个系统中的智能体架构和分工：

## Loomi 3.0 智能体架构详解

### �� 核心智能体数量：3个主要智能体

Loomi 3.0系统采用了**三层智能体架构**，包含3个核心智能体，每个都有明确的职责分工：

---

## 1️⃣ Concierge（前台接待智能体）

### 主要职责
- **用户交互入口**：接收所有用户输入，作为系统的第一道防线
- **意图理解**：快速理解用户的初步需求
- **任务分流**：判断是简单问答还是复杂任务
- **材料收集**：帮助用户保存和管理材料

### 工作特点
```python
def build_concierge_context(user_msg: str, workspace, conversation) -> str:
    # 上下文特点：宽而浅，快速决策
    # 包含：聊天历史、Orchestrator时间线、notes摘要
```

### 决策逻辑
- **简单任务**：直接回复，不调用Orchestrator
- **复杂任务**：通过`<call_orchestrator>`标签传递给Orchestrator
- **材料保存**：通过`<save_material>`标签保存用户提供的材料

---

## 2️⃣ Orchestrator（编排员智能体）

### 主要职责
- **任务规划**：系统的核心大脑，采用ReAct模式
- **策略制定**：制定和调整高层策略（tactics）
- **Action调度**：调用具体的Action智能体执行任务
- **流程控制**：管理整个创作流水线

### 工作流程
```
Observe → Think → Act → Observe → ...
```

### 核心能力
```python
def process_task(self, task_message: str, execution_log: Optional[List[Dict]] = None) -> Dict[str, Any]:
    # 1. 构建上下文观察全局状态
    context = build_orchestrator_context(...)
    
    # 2. LLM进行思考决策
    response = call_llm(...)
    
    # 3. 执行具体Action
    execute_commands = self._extract_execute_commands(response)
```

### 决策机制
- **动态配额管理**：初始12条命令，每次反馈增加3条
- **智能调度**：根据任务需求选择合适的Action组合
- **状态监控**：实时跟踪执行进度和结果

---

## 3️⃣ Action（执行者智能体）

### 主要职责
- **原子化执行**：执行具体的、原子化的指令
- **专业领域处理**：每个Action专注于特定领域
- **结果输出**：生成结构化的notes输出

### Action类型详解

#### 3.1 研究类Action
- **`brand_analysis`**：品牌分析，分析品牌形象和竞品
- **`persona`**：人群画像，描绘目标受众特征
- **`resonant`**：共鸣分析，寻找词与词之间的诗意联系
- **`knowledge`**：知识提供，提供干货信息
- **`content_analysis`**：内容分析，拆解内容质量和技巧
- **`websearch`**：网络搜索，获取实时信息

#### 3.2 创作类Action
- **`hitpoint`**：核心卖点，寻找最有说服力的叙事方向
- **`xhs_post`**：小红书文案创作
- **`wechat_article`**：微信公众号文章创作
- **`tiktok_script`**：抖音脚本创作

### 执行特点
```python
def build_action_context(step: Dict, workspace, conversation) -> str:
    # 上下文特点：窄而深，高度专注
    # 只包含：任务指令、引用的note全文
```

---

## �� 智能体协作机制

### 协作流程
```
用户输入 → Concierge → Orchestrator → Action → Notes → 循环
```

### 状态隔离
- **Concierge**：只能访问聊天历史和notes摘要
- **Orchestrator**：可以访问完整的任务状态和策略
- **Action**：只能访问指定的指令和引用的notes

### 信息传递
- **Notes系统**：通过结构化数据实现智能体间的异步通信
- **执行记录**：详细记录每个Action的执行结果和产出
- **上下文构建**：为每个智能体量身定制上下文

---

## 📊 智能体分工总结

| 智能体 | 角色 | 主要职责 | 上下文特点 | 决策频率 |
|--------|------|----------|------------|----------|
| **Concierge** | 前台接待 | 用户交互、任务分流 | 宽而浅，快速决策 | 每次用户输入 |
| **Orchestrator** | 编排员 | 任务规划、Action调度 | 深而战略性，支持规划 | ReAct循环 |
| **Action** | 执行者 | 原子化任务执行 | 窄而深，高度专注 | 按需调用 |

---

## �� 智能体设计哲学

### 1. 面向上下文的编程
- 每个智能体都是上下文消费者和信息生产者
- 通过精确的上下文控制实现高效协作

### 2. 职责分离
- **Concierge**：专注于用户交互和任务分流
- **Orchestrator**：专注于任务规划和流程控制
- **Action**：专注于具体任务的执行

### 3. 状态隔离
- 每个智能体只能访问其权限范围内的数据
- 通过Context Builders控制信息流动

### 4. 动态协作
- 支持实时补充新需求
- 动态调整执行策略和配额

---

## �� 系统优势

1. **高效协作**：三个智能体各司其职，形成完整的创作流水线
2. **智能分流**：Concierge根据任务复杂度智能分流，避免不必要的复杂处理
3. **灵活扩展**：Action类型可以灵活添加，支持新的创作需求
4. **状态管理**：完善的状态隔离和结果合并机制
5. **性能优化**：通过精确的上下文控制提高系统响应效率

这种三层智能体架构使得Loomi 3.0能够在保持系统简洁性的同时，实现复杂的多Agent协作，为AI驱动的创作工作流提供了高效、灵活的解决方案。
