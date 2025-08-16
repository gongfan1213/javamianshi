# BluePlan Research

> 🤖 面向社交媒体内容创作的Gen-AI Agent系统

BluePlan Research 是一款**面向社交媒体内容创作者与营销运营人员**的一站式生成式智能解决方案，采用 `Supervisor + Researcher Agents` 架构，为创作与增长赋能。

## ✨ 核心特性

- **🎯 多平台支持**: 小红书 / 抖音 / 公众号内容创作
- **🧠 多Agent架构**: Nova3智能编排 + Loomi内容创作
- **🔍 Jina AI搜索**: 集成实时网络搜索，提供最新市场洞察
- **⚡ 流式响应**: 基于Server-Sent Events的实时内容生成
- **🔄 多模型支持**: OpenAI兼容API、Claude、通义千问等
- **📊 配置解耦**: LLM客户端、提示词、系统配置完全分离
- **💾 状态持久化**: 支持多轮对话和会话管理
- **🚀 生产就绪**: Docker部署、健康检查、性能监控

## 🏗️ 系统架构

### 多Agent智能架构

BluePlan Research 采用**模块化Agent架构**，支持两种主要模式：

#### 1. 🎯 Nova3模式 - 智能任务编排

```
Nova3Supervisor (智能编排/任务队列管理)
├── InsightAgent (洞察分析)
├── ProfileAgent (受众画像)  
├── HitpointAgent (内容打点)
├── FactsAgent (事实收集)
├── XHSWritingAgent (小红书创作)
├── TiktokScriptAgent (抖音口播稿)
└── WechatArticleAgent (公众号文章)
```

#### 2. 🚀 Loomi模式 - 内容创作专家

```
LoomiConcierge (智能接待员/Notes管理)
├── Orchestrator (ReAct任务编排)
├── InsightAgent (洞察分析)
├── ProfileAgent (受众画像)
├── HitpointAgent (内容打点)
├── XHSPostAgent (小红书创作)
├── TiktokScriptAgent (抖音口播稿)
└── WechatArticleAgent (公众号文章)
```

## 🚀 快速开始

### 环境要求

- Python 3.8+ （推荐 3.11+）
- Redis (已配置云端实例)
- 虚拟环境 `planvenv/` （项目已包含）

### 环境准备

```bash

# 1. 克隆项目
git clone <repository-url>
cd blueplan_research

# 2. 激活虚拟环境（推荐）
source planvenv/bin/activate

# 3. 安装/更新依赖（如需要）
pip install -r requirements.txt

# 4. 验证环境
python --version

# 5. 启动应用
source planvenv/bin/activate && python -m uvicorn apis.app:app --host 0.0.0.0 --port 8001 --reload
```

### 环境变量

**✅ 项目已包含完整的环境配置文件 `env`**

项目根目录下的 `env` 文件包含了所有必需的配置：

- OpenAI API配置（支持Claude模型）
- LangSmith配置
- Redis配置
- 系统配置

**配置说明:**

```bash
# 查看当前环境配置
cat env

# 配置文件会在启动时自动加载，无需手动设置环境变量
```

**如需自定义配置，修改 `env` 文件即可：**

```bash
# 编辑环境配置文件
nano env
```

### 配置信息

项目已预配置开发环境所需的API和数据库连接：

- **OpenAI兼容API**: `https:/xxxx/v1`
- **模型**: `gemini-2.5-pro`
- **Redis**: `rxxxxx9`
- **LangSmith项目**: `xxxxv`

### 启动服务

#### 方式1: 虚拟环境启动（推荐生产环境）

