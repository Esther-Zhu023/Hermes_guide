# 第七章 Agent Loop 内核剖析

> 本章目标：深入理解 Hermes 的核心执行机制，掌握请求生命周期、事件驱动模型和工具调度原理
> 前置知识：第三章工具系统、第二章配置体系

---

## 7.1 请求流转

### 7.1.1 消息完整生命周期追踪

```
用户输入 → 网关接收 → 协议归一化 → 排队 → 提示词装配 → LLM推理 → 工具执行 → 结果回注 → 流式输出
```

### 7.1.2 四层架构

```
┌─────────────────────────────────────────┐
│  Layer 4: Capability Layer (能力层)     │
│  - 工具注册表 (registry)                │
│  - 工具集 (toolsets)                    │
│  - 记忆系统 (memory)                    │
├─────────────────────────────────────────┤
│  Layer 3: Execution Layer (执行层)      │
│  - 工具调度 (dispatch)                  │
│  - 环境抽象 (terminal backend)          │
│  - 安全审批 (approval)                  │
├─────────────────────────────────────────┤
│  Layer 2: Control Layer (控制层)         │
│  - 提示词装配 (prompt assembly)         │
│  - 上下文管理 (context management)       │
│  - 流式处理 (streaming)                  │
├─────────────────────────────────────────┤
│  Layer 1: Entry Layer (入口层)           │
│  - 网关 (gateway)                       │
│  - 平台适配器 (platform adapters)       │
│  - 协议归一化 (protocol normalization)   │
└─────────────────────────────────────────┘
```

### 7.1.3 端到端消息链路图

```
Telegram/Discord/Slack/飞书...
         │
         ▼
    ┌─────────┐
    │ Gateway │ ← Layer 1: 协议归一化
    └────┬────┘
         │ Message (normalized)
         ▼
    ┌─────────┐
    │ Session │ ← 会话管理、历史记录
    └────┬────┘
         │ Context
         ▼
    ┌─────────────┐
    │  Prompt    │ ← Layer 2: 提示词装配
    │  Assembly  │   - System prompt
    └──────┬──────┘   - 历史消息
           │           - 工具定义
           ▼           - 用户输入
    ┌─────────────┐
    │   LLM API   │ ← 模型推理
    │   (OpenAI/  │
    │  Anthropic) │
    └──────┬──────┘
           │ Tool Call / Response
           ▼
    ┌─────────────┐
    │  Tool      │ ← Layer 3: 工具执行
    │  Dispatch  │
    └──────┬──────┘
           │ Result
           ▼
    ┌─────────────┐
    │  Result    │ ← Layer 2: 结果回注
    │  Injection │   - 追加到上下文
    └──────┬──────┘
           │ Next turn
           ▼
    ┌─────────────┐
    │  Streaming │ ← Layer 2: 流式输出
    │  Output    │
    └──────┬──────┘
           │
           ▼
    返回上游平台
```

### 7.1.4 分层排障策略

| 层级 | 故障表现 | 排查命令 |
|------|----------|----------|
| Layer 1 | 消息未收到 | `hermes platform <platform> --test` |
| Layer 2 | 无响应/格式错误 | `hermes logs --filter "prompt"` |
| Layer 3 | 工具执行失败 | `hermes logs --filter "dispatch"` |
| Layer 4 | 工具不可用 | `hermes tools --check` |

---

## 7.2 嵌入式集成

### 7.2.1 Hermes 如何嵌入 LLM 运行时

Hermes 通过标准的 OpenAI API 格式与 LLM 交互：

```python
# 核心集成代码 (run_agent.py)
from openai import OpenAI

client = OpenAI(
    base_url=config.base_url,  # OpenRouter / Anthropic / 自托管
    api_key=config.api_key
)

response = client.chat.completions.create(
    model=config.model_name,
    messages=messages,  # [system, ...history, user_input]
    tools=tool_definitions,  # 工具 schema
    stream=True
)
```

### 7.2.2 工具注入机制

工具通过 `tools` 参数注入到模型：

```python
# 工具定义格式 (OpenAI兼容)
tools = [
    {
        "type": "function",
        "function": {
            "name": "web_search",
            "description": "搜索网络获取信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string"}
                }
            }
        }
    }
]
```

