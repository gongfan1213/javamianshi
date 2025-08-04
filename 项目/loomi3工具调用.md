我来详细分析Loomi 3.0项目的工具调用机制，结合源码进行深入解析。

Ran tool
Ran tool
Read file: prompts/action_prompts.py
Read file: agents/orchestrator.py
Ran tool
Read file: prompts/concierge_prompt.py
Read file: prompts/orchestrator_prompt.py
现在我来详细分析Loomi 3.0项目的工具调用机制，结合源码进行深入解析：

# Loomi 3.0 工具调用机制深度分析

## 1️⃣ 工具调用架构概览

### 1.1 工具调用类型分类

Loomi 3.0系统采用了**混合工具调用架构**，包含以下三种类型：

#### 1.1.1 静态工具调用（Static Tool Calls）
- **Action工具**：预定义的Action类型，存储在`ACTION_PROMPTS`字典中
- **Concierge工具**：`<save_material>`和`<call_orchestrator>`标签

#### 1.1.2 动态工具调用（Dynamic Tool Calls）
- **Orchestrator的execute命令**：通过XML标签动态调用Action
- **智能参数解析**：支持多种格式的execute标签解析

#### 1.1.3 特殊工具调用（Special Tool Calls）
- **websearch工具**：集成真实网络搜索功能
- **模型切换工具**：支持多种LLM模型提供商

---

## 2️⃣ 静态工具调用分析

### 2.1 Action工具注册机制

**源码分析**：
```python
# prompts/action_prompts.py
ACTION_PROMPTS = {
    "resonant": resonant_prompt,
    "persona": persona_prompt,
    "hitpoint": hitpoint_prompt,
    "xhs_post": xhs_post_prompt,
    "wechat_article": wechat_article_prompt,
    "tiktok_script": tiktok_script_prompt,
    "brand_analysis": brand_analysis_prompt,
    "content_analysis": content_analysis_prompt,
    "websearch": websearch_prompt,
    "knowledge": knowledge_prompt,
}
```

**特点**：
- **预定义注册**：所有Action类型在系统启动时预定义
- **Prompt驱动**：每个Action都有对应的系统提示词
- **类型安全**：通过字典键值对确保类型安全

### 2.2 Concierge工具调用

**源码分析**：
```python
# prompts/concierge_prompt.py
# 保存材料工具
<save_material>
<id>1</id>
<content>直接复制用户提供的内容</content>
</save_material>

# 调用Orchestrator工具
<call_orchestrator>
使用简洁流畅的几句话清楚地描述用户的需求
</call_orchestrator>
```

**工具解析机制**：
```python
def _extract_orchestrator_call(self, response: str) -> Optional[str]:
    """提取orchestrator调用 - 增强版，可处理不完整的XML标签"""
    # 首先尝试匹配完整的XML标签对
    complete_pattern = r'<call_orchestrator>(.*?)</call_orchestrator>'
    match = re.search(complete_pattern, response, re.DOTALL)
    if match:
        content = match.group(1).strip()
        return content if content else None
    
    # 如果没有找到完整的标签对，尝试匹配只有开始标签的情况
    incomplete_pattern = r'<call_orchestrator>(.*?)(?=<\w+|\n\n[^<]|$)'
    match = re.search(incomplete_pattern, response, re.DOTALL)
    if match:
        content = match.group(1).strip()
        if content and not content.startswith('<'):
            content = re.sub(r'\n\n.*?(?:我会|开始|执行).*$', '', content, flags=re.DOTALL)
            content = content.strip()
            if content:
                logger.warning("检测到不完整的call_orchestrator标签，已自动修复")
                return content
    
    return None

def _extract_save_material(self, response: str) -> List[Tuple[str, str]]:
    """提取save_material命令"""
    materials = []
    pattern = r'<save_material>\s*<id>(\d+)</id>\s*<content>(.*?)</content>\s*</save_material>'
    for match in re.finditer(pattern, response, re.DOTALL):
        user_id = match.group(1)
        content = match.group(2).strip()
        materials.append((user_id, content))
    return materials
```

---

## 3️⃣ 动态工具调用分析

### 3.1 Orchestrator的execute命令

