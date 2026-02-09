# CLI 命令系统详解

## 📋 文件概述

`nanobot/cli/commands.py` 是 nanobot 的命令行接口实现，使用 **Typer** 框架构建，提供了完整的 CLI 命令体系。

## 🎯 主要功能

CLI 系统负责：
- 项目初始化和配置
- Agent 交互（单次和交互式）
- Gateway 服务启动
- 渠道管理和登录
- 定时任务管理
- 系统状态查看

## 🏗️ 架构设计

### 命令层次结构

```
nanobot (主应用)
├── onboard           # 初始化命令
├── agent             # Agent 交互命令
├── gateway           # Gateway 服务
├── status            # 状态查看
├── channels          # 渠道管理子命令
│   ├── status       #   查看渠道状态
│   └── login        #   登录 WhatsApp
└── cron              # 定时任务子命令
    ├── list         #   列出任务
    ├── add          #   添加任务
    ├── remove       #   删除任务
    ├── enable       #   启用/禁用任务
    └── run          #   手动运行任务
```

## 📝 核心命令详解

### 1. onboard - 初始化命令

```python
@app.command()
def onboard():
    """Initialize nanobot configuration and workspace."""
```

**功能说明：**
- 创建配置文件 `~/.nanobot/config.json`
- 创建工作空间 `~/.nanobot/workspace/`
- 生成默认模板文件（AGENTS.md, SOUL.md, USER.md）
- 创建记忆目录和 MEMORY.md

**工作流程：**
```
1. 检查配置文件是否存在
   ├─ 存在 → 询问是否覆盖
   └─ 不存在 → 继续
2. 创建默认配置对象
3. 保存配置文件
4. 创建工作空间目录
5. 创建模板文件
   ├─ AGENTS.md   - Agent 指令
   ├─ SOUL.md     - 个性定义
   ├─ USER.md     - 用户信息
   └─ memory/MEMORY.md - 长期记忆
6. 显示下一步操作提示
```

**模板文件内容：**

```markdown
# AGENTS.md
你是一个有用的 AI 助手。要简洁、准确、友好。

## 指导原则
- 在执行操作前解释你将要做什么
- 请求不明确时询问澄清
- 使用工具完成任务
- 在记忆文件中记录重要信息
```

### 2. gateway - Gateway 服务

```python
@app.command()
def gateway(
    port: int = typer.Option(18790, "--port", "-p", help="Gateway port"),
    verbose: bool = typer.Option(False, "--verbose", "-v", help="Verbose output"),
):
```

**功能说明：**
启动 nanobot 网关服务，连接所有组件和渠道。

**核心组件初始化：**

```python
# 1. 消息总线
bus = MessageBus()

# 2. LLM 提供商
provider = LiteLLMProvider(
    api_key=p.api_key,
    api_base=config.get_api_base(),
    default_model=config.agents.defaults.model,
)

# 3. Cron 服务
cron = CronService(cron_store_path)

# 4. Agent 循环
agent = AgentLoop(
    bus=bus,
    provider=provider,
    workspace=config.workspace_path,
    model=config.agents.defaults.model,
    max_iterations=config.agents.defaults.max_tool_iterations,
    brave_api_key=config.tools.web.search.api_key,
    exec_config=config.tools.exec,
    cron_service=cron,
    restrict_to_workspace=config.tools.restrict_to_workspace,
)

# 5. 心跳服务
heartbeat = HeartbeatService(
    workspace=config.workspace_path,
    on_heartbeat=on_heartbeat,
    interval_s=30 * 60,  # 30 分钟
    enabled=True
)

# 6. 渠道管理器
channels = ChannelManager(config, bus)
```

**服务启动流程：**
```
1. 加载配置
2. 初始化所有组件
3. 设置 Cron 回调函数
4. 设置心跳回调函数
5. 启动服务（并发执行）
   ├─ CronService.start()
   ├─ HeartbeatService.start()
   ├─ AgentLoop.run()
   └─ ChannelManager.start_all()
```