```bash
# 🚀 启动应用服务器
source planvenv/bin/activate && python -m uvicorn apis.app:app --host 0.0.0.0 --port 8001 --reload

# 或者带环境配置启动
source planvenv/bin/activate && python switch_env.py dev && python -m uvicorn apis.app:app --host 0.0.0.0 --port 8001 --reload

# 激活虚拟环境并启动调试模式
source planvenv/bin/activate && python main.py debug

# 激活虚拟环境并启动调试模式（自定义查询）
source planvenv/bin/activate && python main.py debug "小红书封面转化率低怎么办？"

# 使用uvicorn直接启动（开发模式）
source planvenv/bin/activate && python -m uvicorn apis.app:app --host 0.0.0.0 --port 8001 --reload
# 带配置启动
source planvenv/bin/activate && python switch_env.py dev && python -m uvicorn apis.app:app --host 0.0.0.0 --port 8001 --reload

# 方式3：远程部署模型
#### 部署普通
nohup bash -c "source planvenv/bin/activate && exec uvicorn apis.app:app --host 0.0.0.0 --port 8099" > uvicorn.log 2>&1 &
### 部署https ssl的方式
nohup bash -c "source planvenv/bin/activate && exec uvicorn apis.app:app --host 0.0.0.0 --port 8443 
 --ssl-keyfile app/zhengshu/airobotix.cn.key --ssl-certfile app/zhengshu/airobotix.cn.pem" > uvicorn.log 2>&1 &

### 重启
sudo lsof -i :8099 -t | xargs kill -9
sudo lsof -i :8443 -t | xargs kill -9

### 方式四 交互式测试

#### Nova3模式
```bash
source planvenv/bin/activate && python interactive_nova3.py
```

#### Loomi模式

```bash
source planvenv/bin/activate && python interactive_loomi.py
```

#### 方式2: 使用启动脚本

```bash
./start.sh api      # 启动API服务
./start.sh debug    # 调试模式
./start.sh test     # 运行测试
./start.sh help     # 查看帮助
```

#### 方式3: 直接运行（需确保依赖已安装）

```bash
python main.py           # 默认启动API服务器
python main.py server    # 启动API服务器
python main.py debug     # 调试模式
python main.py help      # 查看帮助信息
```

### 访问服务

- **API服务**: http://localhost:8000
- **API文档**: http://localhost:8000/docs
- **健康检查**: http://localhost:8000/health

### 快速测试

#### 测试调试模式

```bash
# 1. 激活虚拟环境并测试默认查询
source planvenv/bin/activate && python main.py debug

# 2. 激活虚拟环境并测试自定义查询
source planvenv/bin/activate && python main.py debug "我的小红书账号流量上不去，帮我分析下原因"

# 3. 测试其他查询类型
source planvenv/bin/activate && python main.py debug "小红书封面转化率低怎么办？"
source planvenv/bin/activate && python main.py debug "帮我写一篇抖音烘焙文案"
```

**预期输出示例：**

```
🧠 BluePlan Research - 调试模式
📅 查询时间: 2025-01-14 10:30:00
💬 用户查询: 我的小红书账号流量上不去，帮我分析下原因
📝 背景信息:
   - account_name: 我的手工烘焙坊
   - platform: 小红书
   - target_audience: 20-30岁，喜欢DIY和甜品的女性

🔔 [系统] [BluePlanAPI]: 连接建立，开始处理请求...
📋 [计划] [SupervisorAgent]: 1. 分析账号当前内容和人设定位...
💡 [洞察] [UserInsightAgent]: 根据账号背景分析...
🎯 [总结] [SupervisorAgent]: 综合分析完成...
```

#### 测试API服务

```bash
# 1. 启动API服务器
source planvenv/bin/activate && python -m uvicorn apis.app:app --host 0.0.0.0 --port 8001 --reload

# 2. 在另一个终端测试API（需要安装curl）
curl -X POST "http://localhost:8001/novachat" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "test_user", 
    "session_id": "test_session",
    "request_data": {
      "query": "帮我分析小红书账号优化策略",
      "background": {
        "account_name": "测试账号",
        "platform": "小红书"
      }
    }
  }'