**源码分析**：
```python
def _extract_execute_commands(self, response: str) -> List[Dict[str, str]]:
    """提取execute命令 - 支持多种格式"""
    commands = []
    
    # 尝试多种格式的正则表达式
    patterns = [
        # 标准格式：有双引号
        r'<execute\s+action="([^"]+)"\s+instruction="([^"]+)"\s*/?>',
        # 无双引号格式：修复版，使用正向先行断言正确匹配到结尾标签
        r'<execute\s+action=([^\s]+)\s+instruction=(.*?)(?=\s*/?>)',
        # 混合格式：一个有引号一个没有
        r'<execute\s+action="([^"]+)"\s+instruction=(.*?)(?=\s*/?>)',
        r'<execute\s+action=([^\s]+)\s+instruction="([^"]+)"\s*/?>',
    ]
    
    for i, pattern in enumerate(patterns):
        matches = list(re.finditer(pattern, response, re.DOTALL))
        if matches:
            logger.info(f"✅ 使用模式 {i+1} 成功提取到 {len(matches)} 个execute命令")
            for match in matches:
                action = match.group(1).strip().strip('"')  # 去掉可能的双引号
                instruction = match.group(2).strip().strip('"')  # 去掉可能的双引号
                logger.info(f"📋 提取到命令: action={action}, instruction={instruction[:50]}...")
                commands.append({
                    "action": action,
                    "instruction": instruction
                })
            # 找到匹配就停止尝试其他格式
            break
    
    return commands
```

**动态性特点**：
- **多格式支持**：支持带引号、不带引号、混合格式的XML标签
- **鲁棒性解析**：通过多种正则表达式模式确保解析成功
- **实时验证**：动态检查Action类型是否有效

### 3.2 Action执行机制

**源码分析**：
```python
def execute(action_type: str, instruction: str, workspace, conversation, orchestrator=None) -> Dict[str, Any]:
    """
    执行单个步骤的独立函数
    
    Args:
        action_type: Action类型
        instruction: 执行指令（可包含@引用）
        workspace: WorkSpace实例
        conversation: ConversationManager实例
        orchestrator: Orchestrator实例（可选，用于reflection）
        
    Returns:
        执行结果
    """
    from prompts.action_prompts import ACTION_PROMPTS
    from core.context_builder import build_action_context
    from core.notes_extractor import extract_and_create_notes
    from llm.model_base import call_llm
    
    logger.info(f"执行步骤: [{action_type}] {instruction[:50]}...")
    
    try:
        # 检查action类型是否有效
        if action_type not in ACTION_PROMPTS:
            raise ValueError(f"未知的Action类型: {action_type}")
        
        # 获取系统提示词
        system_prompt = ACTION_PROMPTS[action_type]
        
        # 构建简单的步骤对象（仅用于build_action_context）
        step = {
            "action": action_type,
            "instruction": instruction
        }
        
        # 构建上下文
        user_prompt = build_action_context(step, workspace, conversation)
        
        # 🔍 特殊处理：websearch action始终使用kimi的真实搜索
        if action_type == "websearch":
            from llm.kimi_provider import KimiProvider
            kimi_provider = KimiProvider()
            output = kimi_provider.call_llm(
                system_prompt=system_prompt,
                user_prompt=user_prompt,
                temperature=0.1,  # 降低温度提高准确性
                max_output_tokens=None,  # 不设限制
                enable_websearch=True  # 启用真实搜索
            )
        else:
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
            
            # 构建调用参数
            llm_kwargs = {
                "system_prompt": system_prompt,
                "user_prompt": user_prompt,
                "temperature": temperature,
                "max_output_tokens": max_tokens
            }
            
            # 只有需要时才添加thinking_budget参数
            if thinking_budget is not None:
                llm_kwargs["thinking_budget"] = thinking_budget
            
            # 其他action使用当前选择的基座模型
            output = call_llm(**llm_kwargs)
        
        # 提取notes
        step_id = f"orch_{action_type}_{len(workspace.notes) + 1}"
        notes_created = extract_and_create_notes(output, step_id, workspace, expected_types=[action_type])
        
        logger.info(f"步骤执行成功，创建了 {len(notes_created)} 个notes")
        return {
            "success": True,
            "output": output,
            "notes_created": notes_created
        }
        
    except Exception as e:
        logger.error(f"执行步骤失败: {str(e)}")
        return {
            "success": False,
            "error": str(e),
            "notes_created": []
        }
```

