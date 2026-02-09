# Agent 核心详解

## 📋 文件概述

`nanobot/agent/loop.py` 是 nanobot 的核心处理引擎，实现了完整的 Agent 循环，协调 LLM 调用和工具执行。

## 🎯 核心职责

AgentLoop 负责以下核心功能：
1. 接收来自消息总线的消息
2. 构建上下文（历史记录、记忆、技能）
3. 调用 LLM 生成响应
4. 执行工具调用
5. 将响应发送回用户

## 🏗️ 类结构

```python
class AgentLoop:
    def __init__(
        self,
        bus: MessageBus,              # 消息总线
        provider: LLMProvider,        # LLM 提供商
        workspace: Path,               # 工作空间路径
        model: str | None = None,     # 模型名称
        max_iterations: int = 20,      # 最大迭代次数
        brave_api_key: str | None = None,  # Brave Search API Key
        exec_config: ExecToolConfig | None = None,  # Shell 执行配置
        cron_service: CronService | None = None,  # Cron 服务
        restrict_to_workspace: bool = False,  # 是否限制到工作空间
    )
```

## 📝 初始化流程

### 1. 组件初始化

```python
def __init__(self, ...):
    # 1. 保存配置
    self.bus = bus
    self.provider = provider
    self.workspace = workspace
    self.model = model or provider.get_default_model()
    self.max_iterations = max_iterations
    self.brave_api_key = brave_api_key
    self.exec_config = exec_config or ExecToolConfig()
    self.cron_service = cron_service
    self.restrict_to_workspace = restrict_to_workspace
    
    # 2. 初始化上下文构建器
    self.context = ContextBuilder(workspace)
    
    # 3. 初始化会话管理器
    self.sessions = SessionManager(workspace)
    
    # 4. 初始化工具注册表
    self.tools = ToolRegistry()
    
    # 5. 初始化子代理管理器
    self.subagents = SubagentManager(
        provider=provider,
        workspace=workspace,
        bus=bus,
        model=self.model,
        brave_api_key=brave_api_key,
        exec_config=self.exec_config,
        restrict_to_workspace=restrict_to_workspace,
    )
    
    # 6. 注册默认工具
    self._register_default_tools()
```

### 2. 默认工具注册

```python
def _register_default_tools(self) -> None:
    """Register the default set of tools."""
    
    # 文件工具（如果配置了限制，则限制到工作空间）
    allowed_dir = self.workspace if self.restrict_to_workspace else None
    self.tools.register(ReadFileTool(allowed_dir=allowed_dir))
    self.tools.register(WriteFileTool(allowed_dir=allowed_dir))
    self.tools.register(EditFileTool(allowed_dir=allowed_dir))
    self.tools.register(ListDirTool(allowed_dir=allowed_dir))
    
    # Shell 工具
    self.tools.register(ExecTool(
        working_dir=str(self.workspace),
        timeout=self.exec_config.timeout,
        restrict_to_workspace=self.restrict_to_workspace,
    ))
    
    # Web 工具
    self.tools.register(WebSearchTool(api_key=self.brave_api_key))
    self.tools.register(WebFetchTool())
    
    # 消息工具
    message_tool = MessageTool(send_callback=self.bus.publish_outbound)
    self.tools.register(message_tool)
    
    # 子代理工具
    spawn_tool = SpawnTool(manager=self.subagents)
    self.tools.register(spawn_tool)
    
    # Cron 工具
    if self.cron_service:
        self.tools.register(CronTool(self.cron_service))
```

**注册的工具：**
1. 📁 **ReadFileTool**: 读取文件内容
2. ✍️ **WriteFileTool**: 写入文件
3. ✏️ **EditFileTool**: 编辑文件（替换文本）
4. 📂 **ListDirTool**: 列出目录内容
5. 💻 **ExecTool**: 执行 Shell 命令
6. 🔍 **WebSearchTool**: Web 搜索
7. 🌐 **WebFetchTool**: 获取网页内容
8. 💬 **MessageTool**: 发送消息到渠道
9. 🤖 **SpawnTool**: 生成子代理
10. ⏰ **CronTool**: 管理定时任务

## 🔄 主循环

### run() - 启动 Agent 循环

```python
async def run(self) -> None:
    """Run the agent loop, processing messages from the bus."""
    self._running = True
    logger.info("Agent loop started")
    
    while self._running:
        try:
            # 1. 等待下一条消息（1秒超时）
            msg = await asyncio.wait_for(
                self.bus.consume_inbound(),
                timeout=1.0
            )
            
            # 2. 处理消息
            try:
                response = await self._process_message(msg)
                if response:
                    await self.bus.publish_outbound(response)
            except Exception as e:
                logger.error(f"Error processing message: {e}")
                # 发送错误响应
                await self.bus.publish_outbound(OutboundMessage(
                    channel=msg.channel,
                    chat_id=msg.chat_id,
                    content=f"Sorry, I encountered an error: {str(e)}"
                ))
        except asyncio.TimeoutError:
            continue
```