```

### 故障排除

#### 虚拟环境问题

```bash
# 如果虚拟环境不存在或损坏，重新创建
python -m venv planvenv
source planvenv/bin/activate
pip install -r requirements.txt

# 检查虚拟环境状态
source planvenv/bin/activate && which python
source planvenv/bin/activate && pip list
```

#### 常见启动错误

**1. 依赖缺失：**

```bash
# 重新安装依赖
source planvenv/bin/activate && pip install -r requirements.txt --upgrade
```

**2. 端口被占用：**

```bash
# 检查端口占用
lsof -i :8000

# 使用其他端口启动
source planvenv/bin/activate && python -c "
import sys; sys.path.append('.');
from apis.app import app;
import uvicorn;
uvicorn.run(app, host='0.0.0.0', port=8001)
"
```

**3. 配置文件问题：**

```bash
# 检查配置文件
source planvenv/bin/activate && python -c "
from config.settings import Settings;
print('配置加载:', Settings())
"
```

## 📡 API使用

### 核心接口

#### 1. `/novachat` - Nova3智能编排模式

**POST** `/novachat` - Nova3任务队列智能编排接口

#### 2. `/loomichat` - Loomi内容创作模式

**POST** `/loomichat` - Loomi智能接待员和内容创作接口

```json
{
  "user_id": "user_123",
  "session_id": "session_456", 
  "request_data": {
    "query": "我的小红书账号流量上不去，帮我分析下原因",
    "background": {
      "account_name": "我的手工烘焙坊",
      "platform": "小红书",
      "target_audience": "20-30岁，喜欢DIY和甜品的女性"
    },
    "interaction_type": "full_analysis"
  }
}
```

**响应**: Server-Sent Events 流式数据

```json
{
  "event_type": "llm_chunk",
  "agent_source": "SupervisorAgent",
  "timestamp": "2025-01-23T10:30:00.123Z",
  "payload": {
    "content_type": "plan_step",
    "data": "1. 分析账号当前内容和人设定位..."
  }
}
```

#### BlueChat API 示例

```json
{
  "user_id": "xhs_user_123",
  "session_id": "session_456", 
  "request_data": {
    "query": "我想做美食博主，给我一些建议",
    "background": {
      "platform": "小红书",
      "industry": "美食",
      "target_audience": "年轻人群"
    }
  }
}
```

**BlueChat响应特色**:

```json
{
  "event_type": "llm_chunk",
  "agent_source": "xhs_master_agent",
  "payload": {
    "content_type": "structured_proposals",
    "data": [
      {
        "title": "家常菜变身记",
        "core_selling_point": "普通食材的高级做法",
        "expected_effect": "新手也能做出大师级美食",
        "content_format": "图文教程",
        "difficulty_level": "⭐⭐⭐"
      }
    ]
  }
}
```

### 内容类型标签

| content_type     | 说明          | 来源Agent                    |
| ---------------- | ------------- | ---------------------------- |
| `plan_step`    | 执行计划步骤  | SupervisorAgent              |
| `insight`      | 市场/用户洞察 | ContentStrategy, UserInsight |
| `persona`      | 受众画像分析  | UserInsight                  |
| `case_study`   | 爆款案例分析  | FactCollection               |
| `suggestion`   | 优化建议      | DiagnoseAvatar               |
| `copywriting`  | 文案内容      | XHSWriter, MPWriter          |
| `final_answer` | 最终报告      | SupervisorAgent              |

## 🧪 测试示例

```python
# test_example.py
import asyncio
from main import BluePlanApp

async def test_chat():
    app = BluePlanApp()
  
    query = "帮我写一篇小红书烘焙笔记"
    background = {
        "account_name": "甜品小厨房",
        "platform": "小红书",
        "target_audience": "年轻女性烘焙爱好者"
    }
  
    await app.chat_debug(query, background)

if __name__ == "__main__":
    asyncio.run(test_chat())
