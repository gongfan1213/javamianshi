# Nova3 Supervisor LangSmith 链路追踪指南

## 概述

本文档说明了如何在 Nova3 Supervisor 中实现完整的 LangSmith 链路追踪功能，解决了多 Agent 队列执行时链路断开的问题。

## 问题背景

在原始实现中，Nova3 Supervisor 的执行流程如下：
1. 主 Agent 生成执行计划
2. 创建任务队列
3. 直接从队列执行子 Agent

这种设计导致 LangSmith 链路在队列执行时断开，只能追踪到主 Agent 的计划生成过程，无法看到子 Agent 的执行链路。

## 解决方案

### 1. 数据结构增强

#### TaskItem 增加追踪字段
```python
@dataclass
class TaskItem:
    # ... 原有字段 ...
    # LangSmith追踪信息
    langsmith_run_id: Optional[str] = None
    langsmith_parent_run_id: Optional[str] = None
```

#### TaskQueue 增加主链路信息
```python
@dataclass
class TaskQueue:
    # ... 原有字段 ...
    # LangSmith主链路追踪信息
    langsmith_master_run_id: Optional[str] = None
    langsmith_session_id: Optional[str] = None
```

### 2. 追踪链路架构

```
nova3_master_workflow (主链路)
├── nova3_generate_plan (计划生成)
├── nova3_insight_agent (洞察分析)
├── nova3_profile_agent (受众画像)
├── nova3_hitpoint_agent (内容打点)
├── nova3_facts_agent (事实收集)
└── nova3_xhs_writing_agent (小红书写作)
```

### 3. 核心功能实现

#### 主链路创建
在 `_generate_and_execute_plan` 方法中：
- 创建主工作流链路 (`nova3_master_workflow`)
- 创建计划生成子链路 (`nova3_generate_plan`)
- 为每个任务预分配 `run_id`

#### 子 Agent 链路继承
在 `_execute_next_task` 方法中：
- 创建子 Agent 追踪链路
- 继承主链路的 `parent_run_id`
- 传递追踪信息给子 Agent

#### 队列恢复链路重建
在 `_restore_or_create_tracing_context` 方法中：
- 检测队列是否已有追踪信息
- 为从队列恢复的执行创建新的主链路
- 重新分配未完成任务的追踪 ID

## 使用方法

### 1. 环境配置

```bash
# 启用 LangSmith 追踪
export LANGCHAIN_TRACING_V2=true
export LANGCHAIN_PROJECT=blueplan-research
export LANGCHAIN_API_KEY=your_api_key_here
```

### 2. 代码使用

```python
from agents.nova3.supervisor import Nova3Supervisor

# 创建 supervisor 实例
supervisor = Nova3Supervisor()

# 处理请求（自动启用链路追踪）
request_data = {
    "query": "分析小红书健身内容并写文章",
    "nova3_selections": {},
    "references": []
}

context = {
    "user_id": "user123",
    "session_id": "session456"
}

async for event in supervisor.process_request(request_data, context):
    # 处理事件
    pass
```

### 3. 运行测试

```bash
# 运行测试脚本
python test_langsmith_tracing.py
```

## 追踪信息说明

### 主链路 (nova3_master_workflow)
- **类型**: chain
- **输入**: 用户查询、选择、引用等
- **输出**: 完整的工作流结果
- **标签**: nova3_master, workflow, supervisor

### 计划生成链路 (nova3_generate_plan)
- **类型**: llm
- **父链路**: nova3_master_workflow
- **输入**: 系统提示词和用户提示词
- **输出**: 生成的执行计划
- **标签**: nova3_supervisor, generate_plan, llm

### 子 Agent 链路 (nova3_*_agent)
- **类型**: chain
- **父链路**: nova3_master_workflow
- **输入**: 任务输入、上下文信息
- **输出**: Agent 执行结果
- **标签**: nova3_sub_agent, {agent_name}, agent_execution

### 恢复链路 (nova3_resumed_workflow)
- **类型**: chain
- **用途**: 从队列恢复执行时创建
- **标签**: nova3_master, workflow, supervisor, resumed

## 监控和调试

### 1. 日志信息
系统会输出以下关键日志：
```
创建LangSmith主链路: {master_run_id}, session: {langsmith_session_id}
创建子agent LangSmith链路: {sub_agent_run_id} for {agent_name}
完成子agent LangSmith链路: {sub_agent_run_id}
完成LangSmith主链路: {master_run_id}
```

### 2. 错误处理
- 追踪创建失败不会影响业务逻辑
- 所有追踪操作都有异常捕获
- 失败信息会记录到日志中

### 3. LangSmith 控制台
在 LangSmith 控制台中可以看到：
- 完整的执行链路树
- 每个步骤的输入输出
- 执行时间和性能指标
- 错误信息和调试数据

## 最佳实践

### 1. 会话管理
- 使用有意义的 `user_id` 和 `session_id`
- 避免会话 ID 冲突

### 2. 项目组织
- 为不同环境使用不同的 `LANGCHAIN_PROJECT`
- 开发: blueplan-research-dev
- 测试: blueplan-research-test
- 生产: blueplan-research-prod

### 3. 性能考虑
- LangSmith 调用是异步的，不会阻塞业务逻辑
- 追踪信息会持久化到队列文件中
- 定期清理过期的队列文件

## 故障排查

### 1. 追踪未出现
- 检查环境变量设置
- 确认 API Key 有效
- 查看日志中的错误信息

### 2. 链路断开
- 检查队列持久化是否正常
- 确认任务的追踪 ID 分配
- 查看子 Agent 是否正确接收追踪信息

### 3. 性能问题
- 监控 LangSmith API 调用频率
- 检查网络连接状况
- 考虑调整追踪详细程度

## 更新历史

- **v1.0**: 初始实现，支持基本的主链路追踪
- **v1.1**: 添加子 Agent 链路继承
- **v1.2**: 实现队列恢复时的链路重建
- **v1.3**: 优化错误处理和日志记录 
