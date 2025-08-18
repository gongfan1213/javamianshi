# Loomichat API 文档

## 概述

Loomichat是基于Loomi系统的智能内容创作API，采用Concierge接待员模式和ReAct编排架构，支持Notes系统管理上下文信息。

## 核心特性

- **Concierge接待员**: 理解用户需求并创建Notes
- **ReAct模式编排**: 通过Orchestrator进行智能任务编排
- **Notes系统**: 支持@引用的上下文管理
- **多Agent协作**: 洞察、画像、打点、写作等专业能力
- **流式响应**: 实时返回执行结果

## 主要接口

### 1. 核心聊天接口

```http
POST /loomichat
```

**请求格式**:
```json
{
  "user_id": "用户ID",
  "session_id": "会话ID", 
  "request_data": {
    "query": "用户查询内容",
    "background": {
      "brand": "品牌信息",
      "industry": "行业信息",
      "target_audience": "目标受众"
    },
    "interaction_context": {},
    "references": ["@note1", "@note2"],
    "auto": false
  }
}
```

**响应格式**: Server-Sent Events (SSE)
```json
{
  "event_type": "llm_chunk",
  "agent_source": "Loomi",
  "timestamp": "2024-01-01T00:00:00Z",
  "payload": {
    "content_type": "message",
    "data": "响应内容",
    "metadata": {}
  }
}
```

### 2. Notes管理接口

#### 获取Notes
```http
GET /loomichat/notes/{user_id}/{session_id}
```

#### 清空Notes
```http
DELETE /loomichat/notes/{user_id}/{session_id}
```

#### 搜索Notes
```http
GET /loomichat/notes/{user_id}/{session_id}/search?query=关键词
```

### 3. 系统管理接口

#### 获取可用Agent列表
```http
GET /loomichat/agents
```

#### 获取系统状态
```http
GET /loomichat/status
```

#### 访问统计
```http
GET /loomichat/access/stats
GET /loomichat/access/total
GET /loomichat/access/today
```

## 使用示例

### 基本对话
```bash
curl -X POST "http://localhost:8000/loomichat" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user123",
    "session_id": "session456",
    "request_data": {
      "query": "我想创作一篇关于健康生活的小红书内容"
    }
  }'
```

### 带背景信息的请求
```bash
curl -X POST "http://localhost:8000/loomichat" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user123",
    "session_id": "session456", 
    "request_data": {
      "query": "帮我写一篇小红书内容",
      "background": {
        "brand": "健康生活品牌",
        "industry": "健康养生",
        "target_audience": "25-35岁都市白领"
      }
    }
  }'
```

### 使用@引用
```bash
curl -X POST "http://localhost:8000/loomichat" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user123",
    "session_id": "session456",
    "request_data": {
      "query": "基于@洞察1和@策略1，帮我写一篇小红书内容",
      "references": ["@洞察1", "@策略1"]
    }
  }'
```

## 工作流程

1. **接待阶段**: Concierge理解用户需求，创建相关Notes
2. **编排阶段**: Orchestrator采用ReAct模式，决定执行步骤
3. **执行阶段**: 调用相应的子Agent（insight、profile、hitpoint等）
4. **创作阶段**: 根据前面的分析结果，生成具体内容

## Notes系统

### Notes类型
- **需求**: 用户的核心需求
- **洞察**: 分析得出的洞察
- **画像**: 受众画像信息
- **策略**: 内容策略
- **内容**: 生成的具体内容

### @引用语法
- `@note_id`: 引用特定的Note
- 系统会自动解析@引用并替换为实际内容
- 支持跨对话轮次的引用

## 错误处理

所有接口都返回标准的错误格式：
```json
{
  "status": "error",
  "message": "错误描述",
  "detail": "详细错误信息"
}
```

## 开发测试

可以使用提供的测试脚本：
```bash
python test_loomichat.py
```

## 与其他系统的对比

| 特性 | Loomichat | Novachat | Bluechat |
|------|-----------|----------|----------|
| 架构模式 | Concierge+ReAct | 任务队列 | 单体Agent |
| 上下文管理 | Notes系统 | 轮次结果 | XML状态 |
| 多轮对话 | @引用支持 | 选择机制 | 内置状态 |
| 适用场景 | 复杂创作任务 | 结构化分析 | 快速生成 |

## 注意事项

1. **会话隔离**: 每个session_id对应独立的Notes存储
2. **资源管理**: 长时间不活跃的会话会自动清理
3. **并发限制**: 单个用户同时请求数量有限制
4. **流式响应**: 需要正确处理SSE事件流 