**工作流程：**
```
1. 从消息总线等待消息（1秒超时）
2. 如果收到消息
   ├─ 处理消息
   ├─ 如果成功，发送响应
   └─ 如果失败，发送错误信息
3. 如果超时，继续循环
```

### stop() - 停止 Agent 循环

```python
def stop(self) -> None:
    """Stop the agent loop."""
    self._running = False
    logger.info("Agent loop stopping")
```

## 📨 消息处理

### _process_message() - 处理用户消息

```python
async def _process_message(self, msg: InboundMessage) -> OutboundMessage | None:
    """
    Process a single inbound message.
    
    Args:
        msg: The inbound message to process.
    
    Returns:
        The response message, or None if no response needed.
    """
```

**处理流程：**

#### 1. 处理系统消息

```python
# 处理系统消息（子代理公告）
if msg.channel == "system":
    return await self._process_system_message(msg)
```

#### 2. 记录日志

```python
preview = msg.content[:80] + "..." if len(msg.content) > 80 else msg.content
logger.info(f"Processing message from {msg.channel}:{msg.sender_id}: {preview}")
```

#### 3. 获取或创建会话

```python
session = self.sessions.get_or_create(msg.session_key)
```

#### 4. 更新工具上下文

```python
# 更新消息工具上下文
message_tool = self.tools.get("message")
if isinstance(message_tool, MessageTool):
    message_tool.set_context(msg.channel, msg.chat_id)

# 更新生成工具上下文
spawn_tool = self.tools.get("spawn")
if isinstance(spawn_tool, SpawnTool):
    spawn_tool.set_context(msg.channel, msg.chat_id)

# 更新 Cron 工具上下文
cron_tool = self.tools.get("cron")
if isinstance(cron_tool, CronTool):
    cron_tool.set_context(msg.channel, msg.chat_id)
```

#### 5. 构建消息列表

```python
messages = self.context.build_messages(
    history=session.get_history(),      # 历史记录
    current_message=msg.content,        # 当前消息
    media=msg.media if msg.media else None,  # 附件
    channel=msg.channel,               # 渠道
    chat_id=msg.chat_id,              # 聊天 ID
)
```

#### 6. Agent 循环（工具调用迭代）

```python
iteration = 0
final_content = None

while iteration < self.max_iterations:
    iteration += 1
    
    # 调用 LLM
    response = await self.provider.chat(
        messages=messages,
        tools=self.tools.get_definitions(),
        model=self.model
    )
    
    # 处理工具调用
    if response.has_tool_calls:
        # 添加助手消息（包含工具调用）
        tool_call_dicts = [
            {
                "id": tc.id,
                "type": "function",
                "function": {
                    "name": tc.name,
                    "arguments": json.dumps(tc.arguments)
                }
            }
            for tc in response.tool_calls
        ]
        messages = self.context.add_assistant_message(
            messages, response.content, tool_call_dicts
        )
        
        # 执行工具
        for tool_call in response.tool_calls:
            args_str = json.dumps(tool_call.arguments, ensure_ascii=False)
            logger.info(f"Tool call: {tool_call.name}({args_str[:200]})")
            result = await self.tools.execute(tool_call.name, tool_call.arguments)
            messages = self.context.add_tool_result(
                messages, tool_call.id, tool_call.name, result
            )
    else:
        # 没有工具调用，完成
        final_content = response.content
        break
```

**迭代过程：**
```
1. 调用 LLM（传入消息历史和工具定义）
2. 如果 LLM 返回工具调用
   ├─ 将工具调用添加到消息历史
   ├─ 逐个执行工具
   ├─ 将工具结果添加到消息历史
   └─ 继续迭代（再次调用 LLM）
3. 如果 LLM 返回普通文本
   ├─ 保存为最终响应
   └─ 退出循环
4. 如果超过最大迭代次数
   ├─ 使用最后的内容作为响应
   └─ 退出循环
```

#### 7. 记录日志

```python
preview = final_content[:120] + "..." if len(final_content) > 120 else final_content
logger.info(f"Response to {msg.channel}:{msg.sender_id}: {preview}")
```

#### 8. 保存会话

```python
session.add_message("user", msg.content)
session.add_message("assistant", final_content)
self.sessions.save(session)
```

#### 9. 返回响应

```python
return OutboundMessage(
    channel=msg.channel,
    chat_id=msg.chat_id,
    content=final_content
)
```

