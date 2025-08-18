# Nova3 智能编排系统 API 文档

## 概述

Nova3是基于任务队列的智能编排系统，支持洞察分析、受众画像、内容策略、事实收集和内容创作等多种AI能力的智能协作。系统具备任务队列管理、用户选择交互、引用机制等高级功能。

## 基础信息

- **接口地址**: `POST /novachat`
- **协议**: HTTP/HTTPS
- **响应格式**: Server-Sent Events (SSE) 流式响应
- **Content-Type**: `application/json`
- **Accept**: `text/event-stream`

## 核心特性

1. **智能任务编排**: 根据需求复杂度自动生成执行计划
2. **队列管理**: 基于user_id+session_id的任务队列持久化，支持高并发和数据隔离
3. **用户选择交互**: 支持用户选择特定结果继续执行
4. **引用机制**: 支持@语法引用历史结果
5. **流式响应**: 实时返回执行进度和结果
6. **灵活输入模式**: 支持纯文本查询、用户选择、引用等多种输入方式
7. **持久化存储**: 队列数据支持文件持久化，服务重启后自动恢复

## 请求参数结构

### ChatRequest 主体结构

```json
{
  "user_id": "string",        // 必填：用户唯一标识
  "session_id": "string",     // 必填：会话唯一标识
  "request_data": {           // 必填：请求数据
    // 详见下方RequestData结构
  }
}
```

### RequestData 详细结构

```json
{
  "query": "string",                          // 可选：用户查询内容(1-2000字符)，如果有用户选择或引用，可以为空
  "interaction_type": "string",               // 可选：交互类型
  "background": {                             // 可选：背景信息
    "account_name": "string",
    "platform": "string", 
    "target_audience": "string",
    "previous_content": ["string"],
    "industry": "string",
    "brand_style": "string"
  },
  "interaction_context": {},                  // 可选：交互上下文
  "nova3_selections": {                       // 可选：Nova3用户选择数据
    // 详见下方Nova3UserSelections结构
  },
  "references": ["@insight1", "@profile2"]    // 可选：引用列表
}
```

### Nova3UserSelections 用户选择结构

```json
{
  "nova3_insight": [               // 洞察选择数组
    {
      "id": "insight1",            // 必填：洞察ID（会话内唯一且连续）
      "content": "string",         // 必填：洞察内容
      "selected": true,            // 必填：是否选中
      "user_comment": "string"     // 可选：用户评论
    }
  ],
  "nova3_profile": [               // 受众画像选择数组
    {
      "id": "profile1",            // 必填：画像ID（会话内唯一且连续）
      "content": "string", 
      "selected": true,
      "user_comment": "string"
    }
  ],
  "nova3_hitpoint": [              // 内容打点选择数组
    {
      "id": "hitpoint1",           // 必填：打点ID（会话内唯一且连续）
      "content": "string",
      "selected": true, 
      "user_comment": "string"
    }
  ],
  "nova3_facts": [                 // 事实收集选择数组
    {
      "id": "fact1",               // 必填：事实ID（会话内唯一且连续）
      "content": "string",
      "selected": true,
      "user_comment": "string"
    }
  ],
  "nova3_xhs_post": [              // 小红书内容选择数组
    {
      "id": "xhs_post1",           // 必填：内容ID（会话内唯一且连续）
      "title": "string",           // 必填：内容标题
      "content": "string",         // 必填：内容正文
      "selected": true,
      "user_comment": "string"
    }
  ],
  "task_context": "string"         // 可选：任务上下文说明
}

**🔥 ID管理机制说明**：
- **会话级别唯一性**: 每个session_id内，同类型内容的ID保持唯一且连续递增
- **格式规范**: `{content_type}{number}`，如`xhs_post1`, `xhs_post2`, `insight1`, `profile1`等
- **持续递增**: 同一会话中多次生成内容时，ID会持续递增（如第一次生成xhs_post1、xhs_post2，第二次生成xhs_post3、xhs_post4）
- **跨类型独立**: 不同内容类型的ID计数器独立管理
- **会话隔离**: 不同session_id的ID计数器完全独立
```

### InteractionType 枚举值

- `full_analysis`: 完整分析（默认）
- `quick_suggestion`: 快速建议
- `nova3_continue`: Nova3队列继续执行
- `nova3_select`: Nova3用户选择模式

