# Gemini 客户端迁移指南

从 `vertexai` SDK 迁移到 `google-genai` 库

## 概述

本指南帮助您从现有的 `vertexai` SDK 迁移到新的 `google-genai` 库。新的库提供了更统一的API接口，同时支持 Gemini Developer API 和 Vertex AI。

## 主要变化

### 1. 依赖库变化

**旧版 (vertexai):**
```bash
pip install google-cloud-aiplatform
```

**新版 (google-genai):**
```bash
pip install google-genai
```

### 2. 导入变化

**旧版:**
```python
import vertexai
from vertexai.generative_models import GenerativeModel
import vertexai.preview.generative_models as generative_models
```

**新版:**
```python
from google import genai
from google.genai import types
```

### 3. 客户端初始化变化

**旧版:**
```python
vertexai.init(project=project_id, location=location)
model = GenerativeModel(model_name)
```

**新版:**
```python
# Vertex AI 模式
client = genai.Client(
    vertexai=True, 
    project=project_id, 
    location=location
)

# 或 Gemini Developer API 模式
client = genai.Client(api_key='your-api-key')
```

### 4. API 调用变化

**旧版:**
```python
response = model.generate_content(
    prompt,
    generation_config=generation_config,
    safety_settings=safety_settings
)
```

**新版:**
```python
response = client.models.generate_content(
    model=model_name,
    contents=prompt,
    config=config
)
```

## 文件对比

| 功能 | 旧版文件 | 新版文件 |
|------|----------|----------|
| Gemini 客户端 | `utils/gemini_client.py` | `utils/gemini_client_gen.py` |
| 测试脚本 | 无 | `test_gemini_gen.py` |

## 功能对照表

| 功能 | 旧版方法 | 新版方法 | 状态 |
|------|----------|----------|------|
| 文本生成 | `generate_text()` | `generate_text()` | ✅ 完全兼容 |
| 流式聊天 | `stream_chat()` | `stream_chat()` | ✅ 完全兼容 |
| 非流式聊天 | `chat()` | `chat()` | ✅ 完全兼容 |
| 带使用信息的聊天 | `chat_with_usage()` | `chat_with_usage()` | ✅ 完全兼容 |
| 模型信息 | `get_model_info()` | `get_model_info()` | ✅ 完全兼容 |
| Token估算 | `_estimate_tokens()` | `_estimate_tokens()` | ✅ 完全兼容 |
| 缓存检测 | ✅ 支持 | ✅ 支持 | ✅ 完全兼容 |

## 配置变化

### 环境变量

**新增支持的环境变量:**
- `GOOGLE_API_KEY`: Gemini Developer API 密钥
- `GEMINI_API_KEY`: 备用的 API 密钥名称  
- `GEMINI_LOCATION_GENAI`: 新版客户端专用的区域配置 (推荐: `global`，也可用: `us-central1`, `asia-northeast1`)

**继续支持的环境变量:**
- `GOOGLE_APPLICATION_CREDENTIALS`: Vertex AI 认证文件路径
- `GCLOUD_PROJECT`: Google Cloud 项目ID

### 认证方式

新版支持两种认证方式：

1. **Gemini Developer API** (推荐用于开发)
   ```python
   client = genai.Client(api_key='your-api-key')
   ```

2. **Vertex AI** (推荐用于生产)
   ```python
   client = genai.Client(
       vertexai=True,
       project='your-project-id',
       location='us-central1'
   )
   ```

## 迁移步骤

### 1. 安装新依赖

```bash
pip install google-genai
```

### 2. 更新导入

将代码中的导入语句更新：

```python
# 旧版
from utils.gemini_client import GeminiClient

# 新版
from utils.gemini_client_gen import GeminiClientGen
```

### 3. 更新实例化

```python
# 旧版
client = GeminiClient(config)

# 新版
client = GeminiClientGen(config)
```

### 4. 测试功能