```

## 📁 项目结构

```
blueplan_research/
├── interactive_nova3.py       # Nova3交互式入口
├── interactive_loomi.py       # Loomi交互式入口
├── start.sh                   # 启动脚本
├── requirements.txt           # Python依赖
├── agents/                    # Agent模块
│   ├── base_agent.py          # Agent基类
│   ├── nova3/                 # Nova3模式Agent
│   ├── loomi/                 # Loomi模式Agent
│   ├── user_insight.py        # 用户洞察Agent
│   ├── content_strategy.py    # 内容策略Agent
│   ├── fact_collection.py     # 事实搜集Agent
│   ├── diagnose_avatar.py     # 视觉诊断Agent
│   ├── writer_xhs.py          # 小红书文案Agent
│   └── writer_mp_dy.py        # 公众号/抖音文案Agent
├── apis/                      # API服务模块
│   ├── app.py                 # FastAPI应用
│   ├── routes.py              # API路由
│   ├── schemas.py             # 数据模型
│   ├── middleware.py          # 中间件
│   └── health.py              # 健康检查
├── config/                    # 配置管理
│   ├── settings.py            # 配置类
│   ├── app_config.yaml        # 应用配置
│   └── llm_config.yaml        # LLM配置
├── prompts/                   # 提示词库
│   ├── supervisor_system_new.txt # BlueChat核心提示词 🔥
│   ├── supervisor_*.txt       # 监督者提示词
│   ├── user_insight_*.txt     # 用户洞察提示词
│   ├── prompt_manager.py      # 提示词管理器
│   └── ...                    # 其他Agent提示词
├── utils/                     # 工具模块
│   ├── llm_client.py          # LLM客户端
│   ├── web_search.py          # Jina AI搜索工具 🔥
│   ├── logger.py              # 日志工具
│   └── error_handler.py       # 错误处理
└── logs/                      # 日志目录
```

## 🔧 配置说明

### LLM配置 (config/llm_config.yaml)

```yaml
llm:
  default_provider: "openai"
  providers:
    openai:
      api_key: "${OPENAI_API_KEY}"
      model: "${OPENAI_MODEL:-gemini-2.5-pro}"
      base_url: "${OPENAI_API_BASE:-https://bmc-llm-relay.bluemediagroup.cn/v1}"
      temperature: 0.7
      max_tokens: 4096
```

### 应用配置 (config/app_config.yaml)

```yaml
app:
  name: "BluePlan Research"
  debug: true
  log_level: "INFO"

memory:
  store_type: "redis"
  redis_host: "${REDIS_HOST}"
  redis_port: "${REDIS_PORT}"
  redis_password: "${REDIS_PASSWORD}"
```

## 🐳 Docker部署

```bash
# 构建镜像
docker build -t blueplan-research .

# 运行容器
docker run -p 8000:8000 \
  -e OPENAI_API_KEY=your_key \
  -e REDIS_PASSWORD=your_password \
  blueplan-research
```

## 📊 监控和日志

- **健康检查**: `/health`, `/health/ready`, `/health/metrics`
- **日志文件**: `logs/blueplan.log`
- **性能指标**: Agent执行时间、成功率、内容生成统计

## 🛠️ 开发指南

### 添加新Agent

1. 继承 `BaseAgent` 类
2. 实现 `process_request` 方法
3. 创建对应的提示词文件
4. 在 `SupervisorAgent` 中注册

### 📝 提示词管理系统

**🔄 已升级为Python模块化管理**

**为什么不再使用txt文件？**

1. **解决空文件问题** - 原来很多txt文件只有1字节，缺少实际内容
2. **统一管理** - Python模块提供更好的组织和维护性
3. **类型安全** - IDE支持、自动补全、语法检查
4. **动态变量** - 灵活的变量替换和模板渲染

**新的结构：**

```python
# prompts/prompt_manager.py - 统一管理所有提示词
class PromptManager:
    def get_system_prompt(self, agent_name: str) -> str
    def get_user_prompt(self, agent_name: str, **kwargs) -> str