## 引用机制详解

### @引用语法

Nova3支持两种引用方式：

#### 1. 在query中直接引用
```json
{
  "query": "基于@insight1和@profile2的分析，创作针对性内容"
}
```

#### 2. 在references数组中明确指定
```json
{
  "query": "创作内容策略",
  "references": ["@insight1", "@profile2", "@hitpoint3"]
}
```

### 引用ID格式

- **洞察**: `@insight1`, `@insight2`, ...
- **画像**: `@profile1`, `@profile2`, ...  
- **打点**: `@hitpoint1`, `@hitpoint2`, ...
- **事实**: `@fact1`, `@fact2`, ...
- **内容**: `@xhs_post1`, `@xhs_post2`, ...

### ID管理示例

```bash
# 第一次请求 - 生成洞察和小红书内容
Session: beauty_session_001
Response: insight1, insight2, xhs_post1, xhs_post2

# 第二次请求 - 继续生成更多内容  
Session: beauty_session_001
Response: insight3, xhs_post3, xhs_post4

# 第三次请求 - 生成画像
Session: beauty_session_001  
Response: profile1, profile2

# 新会话 - ID重新开始
Session: skincare_session_002
Response: insight1, profile1, xhs_post1
```

### 引用解析机制

系统会自动：
1. 解析query和references中的@引用
2. 从当前session的历史任务结果中查找对应ID
3. 将引用内容添加到任务上下文中
4. 传递给子Agent执行

## 输入模式详解

Nova3支持多种灵活的输入模式，适应不同的用户交互场景：

### 1. 纯文本查询模式
```json
{
  "query": "帮我分析美妆行业的用户洞察"
}
```
**适用场景**: 首次请求或需要全新分析时

### 2. 用户选择模式
```json
{
  "nova3_selections": {
    "nova3_insight": [
      {"id": "insight1", "content": "25-35岁女性更关注天然成分", "selected": true},
      {"id": "insight3", "content": "价格敏感度高于品牌忠诚度", "selected": true}
    ]
  }
}
```
**适用场景**: 用户从前一步结果中选择特定内容继续处理
**特点**: query可以为空，系统会基于用户选择的内容继续执行

### 3. 引用模式
```json
{
  "references": ["@insight1", "@profile2"]
}
```
**适用场景**: 需要基于特定历史结果进行新的分析
**特点**: query可以为空，系统会自动获取引用内容作为输入

### 4. 混合模式
```json
{
  "query": "基于选择的洞察，生成针对性的内容策略",
  "nova3_selections": {
    "nova3_insight": [
      {"id": "insight1", "content": "...", "selected": true}
    ]
  },
  "references": ["@profile1"]
}
```
**适用场景**: 复杂的分析需求，结合多种输入源

### 输入验证规则

系统会验证请求数据的有效性：

✅ **有效请求**:
- 有query（不管是否有选择/引用）
- 无query但有用户选择（nova3_selections中至少有一个非空数组）
- 无query但有引用（references数组非空）

❌ **无效请求**:
- 既无query，也无用户选择，也无引用
- query超过5000字符

## 响应格式

### SSE事件流格式

每个事件都是JSON格式的StreamEvent：

```json
{
  "event_type": "llm_chunk",           // 事件类型
  "agent_source": "Nova3",             // 事件来源
  "timestamp": "2024-01-01T12:00:00Z", // 时间戳
  "payload": {
    "id": 1750905399940,               // 唯一标识符，用于区分每次方法调用
    "content_type": "nova3_insight",   // 内容类型
    "data": [{"id":"insight1","content":"...","type":"insight"}],  // JSON对象（不再是字符串）
    "metadata": {                      // 元数据
      "insight_count": 3,
      "total_tasks": 4
    }
  }
}
```

### 主要事件类型

- `system`: 系统消息
- `llm_chunk`: LLM输出内容
- `error`: 错误信息

### 主要内容类型

- `nova3_insight`: 洞察分析结果
- `nova3_profile`: 受众画像结果  
- `nova3_hitpoint`: 内容打点结果
- `nova3_facts`: 事实收集结果
- `nova3_xhs_post`: 小红书内容结果
- `plan_step`: 执行计划
- `final_answer`: 最终回答