---

## 4️⃣ 工具调用类型详解

### 4.1 研究类Action工具

#### 4.1.1 brand_analysis（品牌分析）
```python
# 使用示例
<execute action="brand_analysis" instruction="从品牌产品形象角度分析，为什么理想L9在东北反而销量很高？极氪在东北市场怎么就卖得不好？"/>
```

#### 4.1.2 persona（人群画像）
```python
# 使用示例
<execute action="persona" instruction="北京鸡娃妈妈分几类？从妈妈的身心角度，辅导孩子的过程中，到底哪些点让她们烦躁崩溃？学而思学习机能怎样解决这些场景的根本问题"/>
```

#### 4.1.3 resonant（共鸣分析）
```python
# 使用示例
<execute action="resonant" instruction=""两只猫"、"拿铁"、"盐渍梅子""/>
```

#### 4.1.4 knowledge（知识提供）
```python
# 使用示例
<execute action="knowledge" instruction="体质偏瘦的孕妈有哪些常被忽略的孕期营养管理问题？增重慢的原因有哪些？什么情况下需要特别注意了？"/>
```

#### 4.1.5 content_analysis（内容分析）
```python
# 使用示例
<execute action="content_analysis" instruction="@material1：从用户注意力与接受屏障的角度看，这篇抖音口播稿为什么5秒完播率不高？是为什么断在哪里？"/>
```

#### 4.1.6 websearch（网络搜索）
```python
# 特殊处理：使用真实搜索功能
if action_type == "websearch":
    from llm.kimi_provider import KimiProvider
    kimi_provider = KimiProvider()
    output = kimi_provider.call_llm(
        system_prompt=system_prompt,
        user_prompt=user_prompt,
        temperature=0.1,
        max_output_tokens=None,
        enable_websearch=True  # 启用真实搜索
    )
```

### 4.2 创作类Action工具

#### 4.2.1 hitpoint（核心卖点）
```python
# 使用示例
<execute action="hitpoint" instruction="基于@persona2描述的受众和理想L9的舒适配置@knowledge1，给我三个超有说服力的打点。"/>
```

#### 4.2.2 xhs_post（小红书文案）
```python
# 使用示例
<execute action="xhs_post" instruction="基于@hitpoint1写一篇小红书内容，这篇内容是理想汽车销售会发的，不要有营销感，要有活人感，真实感。要能爆."/>
```

#### 4.2.3 wechat_article（公众号文章）
```python
# 使用示例
<execute action="wechat_article" instruction="写一篇咪蒙风的长文章，第一人称女儿视角，写爸爸偷偷用看病钱买了48罐羊奶粉，描述羊奶粉会销的四步骗局：（发鸡蛋→专家讲座→限时特惠价→跑路），并详细说明这四步套路为什么能稳稳吃住当代老年人。你还从营养学和生产工艺角度（基于@knowledge2）。"/>
```

#### 4.2.4 tiktok_script（抖音脚本）
```python
# 使用示例
<execute action="tiktok_script" instruction="基于@hitpoint2写一篇口播稿自述关于育儿：好相处的孩子一定是早早被当人看的孩子。因为现在孩子接触信息早，心智成熟很快，哪怕是很小的孩子，也要像跟成年朋友一样，正常地分享情绪和信息，早早建立正常的人际关系。你把小孩当小狗养，他就会像小狗一样给你捣乱惹麻烦。你把小孩当同龄人，他就能快速对你的烦恼、生活感同深受。"/>
```

---

## 5️⃣ 工具调用机制特点

### 5.1 混合架构设计

**静态工具**：
- **预定义Action**：系统启动时注册所有Action类型
- **Concierge工具**：固定的XML标签格式
- **类型安全**：通过字典键值对确保类型安全