**Cron 回调函数：**
```python
async def on_cron_job(job: CronJob) -> str | None:
    """通过 Agent 执行定时任务"""
    response = await agent.process_direct(
        job.payload.message,
        session_key=f"cron:{job.id}",
        channel=job.payload.channel or "cli",
        chat_id=job.payload.to or "direct",
    )
    
    # 如果需要发送到渠道
    if job.payload.deliver and job.payload.to:
        await bus.publish_outbound(OutboundMessage(
            channel=job.payload.channel or "cli",
            chat_id=job.payload.to,
            content=response or ""
        ))
    return response
```

### 3. agent - Agent 交互命令

```python
@app.command()
def agent(
    message: str = typer.Option(None, "--message", "-m", help="Message to send"),
    session_id: str = typer.Option("cli:default", "--session", "-s", help="Session ID"),
):
```

**两种模式：**

#### a) 单次消息模式
```bash
nanobot agent -m "What is 2+2?"
```

```python
async def run_once():
    response = await agent_loop.process_direct(message, session_id)
    console.print(f"\n{__logo__} {response}")
asyncio.run(run_once())
```

#### b) 交互式模式
```bash
nanobot agent
```

```python
async def run_interactive():
    while True:
        try:
            user_input = console.input("[bold blue]You:[/bold blue] ")
            if not user_input.strip():
                continue
            
            response = await agent_loop.process_direct(user_input, session_id)
            console.print(f"\n{__logo__} {response}\n")
        except KeyboardInterrupt:
            console.print("\nGoodbye!")
            break

asyncio.run(run_interactive())
```

### 4. channels - 渠道管理命令

#### channels status - 查看渠道状态

```python
@channels_app.command("status")
def channels_status():
    """Show channel status."""
```

**输出格式：**
```
┏━━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Channel ┃ Enabled ┃ Configuration        ┃
┡━━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━┩
│ WhatsApp │   ✓    │ ws://localhost:3001  │
│ Discord  │   ✗    │                       │
│ Telegram │   ✓    │ token: 123456...      │
└─────────┴────────┴───────────────────────┘
```

#### channels login - 登录 WhatsApp

```python
@channels_app.command("login")
def channels_login():
    """Link device via QR code."""
```

**工作流程：**
```
1. 获取桥接目录
   ├─ 检查是否已构建
   ├─ 检查 npm 是否可用
   └─ 复制源代码到用户目录
2. 安装依赖
   └─ npm install
3. 构建项目
   └─ npm run build
4. 启动桥接服务
   └─ npm start
   └─ 显示 QR 码供扫描
```

**桥接目录结构：**
```
~/.nanobot/bridge/
├── node_modules/      # 依赖
├── dist/              # 编译输出
│   └── index.js      #   入口文件
├── src/
│   ├── index.ts
│   ├── server.ts
│   ├── whatsapp.ts
│   └── types.d.ts
├── package.json
└── tsconfig.json
```

### 5. cron - 定时任务命令

#### cron list - 列出任务

```python
@cron_app.command("list")
def cron_list(
    all: bool = typer.Option(False, "--all", "-a", help="Include disabled jobs"),
):
```

**输出格式：**
```
┏━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━━━━━┓
┃ ID    ┃ Name    ┃ Schedule     ┃ Status ┃ Next Run          ┃
┡━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━━━━━┩
│ 001   │ daily   │ every 86400s  │ enabled │ 2026-02-09 09:00 │
│ 002   │ hourly  │ every 3600s   │ enabled │ 2026-02-08 10:00 │
└───────┴─────────┴──────────────┴────────┴───────────────────┘
```

#### cron add - 添加任务

```python
@cron_app.command("add")
def cron_add(
    name: str = typer.Option(..., "--name", "-n", help="Job name"),
    message: str = typer.Option(..., "--message", "-m", help="Message for agent"),
    every: int = typer.Option(None, "--every", "-e", help="Run every N seconds"),
    cron_expr: str = typer.Option(None, "--cron", "-c", help="Cron expression"),
    at: str = typer.Option(None, "--at", help="Run once at time (ISO format)"),
    deliver: bool = typer.Option(False, "--deliver", "-d", help="Deliver response"),
    to: str = typer.Option(None, "--to", help="Recipient"),
    channel: str = typer.Option(None, "--channel", help="Channel"),
):
```