### 重要更新说明

**🎉 数据格式优化 (v2.0)**：
- ✅ **payload.id**: 新增唯一标识符字段，用于区分每次方法调用
- ✅ **payload.data**: Nova3子agent现在返回JSON对象而不是JSON字符串
- ✅ **前端处理**: 无需再调用`JSON.parse()`，直接使用`event.payload.data`

**迁移指南**：
```javascript
// 旧版本（需要解析JSON字符串）
const data = JSON.parse(event.payload.data);

// 新版本（直接使用JSON对象）
const data = event.payload.data;
const uniqueId = event.payload.id;  // 新增：唯一标识符
```

## 完整使用示例

### 示例1：复杂任务 - 首次请求

```bash
curl -X POST "http://127.0.0.1:8001/novachat" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{
    "user_id": "demo_user",
    "session_id": "beauty_strategy_001", 
    "request_data": {
      "query": "深入研究小红书美妆内容策略，分析用户群体，制定爆款方向并创作示例",
      "interaction_type": "full_analysis",
      "background": {
        "account_name": "美妆研究室",
        "platform": "小红书",
        "target_audience": "20-35岁都市女性",
        "industry": "美妆护肤"
      }
    }
  }'
```

**预期响应**：
```json
// 系统消息
{"event_type":"system","agent_source":"Nova3","payload":{"data":"Nova3 Supervisor开始处理请求..."}}

// 执行计划
{"event_type":"llm_chunk","agent_source":"nova3_supervisor","payload":{"id":1750905399938,"content_type":"plan_step","data":[{"id":"task001","step":1,"action":"insight","description":"分析美妆行业发展趋势和用户行为变化","full_input":"基于用户需求分析美妆行业在小红书上的发展趋势、用户行为变化和内容消费偏好的演变","status":"pending","type":"plan_step"},{"id":"task002","step":2,"action":"profile","description":"刻画小红书美妆内容的核心受众画像","full_input":"刻画小红书美妆内容的核心受众画像，包括不同细分群体的特征、需求和行为模式","status":"pending","type":"plan_step"},{"id":"task003","step":3,"action":"hitpoint","description":"生成针对性的内容打点策略","full_input":"基于洞察和画像分析，生成针对性的内容打点策略和关键信息点","status":"pending","type":"plan_step"},{"id":"task004","step":4,"action":"xhs_writing","description":"创作小红书内容","full_input":"基于前面的分析结果，创作针对性的小红书内容，包括标题、正文和话题标签","status":"pending","type":"plan_step"}],"metadata":{"total_tasks":4}}}

// 洞察结果
{"event_type":"llm_chunk","agent_source":"insight_agent","payload":{"id":1750905399940,"content_type":"nova3_insight","data":[{"id":"insight1","content":"95后女性追求个性化妆容表达...","type":"insight"},{"id":"insight2","content":"职场女性注重效率与效果平衡...","type":"insight"}],"metadata":{"insight_count":2}}}

// 等待用户选择提示
{"event_type":"system","agent_source":"Nova3","payload":{"data":"下一个任务: profile（输入内容继续执行）"}}
```

### 示例2：用户选择继续执行

```bash
curl -X POST "http://127.0.0.1:8001/novachat" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{
    "user_id": "demo_user",
    "session_id": "beauty_strategy_001",
    "request_data": {
      "query": "基于我选择的洞察继续分析受众画像",
      "interaction_type": "nova3_continue",
      "nova3_selections": {
        "nova3_insight": [
          {
            "id": "insight1",
            "content": "95后女性追求个性化妆容表达，拒绝千篇一律",
            "selected": true,
            "user_comment": "重点关注个性化需求"
          },
          {
            "id": "insight2", 
            "content": "职场女性注重效率与效果平衡",
            "selected": true
          }
        ],
        "task_context": "重点分析个性化和效率需求的用户群体"
      },
      "references": ["@insight1", "@insight2"]
    }
  }'
```

**预期响应**：
```json
// 继续执行profile任务
{"event_type":"system","agent_source":"Nova3","payload":{"data":"检测到队列中有待执行任务，继续执行队列中的下一个任务..."}}

// 受众画像结果
{"event_type":"llm_chunk","agent_source":"profile_agent","payload":{"id":1750905401234,"content_type":"nova3_profile","data":[{"id":"profile1","content":"22-28岁一线城市白领女性...","type":"profile"}]}}
```