**动态工具**：
- **Orchestrator调度**：动态决定调用哪些Action
- **参数动态化**：根据Action类型动态调整参数
- **实时验证**：运行时检查工具有效性

### 5.2 鲁棒性设计

**多格式支持**：
```python
patterns = [
    # 标准格式：有双引号
    r'<execute\s+action="([^"]+)"\s+instruction="([^"]+)"\s*/?>',
    # 无双引号格式
    r'<execute\s+action=([^\s]+)\s+instruction=(.*?)(?=\s*/?>)',
    # 混合格式
    r'<execute\s+action="([^"]+)"\s+instruction=(.*?)(?=\s*/?>)',
    r'<execute\s+action=([^\s]+)\s+instruction="([^"]+)"\s*/?>',
]
```

**错误处理机制**：
```python
try:
    # 检查action类型是否有效
    if action_type not in ACTION_PROMPTS:
        raise ValueError(f"未知的Action类型: {action_type}")
    
    # 执行Action
    output = call_llm(**llm_kwargs)
    
    return {"success": True, "output": output, "notes_created": notes_created}
except Exception as e:
    return {"success": False, "error": str(e), "notes_created": []}
```

### 5.3 智能参数调整

**根据Action类型动态调整**：
```python
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
```

---

## 6️⃣ 工具调用流程

### 6.1 Concierge工具调用流程

```
用户输入 → Concierge处理 → 提取工具标签 → 执行工具 → 更新状态
```

**具体流程**：
1. **用户输入**：Concierge接收用户输入
2. **构建上下文**：调用`build_concierge_context`
3. **LLM处理**：生成包含工具标签的响应
4. **工具提取**：`_extract_orchestrator_call`和`_extract_save_material`
5. **工具执行**：执行`save_material`或`call_orchestrator`
6. **状态更新**：更新ConversationManager和WorkSpace

### 6.2 Orchestrator工具调用流程

```
任务输入 → Orchestrator处理 → 生成execute命令 → 执行Action → 提取Notes → 循环
```

**具体流程**：
1. **任务输入**：Orchestrator接收任务消息
2. **构建上下文**：调用`build_orchestrator_context`
3. **LLM处理**：生成包含execute标签的响应
4. **命令提取**：`_extract_execute_commands`解析execute命令
5. **Action执行**：调用`execute`函数执行每个Action
6. **结果处理**：提取Notes并更新WorkSpace
7. **循环继续**：进入下一个ReAct循环

### 6.3 Action执行流程

```
Action调用 → 参数验证 → 上下文构建 → LLM执行 → 结果提取 → Notes创建
```

**具体流程**：
1. **Action调用**：从execute命令中获取action_type和instruction
2. **参数验证**：检查action_type是否在ACTION_PROMPTS中
3. **上下文构建**：调用`build_action_context`构建精简上下文
4. **LLM执行**：根据Action类型调整参数并调用LLM
5. **结果提取**：使用`extract_and_create_notes`提取结构化结果
6. **Notes创建**：将结果存储到WorkSpace中

---

## 7️⃣ 总结

Loomi 3.0的工具调用机制体现了以下核心特点：

### 7.1 架构特点
1. **混合架构**：静态预定义 + 动态调度
2. **分层设计**：Concierge、Orchestrator、Action三层工具调用
3. **类型安全**：通过字典注册确保工具类型安全

### 7.2 技术特点
1. **鲁棒性解析**：支持多种XML标签格式
2. **智能参数调整**：根据Action类型动态调整LLM参数
3. **错误处理机制**：完善的异常捕获和恢复机制

### 7.3 功能特点
1. **研究类工具**：brand_analysis、persona、resonant、knowledge、content_analysis、websearch
2. **创作类工具**：hitpoint、xhs_post、wechat_article、tiktok_script
3. **系统工具**：save_material、call_orchestrator

### 7.4 动态性特点
- **Orchestrator调度**：动态决定调用哪些Action
- **参数动态化**：根据Action类型和任务需求动态调整
- **实时验证**：运行时检查工具有效性和参数正确性

这种设计使得Loomi 3.0能够在保持系统稳定性的同时，实现灵活、智能的工具调用机制，为AI驱动的创作工作流提供了强有力的技术支撑。