### _process_system_message() - 处理系统消息

```python
async def _process_system_message(self, msg: InboundMessage) -> OutboundMessage | None:
    """
    Process a system message (e.g., subagent announce).
    
    The chat_id field contains "original_channel:original_chat_id" to route
    the response back to the correct destination.
    """
```

**特点：**
- 处理子代理发送的系统消息
- 从 `chat_id` 解析原始渠道和聊天 ID（格式：`channel:chat_id`）
- 使用原始会话的上下文
- 响应发送回原始渠道

**工作流程：**
```
1. 从 chat_id 解析原始渠道和聊天 ID
   └─ 格式: "channel:chat_id" → (channel, chat_id)
2. 获取原始会话
3. 更新工具上下文
4. 构建消息（包含公告内容）
5. 执行 Agent 循环
6. 保存到会话
7. 发送回原始渠道
```

## 💬 直接处理模式

### process_direct() - 直接处理消息

```python
async def process_direct(
    self,
    content: str,
    session_key: str = "cli:direct",
    channel: str = "cli",
    chat_id: str = "direct",
) -> str:
    """
    Process a message directly (for CLI or cron usage).
    
    Args:
        content: The message content.
        session_key: Session identifier.
        channel: Source channel (for context).
        chat_id: Source chat ID (for context).
    
    Returns:
        The agent's response.
    """
    msg = InboundMessage(
        channel=channel,
        sender_id="user",
        chat_id=chat_id,
        content=content
    )
    
    response = await self._process_message(msg)
    return response.content if response else ""
```

**使用场景：**
- CLI 交互模式
- Cron 任务执行
- 心跳服务调用
- 测试和调试

## 🎨 工具调用示例

### 示例 1: 简单问答

```python
# 用户消息: "What is 2+2?"

# 第一次迭代
messages = [
    {"role": "system", "content": "You are nanobot..."},
    {"role": "user", "content": "What is 2+2?"}
]

response = await provider.chat(messages=messages, tools=tool_defs)
# response.content = "2+2 equals 4."
# response.has_tool_calls = False

# 直接返回响应
final_content = "2+2 equals 4."
```

### 示例 2: 文件操作

```python
# 用户消息: "Read the file README.md and tell me what it says"

# 第一次迭代
messages = [
    {"role": "system", "content": "..."},
    {"role": "user", "content": "Read the file README.md..."}
]

response = await provider.chat(messages=messages, tools=tool_defs)
# response.content = None
# response.tool_calls = [
#     ToolCallRequest(
#         id="call_123",
#         name="read_file",
#         arguments={"path": "README.md"}
#     )
# ]

# 添加助手消息
messages.append({
    "role": "assistant",
    "content": None,
    "tool_calls": [...]
})

# 执行工具
result = await tools.execute("read_file", {"path": "README.md"})
# result = "# nanobot\n\nThis is a project..."

# 添加工具结果
messages.append({
    "role": "tool",
    "tool_call_id": "call_123",
    "name": "read_file",
    "content": "# nanobot\n\nThis is a project..."
})

# 第二次迭代
response = await provider.chat(messages=messages, tools=tool_defs)
# response.content = "The README.md file describes nanobot as..."
# response.has_tool_calls = False

# 返回响应
final_content = "The README.md file describes nanobot as..."
```

### 示例 3: 多步工具调用

```python
# 用户消息: "Create a Python script that prints 'Hello World'"

# 第一次迭代
LLM 调用 write_file 工具
→ 写入文件 hello.py

# 第二次迭代
LLM 调用 exec 工具
→ 执行 python hello.py

# 第三次迭代
LLM 返回文本响应
→ "I've created the script and it works!"
```

## 🔐 安全特性

### 1. 工作空间限制

```python
allowed_dir = self.workspace if self.restrict_to_workspace else None
self.tools.register(ReadFileTool(allowed_dir=allowed_dir))
```

**效果：**
- 文件工具只能访问工作空间内的文件
- 防止路径遍历攻击
- 防止访问敏感系统文件

### 2. Shell 命令限制

```python
self.tools.register(ExecTool(
    working_dir=str(self.workspace),
    timeout=self.exec_config.timeout,
    restrict_to_workspace=self.restrict_to_workspace,
))
```

**效果：**
- Shell 命令在工作空间内执行
- 超时保护（防止死循环）
- 可选的命令白名单/黑名单

### 3. 用户白名单

在渠道配置中设置：
```json
{
  "channels": {
    "telegram": {
      "allowFrom": ["user_id1", "user_id2"]
    }
  }
}
```

**效果：**
- 只允许指定用户与 AI 交互
- 防止未授权访问