### 示例3：纯用户选择模式（无query）

```bash
curl -X POST "http://127.0.0.1:8001/novachat" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{
    "user_id": "demo_user",
    "session_id": "beauty_strategy_001",
    "request_data": {
      "interaction_type": "nova3_continue",
      "nova3_selections": {
        "nova3_insight": [
          {
            "id": "insight1",
            "content": "95后女性追求个性化妆容表达",
            "selected": true,
            "user_comment": "这个洞察很有价值"
          },
          {
            "id": "insight3",
            "content": "社交媒体影响购买决策",
            "selected": true
          }
        ]
      }
    }
  }'
```

**适用场景**: 用户从前一步的洞察结果中选择了特定内容，希望基于这些选择继续执行下一个任务（如生成受众画像）。

**预期响应**：
```json
// 系统检测到用户选择，继续执行队列
{"event_type":"system","agent_source":"Nova3","payload":{"data":"基于用户选择的内容继续执行..."}}

// 基于选择的洞察生成受众画像
{"event_type":"llm_chunk","agent_source":"profile_agent","payload":{"id":1750905402345,"content_type":"nova3_profile","data":[{"id":"profile1","content":"追求个性化的95后女性群体画像...","type":"profile"}]}}
```

### 示例4：引用历史结果

```bash
curl -X POST "http://127.0.0.1:8001/novachat" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{
    "user_id": "demo_user",
    "session_id": "beauty_strategy_001",
    "request_data": {
      "query": "结合@insight1和@profile1，创作一篇针对个性化需求的小红书内容",
      "interaction_type": "nova3_continue"
    }
  }'
```

### 示例5：简单请求

```bash
curl -X POST "http://127.0.0.1:8001/novachat" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{
    "user_id": "demo_user",
    "session_id": "simple_query_001",
    "request_data": {
      "query": "Nova3系统有哪些功能？",
      "interaction_type": "quick_suggestion"
    }
  }'
```

## 队列管理接口

### 查看队列状态

```bash
GET /novachat/queue/{user_id}/{session_id}
```

**说明**: 需要同时提供user_id和session_id，确保队列唯一性和安全性

**响应示例**：
```json
{
  "status": "active",
  "total_tasks": 4,
  "current_index": 2,
  "tasks": [
    {
      "id": "task001",
      "action": "insight",
      "status": "completed",
      "result": "[{\"id\":\"insight1\",\"content\":\"...\"}]"
    },
    {
      "id": "task002", 
      "action": "profile",
      "status": "running"
    }
  ]
}
```

### 清空队列

```bash
DELETE /novachat/queue/{user_id}/{session_id}
```

**说明**: 清空指定用户的指定会话队列，防止误删其他用户的队列

### 系统状态

```bash
GET /novachat/status
```

## 前端集成指南

### 1. 流式数据处理

```javascript
const eventSource = new EventSource('/novachat', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(requestData)
});

eventSource.onmessage = function(event) {
  if (event.data === '[DONE]') {
    eventSource.close();
    return;
  }
  
  const eventData = JSON.parse(event.data);
  
  // 处理不同类型的事件
  switch(eventData.payload.content_type) {
    case 'plan_step':
      displayPlan(eventData.payload.data); // data已经是对象数组
      break;
    case 'nova3_insight':
      displayInsights(eventData.payload.data); // data已经是对象数组
      break;
    case 'nova3_profile':
      displayProfiles(eventData.payload.data); // data已经是对象数组
      break;
    // ... 其他类型
  }
};
```

### 2. 用户选择界面