**三种调度类型：**

1. **间隔调度**（`--every`）
```bash
nanobot cron add --name "hourly" --message "Check status" --every 3600
```

2. **Cron 表达式**（`--cron`）
```bash
nanobot cron add --name "daily" --message "Good morning!" --cron "0 9 * * *"
```

3. **一次性任务**（`--at`）
```bash
nanobot cron add --name "once" --message "Task" --at "2026-02-09T10:00:00"
```

**创建 CronSchedule 对象：**
```python
if every:
    schedule = CronSchedule(kind="every", every_ms=every * 1000)
elif cron_expr:
    schedule = CronSchedule(kind="cron", expr=cron_expr)
elif at:
    dt = datetime.datetime.fromisoformat(at)
    schedule = CronSchedule(kind="at", at_ms=int(dt.timestamp() * 1000))
```

#### cron remove - 删除任务

```python
@cron_app.command("remove")
def cron_remove(
    job_id: str = typer.Argument(..., help="Job ID to remove"),
):
```

#### cron enable - 启用/禁用任务

```python
@cron_app.command("enable")
def cron_enable(
    job_id: str = typer.Argument(..., help="Job ID"),
    disable: bool = typer.Option(False, "--disable", help="Disable instead of enable"),
):
```

#### cron run - 手动运行任务

```python
@cron_app.command("run")
def cron_run(
    job_id: str = typer.Argument(..., help="Job ID to run"),
    force: bool = typer.Option(False, "--force", "-f", help="Run even if disabled"),
):
```

### 6. status - 系统状态

```python
@app.command()
def status():
    """Show nanobot status."""
```

**检查项目：**
- 配置文件是否存在
- 工作空间是否存在
- 当前配置的模型
- 各提供商 API Key 是否设置
- vLLM 是否配置

**输出示例：**
```
🐈 nanobot Status

Config: /home/user/.nanobot/config.json ✓
Workspace: /home/user/.nanobot/workspace ✓
Model: anthropropic/claude-opus-4-5

OpenRouter API: ✓
Anthropic API: not set
OpenAI API: not set
Gemini API: not set
Zhipu AI API: not set
AiHubMix API: not set
vLLM/Local: not set
```

## 🔧 辅助函数

### _make_provider - 创建 LLM 提供商

```python
def _make_provider(config):
    """Create LiteLLMProvider from config."""
    from nanobot.providers.litellm_provider import LiteLLMProvider
    
    p = config.get_provider()
    model = config.agents.defaults.model
    
    if not (p and p.api_key) and not model.startswith("bedrock/"):
        console.print("[red]Error: No API key configured.[/red]")
        raise typer.Exit(1)
    
    return LiteLLMProvider(
        api_key=p.api_key if p else None,
        api_base=config.get_api_base(),
        default_model=model,
        extra_headers=p.extra_headers if p else None,
    )
```

### _create_workspace_templates - 创建工作空间模板

```python
def _create_workspace_templates(workspace: Path):
    """Create default workspace template files."""
    templates = {
        "AGENTS.md": "...",
        "SOUL.md": "...",
        "USER.md": "...",
    }
    
    for filename, content in templates.items():
        file_path = workspace / filename
        if not file_path.exists():
            file_path.write_text(content)
    
    # 创建记忆目录
    memory_dir = workspace / "memory"
    memory_dir.mkdir(exist_ok=True)
    memory_file = memory_dir / "MEMORY.md"
    memory_file.write_text("...")
```

### _get_bridge_dir - 获取桥接目录