工具发现流程：
```
1. tools/*.py 模块导入
2. 自注册: registry.register(tool_def, handler, metadata)
3. get_tool_definitions() 收集所有工具
4. 传递给 LLM API
```

### 7.2.3 事件订阅

Hermes 支持插件系统的事件订阅：

```python
# 插件可以订阅的事件
hooks = [
    "pre_tool_call",       # 工具调用前
    "post_tool_call",      # 工具调用后
    "transform_tool_result",  # 结果转换
    "on_message",         # 消息接收
    "on_response",        # 响应发送
]
```

使用示例：
```python
# hermes_cli/plugins.py
def my_plugin():
    def pre_tool_call(tool_name, args, **kwargs):
        if tool_name == "terminal":
            log_command(args["command"])
    return {"pre_tool_call": pre_tool_call}
```

---

## 7.3 事件驱动模型

### 7.3.1 事件总线设计

```
┌────────────────────────────────────────┐
│           Event Bus                     │
├────────────────────────────────────────┤
│  on_message → on_llm_call → on_tool   │
│      ↓           ↓           ↓         │
│  on_tool_result → on_response → ...   │
└────────────────────────────────────────┘
```

### 7.3.2 Tick 机制

定时任务使用 Tick 机制检查：

```python
# cronjob_tools.py 核心逻辑
class CronScheduler:
    def tick(self):
        for job in self.pending_jobs:
            if job.should_run(now()):
                self.enqueue(job)
```

调度间隔配置：
```yaml
cron:
  check_interval: 60  # 每60秒检查一次
```

### 7.3.3 状态恢复

Hermes 使用 SQLite 持久化状态：

```python
# hermes_state.py 核心结构
class HermesState:
    sessions: Dict[str, Session]
    active_tasks: Dict[str, Task]
    cron_jobs: List[CronJob]

    def save(self):
        # 序列化到 ~/.hermes/state.db

    def restore(self):
        # 从数据库恢复状态
```

---

## 7.4 入口与排队

### 7.4.1 协议归一化

各平台消息转换为统一格式：

```python
# 归一化消息结构
NormalizedMessage = {
    "platform": str,      # telegram, discord, etc.
    "chat_id": str,
    "user_id": str,
    "text": str,
    "timestamp": datetime,
    "metadata": dict     # 平台特有数据
}
```

### 7.4.2 幂等控制（防重放）

使用 `tool_call_id` 实现幂等：

```python
# 防止重复执行
executed_calls = set()

def execute_tool(call):
    if call.id in executed_calls:
        return cached_result(call.id)
    result = do_execute(call)
    executed_calls.add(call.id)
    return result
```

### 7.4.3 分道排队

```
                    ┌─────────────┐
                    │   Request   │
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ Session  │    │  Global  │    │ Background│
    │   Queue  │    │   Queue  │    │   Queue   │
    │ (单聊)    │    │ (群聊)   │    │ (定时任务) │
    └──────────┘    └──────────┘    └──────────┘
```

配置：
```yaml
# config.yaml
queue:
  session_queue_size: 10
  global_queue_size: 50
  background_queue_size: 100
```

### 7.4.4 反压与超时

```python
# 反压策略
class BackpressureHandler:
    def should_reject(self):
        return (
            self.queue_size > self.max_queue or
            self.active_tasks > self.max_concurrent
        )

    def get_timeout(self, attempt: int) -> int:
        return min(30 * (2 ** attempt), 300)  # 指数退避
```

---

## 7.5 提示词装配

### 7.5.1 动态 System Prompt

System Prompt 根据上下文动态生成：

```python
def build_system_prompt(config, session):
    parts = []

    # 基础人格
    parts.append(config.personality)

    # 工具说明
    parts.append(build_tool_section(session.enabled_toolsets))

    # 记忆上下文
    if session.has_memory():
        parts.append(session.get_relevant_memory())

    # 技能说明
    parts.append(build_skills_section())

    return "\n\n".join(parts)
```

### 7.5.2 上下文预算与裁剪