```

**特点：**

- ✅ 7个Agent的完整系统提示词
- ✅ 灵活的用户提示词模板
- ✅ 专业的内容策略和创作指导
- ✅ 支持动态变量替换

详情查看：[prompts/README.md](prompts/README.md)

## 🤖 Agent详细说明

### 🚀 BlueChat - 小红书爆款搭档 (推荐)

**单一全能Agent，专为小红书内容创作打造**

#### 🎯 工作流程

1. **需求诊断阶段** - 通过提问深入了解用户背景和需求
2. **策略提案阶段** - 生成结构化的内容策略提案
3. **内容生成阶段** - 输出爆款文案和创作建议

#### 🔍 搜索能力

- **Jina AI集成**: 实时搜索最新市场趋势
- **三种搜索策略**: 小红书洞察、趋势分析、案例研究
- **上下文感知**: 基于用户背景优化搜索结果

#### 💾 会话管理

- **多用户支持**: 独立的用户会话管理
- **状态持久化**: 对话历史和状态自动保存
- **上下文保持**: 记住最近10轮对话内容

## 🔍 Jina AI搜索集成

### 配置信息

- **API Key**: `xxxx`
- **Base URL**: `https://r.jina.ai/`
- **搜索深度**: 每次查询最多10个结果

### 搜索策略

1. **小红书洞察搜索**: 专注平台特定内容和趋势
2. **趋势搜索**: 行业最新发展和热点话题
3. **案例搜索**: 成功案例和最佳实践

### 技术特点

- **异步处理**: 非阻塞的搜索请求
- **智能解析**: 自动提取关键信息
- **频率控制**: 防止API调用过频
- **错误处理**: 完善的异常处理机制

详细集成指南请参考: [JINA_AI_INTEGRATION.md](JINA_AI_INTEGRATION.md)

## 🤝 贡献指南

1. Fork 项目
2. 创建功能分支
3. 提交代码变更
4. 发起 Pull Request

## 📄 许可证

MIT License

---

**🧠 BluePlan Research - 让每一次内容创作都更智能、更精准**

# Token统计功能

## 功能概述

为BluePlan Research系统添加了完整的Token消耗统计功能，包括：

### 📊 统计指标
- **日Token消耗**：每日Token使用量统计
- **月Token消耗**：每月Token使用量统计  
- **用户Token排行榜**：按用户ID Token消耗排序的每日前10名

### 🔧 API端点

#### 1. 每日Token统计
```bash
GET /api/token-stats/daily?days=7
```

#### 2. 月度Token统计
```bash
GET /api/token-stats/monthly?months=3
```

#### 3. 用户Token排行榜
```bash
GET /api/token-stats/user-ranking?target_date=2024-01-01&top_n=10
```

#### 4. Token统计仪表盘数据
```bash
GET /api/token-stats/dashboard
```

### 📱 前端界面

访问 `/dashboard` 页面可以查看：
- Token消耗统计卡片
- 每日Token消耗趋势图表
- 月度Token消耗趋势图表
- 用户Token消耗排行榜

### 💾 数据存储

- 基于现有的`token_accumulator`系统
- 数据存储在Redis中，键格式：`token_accumulator:{user_id}:{session_id}`
- 支持Nova3和Loomi两个系统的统一统计

### 🔄 数据更新

- 实时更新：每次LLM调用都会更新Token统计
- 自动刷新：仪表盘页面每30秒自动刷新数据
- 历史数据：保留最近7天的详细数据，3个月的月度数据

### 📈 使用示例

```python
# 获取Token统计数据
from utils.token_accumulator import token_accumulator

# 获取最近7天的Token统计
daily_stats = await token_accumulator.get_daily_token_stats(7)

# 获取用户排行榜
ranking = await token_accumulator.get_daily_user_token_ranking(top_n=10)

# 获取仪表盘数据
dashboard_data = await token_accumulator.get_token_dashboard_data()
```
