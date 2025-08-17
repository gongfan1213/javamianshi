# chat_messages表数据存储说明

## 表结构
```sql
create table public.chat_messages (
  id uuid not null default gen_random_uuid (),
  project_id uuid not null,
  role text not null,
  content text not null,
  llm_raw_output jsonb null,
  associated_card_id uuid null,
  created_at timestamp with time zone not null default now(),
  function_tools jsonb null default '[]'::jsonb,
  reference_tools jsonb null default '[]'::jsonb,
  auto_select_tools jsonb null default '[]'::jsonb,
  constraint chat_messages_pkey primary key (id),
  constraint chat_messages_associated_card_id_fkey foreign KEY (associated_card_id) references cards (id) on delete set null,
  constraint chat_messages_project_id_fkey foreign KEY (project_id) references projects (id) on delete CASCADE,
  constraint chat_messages_role_check check (
    (
      role = any (
        array['user'::text, 'assistant'::text, 'system'::text]
      )
    )
  )
);
```

## 连接配置
```python
# pip install supabase
from supabase import create_client

supabase = create_client(
    "https://auth.loomi.live",
    "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImV2cGN6dnd5Z2VscnZ4emZkY2d2Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3NDk4OTU3ODUsImV4cCI6MjA2NTQ3MTc4NX0.y7uP6NVj48UAKnMWcB_5LltTVCVFuSeo7xmrCEHlp1I"
)
```

## 数据存储方式

### 1. 用户消息存储
```python
def save_user_message(project_id: str, content: str):
    """存储用户消息 - 简单模式"""
    data = {
        'project_id': project_id,
        'role': 'user',
        'content': content,
        'function_tools': [],
        'reference_tools': [],
        'auto_select_tools': []
    }
    
    response = supabase.table('chat_messages').insert(data).execute()
    return response.data[0] if response.data else None
```

### 2. AI助手消息存储 - 带工具调用
```python
def save_assistant_message(project_id: str, content: str, function_tools_data: list):
    """存储AI助手消息 - 完整模式"""
    data = {
        'project_id': project_id,
        'role': 'assistant', 
        'content': content,
        'function_tools': function_tools_data,  # 重点：工具调用数据
        'reference_tools': [],
        'auto_select_tools': []
    }
    
    response = supabase.table('chat_messages').insert(data).execute()
    return response.data[0] if response.data else None
```

### 3. function_tools字段数据格式
基于您的实际数据，`function_tools`应该这样存储：

```python
# 示例：存储带有多个工具调用的消息
function_tools_data = [
    {
        "id": 1753081507585,
        "tool": "nova3_concierge", 
        "raw_output": {
            "id": 1753081507585,
            "data": [{
                "id": "concierge1",
                "type": "message", 
                "content": "收到！我们马上启动分析。请稍等片刻，我会带着第一手的洞察回来向您汇报。"
            }],
            "metadata": {
                "raw_response": "收到！我们马上启动分析...",
                "concierge_count": 1
            },
            "content_type": "nova3_concierge"
        },
        "tool_input": "",
        "observation": "",
        "tool_labels": {}
    },
    {
        "id": 1753081507586,
        "tool": "nova3_observe_think",
        "raw_output": {
            "id": 1753081507586,
            "data": [{
                "id": "insight1",
                "content": "基于小红书平台特性分析..."
            }],
            "content_type": "nova3_observe_think"
        },
        "tool_input": "",
        "observation": "",
        "tool_labels": {}
    }
]

# 存储消息
save_assistant_message(
    project_id="your-project-id",
    content='<nova3_tools id="1753081507585" name="nova3_concierge"></nova3_tools><nova3_tools id="1753081507586" name="nova3_observe_think"></nova3_tools>',
    function_tools_data=function_tools_data
)
```

## 完整存储类
```python
class ChatMessageStorage:
    def __init__(self):
        self.supabase = supabase
    
    def store_conversation(self, project_id: str, user_input: str, ai_response: dict):
        """存储一轮完整对话"""
        
        # 1. 存储用户消息
        user_msg = self.supabase.table('chat_messages').insert({
            'project_id': project_id,
            'role': 'user',
            'content': user_input,
            'function_tools': [],
            'reference_tools': [],
            'auto_select_tools': []
        }).execute()
        
        # 2. 存储AI回复消息
        assistant_msg = self.supabase.table('chat_messages').insert({
            'project_id': project_id,
            'role': 'assistant',
            'content': ai_response.get('content', ''),
            'function_tools': ai_response.get('function_tools', []),
            'reference_tools': ai_response.get('reference_tools', []),
            'auto_select_tools': ai_response.get('auto_select_tools', []),
            'llm_raw_output': ai_response.get('llm_raw_output')
        }).execute()
        
        return {
            'user_message_id': user_msg.data[0]['id'],
            'assistant_message_id': assistant_msg.data[0]['id']
        }
    
    def get_chat_history(self, project_id: str):
        """获取对话历史"""
        response = self.supabase.table('chat_messages')\
            .select('*')\
            .eq('project_id', project_id)\
            .order('created_at')\
            .execute()
        
        return response.data

# 使用方式
storage = ChatMessageStorage()

# 存储对话
ai_response = {
    'content': '<nova3_tools id="123" name="nova3_concierge"></nova3_tools>',
    'function_tools': [
        {
            "id": 123,
            "tool": "nova3_concierge",
            "raw_output": {
                "id": 123,
                "data": [{"id": "msg1", "content": "AI回复内容"}],
                "content_type": "nova3_concierge"
            },
            "tool_input": "",
            "observation": "",
            "tool_labels": {}
        }
    ]
}

result = storage.store_conversation(
    project_id="your-project-id",
    user_input="用户问题", 
    ai_response=ai_response
)
```

## 关键点

1. **function_tools** - 存储工具调用的完整数据，包括raw_output
2. **content** - 存储带有工具标签的内容，如`<nova3_tools id="123" name="tool_name"></nova3_tools>`
3. **project_id** - 必须是有效的项目ID
4. **role** - 必须是'user'、'assistant'或'system'之一

这样您的Python项目就能正确存储和读取chat_messages表的数据了。