```javascript
// 显示执行计划
function displayPlan(planSteps) {
  const planContainer = document.getElementById('plan-container');
  planContainer.innerHTML = `<h3>执行计划（${planSteps.length}个步骤）</h3>`;
  
  planSteps.forEach(step => {
    const stepElement = document.createElement('div');
    stepElement.className = 'plan-step';
    stepElement.innerHTML = `
      <div class="step-header">
        <span class="step-number">步骤${step.step}</span>
        <span class="step-action">${step.action}</span>
        <span class="step-status ${step.status}">${step.status}</span>
      </div>
      <div class="step-description">${step.description}</div>
    `;
    planContainer.appendChild(stepElement);
  });
}

// 显示洞察选择界面
function displayInsights(insights) {
  insights.forEach(insight => {
    const checkbox = createCheckbox(insight.id, insight.content);
    const commentBox = createCommentBox(insight.id);
    // 添加到界面...
  });
}

// 收集用户选择
function collectUserSelections() {
  return {
    nova3_insight: getSelectedInsights(),
    nova3_profile: getSelectedProfiles(),
    nova3_hitpoint: getSelectedHitpoints(),
    nova3_facts: getSelectedFacts(),
    nova3_xhs_post: getSelectedXHSPosts()
  };
}
```

### 3. 构造后续请求

```javascript
function continueTask(userSelections) {
  const request = {
    user_id: currentUserId,
    session_id: currentSessionId,
    request_data: {
      query: "继续执行下一个任务",
      interaction_type: "nova3_continue",
      nova3_selections: userSelections
    }
  };
  
  sendNovaRequest(request);
}
```

## 最佳实践

### 1. Session管理
- 每个独立任务使用不同的session_id
- 长对话保持session_id一致
- 定期清理无用的队列

### 2. 用户体验
- 显示任务进度和队列状态
- 提供选择界面让用户参与决策
- 支持任务中断和重新开始

### 3. 错误处理
- 监听error事件类型
- 提供队列重置功能
- 处理网络中断和重连

### 4. 性能优化
- 对长时间运行的任务显示进度
- 分批处理大量选择数据
- 缓存用户的选择偏好

## 常见问题

### Q: 如何判断是否需要用户选择？
A: 观察返回的结构化数据（如insights数组），如果有多个项目就需要用户选择。

### Q: @引用找不到对应内容怎么办？
A: 系统会忽略无效引用，建议先调用队列状态接口确认可用的ID。

### Q: 队列执行过程中可以修改任务吗？
A: 当前不支持动态修改，但可以清空队列重新开始。

### Q: 如何处理任务执行失败？
A: 系统会返回error事件，可以选择跳过失败任务继续执行或重置队列。

---

## 配置说明

### 环境变量
- `OPENAI_API_KEY`: OpenAI API密钥
- `REDIS_HOST`: Redis服务器地址
- `REDIS_PASSWORD`: Redis密码
- `LANGCHAIN_API_KEY`: LangSmith API密钥（可选）

### Nova3配置
Nova3系统支持可配置的返回条数限制，可以在`config/app_config.yaml`中调整：

```yaml
nova3:
  # Nova3子agent返回条数限制配置
  max_results:
    insight: 5      # 洞察分析最大返回条数
    profile: 5      # 受众画像最大返回条数
    hitpoint: 5     # 内容打点最大返回条数
    facts: 5        # 事实收集最大返回条数
    xhs_post: 10    # 小红书内容最大返回条数
  
  # 是否启用条数限制
  enable_limits: true
  
  # 默认返回条数（当具体类型未配置时使用）
  default_max_results: 5
```

**配置说明：**
- `max_results`: 各类型内容的最大返回条数
- `enable_limits`: 是否启用条数限制，设为false时返回所有结果
- `default_max_results`: 默认最大返回条数

**环境变量覆盖：**
- `NOVA3_ENABLE_LIMITS`: 覆盖enable_limits设置
- `NOVA3_DEFAULT_MAX_RESULTS`: 覆盖default_max_results设置

**使用示例：**
```bash
# 临时禁用条数限制
export NOVA3_ENABLE_LIMITS=false

# 调整默认返回条数
export NOVA3_DEFAULT_MAX_RESULTS=8
```

---

## 更新日志

### v1.1 (2024-06-26)
- 新增Nova3子agent返回条数限制配置
- 支持可配置的最大结果数量
- 默认限制洞察、画像、打点、事实为5条
- 小红书内容默认限制10条
- FactsAgent搜索工具升级：使用新的社交媒体搜索API
- 优化小红书平台搜索，提供更准确的事实信息

### v1.0 (2024-01-01)
- 新增nova3_selections用户选择字段
- 支持@引用机制
- 优化任务队列管理
- 完善流式响应格式 