运行测试脚本验证功能：

```bash
python test_gemini_gen.py
```

## 优势

### 新版本的优势

1. **统一API**: 支持 Gemini Developer API 和 Vertex AI
2. **更好的错误处理**: 改进的异常处理和错误消息
3. **简化的配置**: 自动检测认证方式
4. **向前兼容**: 与旧版API完全兼容
5. **更好的文档**: 基于最新的Google GenAI SDK

### 缓存功能保持

- ✅ 继续支持上下文缓存检测
- ✅ 继续支持token使用统计
- ✅ 继续支持缓存命中率计算

### 安全配置支持

- ✅ 完全支持加密的 `bluelab.json` 配置文件
- ✅ 自动创建临时认证文件并设置安全权限 (600)
- ✅ 自动清理临时认证文件
- ✅ 多层级认证回退机制

## 配置示例

### 项目配置文件

在 `config/app_config.yaml` 中：

```yaml
gemini:
  project_id: "bluemedia-db-bluelab"
  location: "global"
  
llm_providers:
  gemini:
    model: "gemini-2.0-flash-001"
    temperature: 0.7
    max_tokens: 8192
    api_key: ""  # 可选，也可通过环境变量设置
```

### 环境变量设置

```bash
# 方式1: 使用 Gemini Developer API
export GOOGLE_API_KEY="your-gemini-api-key"

# 方式2: 使用 Vertex AI
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"
export GCLOUD_PROJECT="bluemedia-db-bluelab"

# 新版客户端专用区域配置 (可选，推荐使用global)
export GEMINI_LOCATION_GENAI="global"  # 推荐配置，或使用其他区域如 us-central1, asia-northeast1
```

### 安全配置设置

新版客户端完全支持加密的 `bluelab.json` 配置文件：

1. **自动检测加密配置**
   - 客户端会自动尝试从安全配置管理器加载配置
   - 支持 `config/bluelab.bin` 和 `config/bluelab.enc` 等加密文件

2. **临时认证文件管理**
   ```python
   # 客户端会自动：
   # 1. 解密配置文件
   # 2. 创建临时 JSON 认证文件
   # 3. 设置安全权限 (600)
   # 4. 设置环境变量 GOOGLE_APPLICATION_CREDENTIALS
   # 5. 注册清理函数确保临时文件被删除
   ```

3. **认证优先级**
   ```
   1. GOOGLE_API_KEY / GEMINI_API_KEY (环境变量)
   2. provider_config.api_key (配置文件)
   3. 安全配置管理器 (加密配置)
   4. 传统 bluelab.json 文件
   5. 默认 Google Cloud 认证
   ```

## 故障排除

### 常见问题

1. **ImportError: 无法导入 google.genai**
   ```bash
   pip install google-genai
   ```

2. **认证失败**
   - 检查 API 密钥是否正确设置
   - 检查 Vertex AI 认证文件路径
   - 检查项目ID和区域设置

3. **模型不可用**
   - 确认使用的模型名称是否正确
   - 检查模型在指定区域是否可用

### 调试技巧

1. 启用详细日志：
   ```python
   import logging
   logging.basicConfig(level=logging.DEBUG)
   ```

2. 检查客户端状态：
   ```python
   model_info = client.get_model_info()
   print(model_info)
   ```

## 向后兼容性

新版客户端 `GeminiClientGen` 与旧版 `GeminiClient` 完全兼容：

- ✅ 相同的方法签名
- ✅ 相同的返回值格式
- ✅ 相同的错误处理方式
- ✅ 相同的配置选项

## 支持

如果在迁移过程中遇到问题，请：

1. 查看错误日志
2. 检查配置文件
3. 运行测试脚本
4. 查看本指南的故障排除部分

## 结论

新的 `google-genai` 库提供了更现代、更统一的API接口，同时保持了与现有代码的完全兼容性。迁移过程简单，只需要更换导入和客户端类名即可。 