```python
def _get_bridge_dir() -> Path:
    """Get the bridge directory, setting it up if needed."""
    user_bridge = Path.home() / ".nanobot" / "bridge"
    
    # 检查是否已构建
    if (user_bridge / "dist" / "index.js").exists():
        return user_bridge
    
    # 检查 npm
    if not shutil.which("npm"):
        console.print("[red]npm not found. Please install Node.js >= 18.[/red]")
        raise typer.Exit(1)
    
    # 查找源代码
    pkg_bridge = Path(__file__).parent.parent / "bridge"
    src_bridge = Path(__file__).parent.parent.parent / "bridge"
    
    source = None
    if (pkg_bridge / "package.json").exists():
        source = pkg_bridge
    elif (src_bridge / "package.json").exists():
        source = src_bridge
    
    if not source:
        console.print("[red]Bridge source not found.[/red]")
        raise typer.Exit(1)
    
    # 复制并构建
    user_bridge.parent.mkdir(parents=True, exist_ok=True)
    if user_bridge.exists():
        shutil.rmtree(user_bridge)
    shutil.copytree(source, user_bridge, ignore=shutil.ignore_patterns("node_modules", "dist"))
    
    # 安装和构建
    subprocess.run(["npm", "install"], cwd=user_bridge, check=True, capture_output=True)
    subprocess.run(["npm", "run", "build"], cwd=user_bridge, check=True, capture_output=True)
    
    return user_bridge
```

## 🎨 UI/UX 特性

### Rich Console

使用 **Rich** 库提供美观的终端输出：
- 彩色输出（`[green]`, `[red]`, `[yellow]`, `[cyan]`）
- 表格格式化
- 加粗文本
- 确认提示

### 进度提示

```python
console.print(f"[green]✓[/green] Created config at {config_path}")
console.print(f"[green]✓[/green] Created workspace at {workspace}")
console.print(f"[green]✓[/green] Channels enabled: {', '.join(channels.enabled_channels)}")
console.print(f"[green]✓[/green] Cron: {cron_status['jobs']} scheduled jobs")
console.print(f"[green]✓[/green] Heartbeat: every 30m")
```

## 🔐 错误处理

### API Key 检查

```python
if not (p and p.api_key) and not model.startswith("bedrock/"):
    console.print("[red]Error: No API key configured.[/red]")
    console.print("Set one in ~/.nanobot/config.json under providers section")
    raise typer.Exit(1)
```

### 配置覆盖确认

```python
if config_path.exists():
    console.print(f"[yellow]Config already exists at {config_path}[/yellow]")
    if not typer.confirm("Overwrite?"):
        raise typer.Exit()
```

### 桥接服务构建失败

```python
try:
    subprocess.run(["npm", "install"], cwd=user_bridge, check=True, capture_output=True)
    subprocess.run(["npm", "run", "build"], cwd=user_bridge, check=True, capture_output=True)
except subprocess.CalledProcessError as e:
    console.print(f"[red]Build failed: {e}[/red]")
    if e.stderr:
        console.print(f"[dim]{e.stderr.decode()[:500]}[/dim]")
    raise typer.Exit(1)
```

## 🌟 使用示例

### 完整初始化流程

```bash
# 1. 安装
pip install nanobot-ai

# 2. 初始化
nanobot onboard

# 3. 编辑配置
vim ~/.nanobot/config.json

# 4. 测试
nanobot agent -m "Hello!"

# 5. 启动网关
nanobot gateway
```

### 设置定时任务

```bash
# 每日早上 9 点发送提醒
nanobot cron add \
  --name "morning" \
  --message "Good morning! Check your schedule." \
  --cron "0 9 * * *" \
  --deliver \
  --channel "telegram" \
  --to "123456789"

# 每小时检查状态
nanobot cron add \
  --name "status" \
  --message "Check system status" \
  --every 3600

# 一次性任务
nanobot cron add \
  --name "reminder" \
  --message "Meeting at 3pm" \
  --at "2026-02-09T15:00:00"
```

## 📚 相关文档

- [01-项目概述.md](./01-项目概述.md) - 项目整体介绍
- [03-配置系统.md](./03-配置系统.md) - 配置文件详解
- [04-Agent核心.md](./04-Agent核心.md) - Agent 核心逻辑