```python
class ContextBudget:
    MAX_TOKENS = 200000

    def fit(self, messages):
        while self.used_tokens > self.MAX_TOKENS:
            self.prune_oldest_messages()

    def prune_oldest_messages(self):
        # 保留 system prompt 和最近消息
        # 中间消息 LLM 摘要后保留
        pass
```

配置：
```yaml
model:
  max_context_tokens: 200000
  context_compression_threshold: 150000
  compression_model: gpt-4o-mini
```

### 7.5.3 注入防御

防止 Prompt 注入攻击：

```python
# 检测用户输入中的注入尝试
def detect_injection(text: str) -> bool:
    injection_patterns = [
        r"ignore previous instructions",
        r"disregard.*system",
        r"you are now.*assistant",
    ]
    return any(re.match(p, text) for p in injection_patterns)

# 如果检测到注入，标记但不直接拒绝
# 让 LLM 自己判断如何处理
```

---

## 7.6 工具执行

### 7.6.1 工具签名与调度

```python
# registry.py 核心结构
class ToolRegistry:
    def register(self, name, schema, handler, metadata):
        self._tools[name] = {
            "schema": schema,
            "handler": handler,
            "metadata": metadata
        }

    def dispatch(self, name, args, **kwargs):
        tool = self._tools.get(name)
        if not tool:
            raise ToolNotFoundError(name)
        return tool["handler"](args, **kwargs)
```

### 7.6.2 结果回注

工具结果追加到对话历史：

```python
# run_agent.py 中的循环
while True:
    response = client.chat.completions.create(
        messages=conversation,
        tools=tools
    )

    if response.finish_reason == "tool_calls":
        for call in response.tool_calls:
            result = registry.dispatch(call.name, call.args)
            conversation.append({
                "role": "tool",
                "tool_call_id": call.id,
                "content": result
            })
    else:
        return response
```

### 7.6.3 大尺寸结果裁剪

```python
MAX_RESULT_SIZE = 50000  # 字符

def truncate_result(result: str) -> str:
    if len(result) > MAX_RESULT_SIZE:
        return (
            f"[结果已截断，原长度 {len(result)} 字符]\n\n"
            + result[:MAX_RESULT_SIZE]
            + "\n\n... (截断)"
        )
    return result
```

配置：
```yaml
tools:
  max_result_size_chars: 50000
```

---

## 7.7 流式输出

### 7.7.1 Block Chunker

流式输出分块处理：

```python
# run_agent.py
for chunk in response:
    if chunk.choices[0].delta.content:
        yield chunk.choices[0].delta.content
```

### 7.7.2 重试策略

```python
class RetryStrategy:
    def should_retry(self, error, attempt):
        if attempt >= self.max_attempts:
            return False
        return error.is_retryable()

    def get_delay(self, attempt):
        return min(2 ** attempt * self.base_delay, self.max_delay)
```

### 7.7.3 提前终止

某些条件下提前结束推理：

```python
# 检测到完整响应时提前终止
def should_stop(response) -> bool:
    return (
        response.is_complete() or
        response.has_error() or
        response.exceeds_max_tokens()
    )
```

---

## 7.8 本章小结

### 核心概念总结

| 概念 | 说明 |
|------|------|
| 四层架构 | 入口层→控制层→执行层→能力层 |
| 自注册模式 | 工具模块导入时自动注册到 Registry |
| 事件驱动 | 通过事件总线连接各组件 |
| 分道排队 | Session/Global/Background 三种队列 |
| 动态提示词 | 根据上下文实时组装 System Prompt |

### 源码位置索引

| 功能 | 源码文件 |
|------|----------|
| Agent Loop | `run_agent.py` |
| 工具注册 | `tools/registry.py` |
| 工具调度 | `model_tools.py` |
| 状态管理 | `hermes_state.py` |
| 网关入口 | `gateway/` |

### 调试命令

```bash
# 追踪完整请求流程
hermes logs --trace --request-id <id>

# 查看工具调用详情
hermes logs --filter "dispatch" --verbose

# 模拟 Agent Loop
python run_agent.py --prompt "test"
```

---

*下章预告：第八章 高级特性 - 语音模式、API服务器、MCP集成、RL训练*