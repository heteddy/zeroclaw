# ZeroClaw 架构图

本文档提供 ZeroClaw 架构、执行模式和数据流的可视化表示。

---

## 1. 执行模式

**ZeroClaw 的运行方式：**

```mermaid
flowchart TD
    Start[zeroclaw CLI] --> Onboard[onboard<br/>安装向导]
    Start --> Agent[agent<br/>交互式 CLI]
    Start --> Gateway[gateway<br/>HTTP 服务器]
    Start --> Daemon[daemon<br/>长期运行的运行时]
    Start --> Channel[channel<br/>消息平台]
    Start --> Service[service<br/>操作系统服务管理]
    Start --> Models[models<br/>提供商目录]
    Start --> Cron[cron<br/>定时任务]
    Start --> Hardware[hardware<br/>外设发现]
    Start --> Peripheral[peripheral<br/>硬件管理]
    Start --> Status[status<br/>系统概览]
    Start --> Doctor[doctor<br/>诊断工具]
    Start --> Migrate[migrate<br/>数据导入]
    Start --> Skills[skills<br/>用户能力]
    Start --> Integrations[integrations<br/>浏览 50+ 应用]

    Agent --> AgentSingle[-m message<br/>单次执行]
    Agent --> AgentInteractive[交互式 REPL<br/>stdin/stdout]

    Daemon --> DaemonSupervised[受监管的运行时<br/>网关 + 渠道 + 调度器]
```

---

## 2. 系统架构概览

**高级组件结构：**

```mermaid
flowchart TB
    subgraph CLI[CLI 入口点]
        Main[main.rs]
    end

    subgraph Core[核心子系统]
        Config[config/<br/>配置和架构]
        Agent[agent/<br/>编排循环]
        Providers[providers/<br/>LLM 适配器]
        Channels[channels/<br/>消息平台]
        Tools[tools/<br/>工具执行]
        Memory[memory/<br/>存储后端]
        Security[security/<br/>策略和配对]
        Runtime[runtime/<br/>执行适配器]
        Gateway[gateway/<br/>HTTP/Webhook 服务器]
        Daemon[daemon/<br/>受监管的运行时]
        Peripherals[peripherals/<br/>硬件控制]
        Observability[observability/<br/>遥测和指标]
        RAG[rag/<br/>硬件文档]
        Cron[cron/<br/>调度器]
        Skills[skills/<br/>用户能力]
    end

    subgraph Integrations[集成]
        Composio[Composio<br/>1000+ 应用]
        Browser[Browser<br/>Brave 集成]
        Tunnel[Tunnel<br/>Cloudflare/boringproxy]
    end

    Main --> Config
    Main --> Agent
    Main --> Gateway
    Main --> Daemon
    Main --> Channels

    Agent --> Providers
    Agent --> Tools
    Agent --> Memory
    Agent --> Security
    Agent --> Runtime
    Agent --> Peripherals
    Agent --> RAG
    Agent --> Skills

    Channels --> Agent
    Gateway --> Agent

    Daemon --> Gateway
    Daemon --> Channels
    Daemon --> Cron
    Daemon --> Observability

    Tools --> Composio
    Tools --> Browser
    Gateway --> Tunnel

    classDef coreComp fill:#4A90E2,stroke:#1E3A5F,color:#fff
    classDef integComp fill:#50C878,stroke:#1E3A5F,color:#fff
    classDef cliComp fill:#F5A623,stroke:#1E3A5F,color:#fff

    class Config,Agent,Providers,Channels,Tools,Memory,Security,Runtime,Gateway,Daemon,Peripherals,Observability,RAG,Cron,Skills coreComp
    class Composio,Browser,Tunnel integComp
    class Main cliComp
```

---

## 3. 消息流经系统

**用户消息如何变成响应：**

```mermaid
sequenceDiagram
    participant User as 用户
    participant Channel as 渠道层
    participant Dispatcher as 消息分发器
    participant Agent as Agent 循环
    participant Provider as LLM 提供商
    participant Tools as 工具注册表
    participant Memory as 记忆后端

    User->>Channel: 发送消息
    Channel->>Dispatcher: ChannelMessage{id, sender, content}
    Dispatcher->>Memory: 召回上下文
    Memory-->>Dispatcher: 相关记忆
    Dispatcher->>Agent: process_message()

    Note over Agent: 构建系统提示词<br/>+ 记忆上下文

    Agent->>Provider: chat_with_tools(history)
    Provider-->>Agent: LLM response

    alt 存在工具调用
        loop 对每个工具调用
            Agent->>Tools: execute(args)
            Tools-->>Agent: ToolResult
        end
        Agent->>Provider: chat_with_tools(+ 工具结果)
        Provider-->>Agent: 最终响应
    end

    Agent-->>Dispatcher: 响应文本
    Dispatcher->>Memory: 存储对话
    Dispatcher-->>Channel: SendMessage{content, recipient}
    Channel-->>User: 回复
```

---

## 4. Agent 循环执行流程

**核心 Agent 编排循环：**

```mermaid
flowchart TD
    Start[[开始：用户消息]] --> BuildContext[构建上下文]

    BuildContext --> MemoryRecall[Memory.recall<br/>检索相关条目]
    BuildContext --> HardwareRAG{硬件<br/>已启用？}
    HardwareRAG -->|是| LoadDatasheets[加载硬件 RAG<br/>引脚别名 + 块]
    HardwareRAG -->|否| BuildPrompt[构建系统提示词]
    LoadDatasheets --> BuildPrompt

    MemoryRecall --> Enrich[丰富消息<br/>记忆 + RAG 上下文]
    Enrich --> BuildPrompt

    BuildPrompt --> InitHistory[初始化历史<br/>系统 + 用户消息]

    InitHistory --> ToolLoop{工具调用循环<br/>最多 10 次迭代}

    ToolLoop --> LLMRequest[Provider.chat_with_tools<br/>或 chat_with_history]
    LLMRequest --> ParseResponse[解析响应]

    ParseResponse --> HasTools{存在<br/>工具调用？}

    HasTools -->|否| SaveResponse[推送助手响应]
    SaveResponse --> Return[[返回：最终响应]]

    HasTools -->|是| Approval{需要<br/>审批？}
    Approval -->|是且拒绝| DenyTool[记录拒绝]
    DenyTool --> NextIteration

    Approval -->|否 / 已批准| ExecuteTools[执行工具<br/>并行执行]

    ExecuteTools --> ScrubResults[从输出中擦除凭据]
    ScrubResults --> AddResults[将工具结果<br/>添加到历史]
    AddResults --> NextIteration

    DenyTool --> NextIteration[增加迭代计数]
    NextIteration --> MaxIter{达到<br/>最大 10 次？}
    MaxIter -->|是| Error[[错误：达到最大迭代次数]]
    MaxIter -->|否| ToolLoop

    classDef contextStep fill:#E8F4FD,stroke:#4A90E2
    classDef llmStep fill:#FFF4E6,stroke:#F5A623
    classDef toolStep fill:#E8FDF5,stroke:#50C878
    classDef errorStep fill:#FDE8E8,stroke:#D0021B

    class BuildContext,MemoryRecall,HardwareRAG,LoadDatasheets,Enrich,BuildPrompt,InitHistory contextStep
    class LLMRequest,ParseResponse llmStep
    class ExecuteTools,ScrubResults,AddResults toolStep
    class Error errorStep
```

---

## 5. Daemon 监管模型

**Daemon 如何保持组件运行：**

```mermaid
flowchart TB
    Start[[zeroclaw daemon]] --> SpawnComponents

    SpawnComponents --> SpawnState[生成状态写入器<br/>5 秒刷新间隔]
    SpawnComponents --> SpawnGateway[生成网关监管器]
    SpawnComponents --> SpawnChannels{配置了<br/>渠道？}
    SpawnComponents --> SpawnHeartbeat{启用了<br/>心跳？}
    SpawnComponents --> SpawnScheduler{启用了<br/>Cron？}

    SpawnChannels -->|是| SpawnChannelSup[生成渠道监管器]
    SpawnChannels -->|否| MarkChannelsOK[标记渠道正常<br/>已禁用]

    SpawnHeartbeat -->|是| SpawnHeartbeatWorker[生成心跳工作器]
    SpawnHeartbeat -->|否| MarkHeartbeatOK[标记心跳正常<br/>已禁用]

    SpawnScheduler -->|是| SpawnSchedulerWorker[生成 Cron 调度器]
    SpawnScheduler -->|否| MarkSchedulerOK[标记调度器正常<br/>已禁用]

    SpawnGateway --> GatewayLoop{Gateway Loop}
    SpawnChannelSup --> ChannelLoop{Channel Loop}
    SpawnHeartbeatWorker --> HeartbeatLoop{Heartbeat Loop}
    SpawnSchedulerWorker --> SchedulerLoop{Scheduler Loop}

    GatewayLoop --> GatewayRun[run_gateway]
    GatewayRun --> GatewayExit{正常退出？}
    GatewayExit -->|否| GatewayError[标记错误 + 日志]
    GatewayExit -->|是| GatewayUnexpected[标记：意外退出]
    GatewayError --> GatewayBackoff[等待并退避]
    GatewayUnexpected --> GatewayBackoff
    GatewayBackoff --> GatewayLoop

    ChannelLoop --> ChannelRun[start_channels]
    ChannelRun --> ChannelExit{正常退出？}
    ChannelExit -->|否| ChannelError[标记错误 + 日志]
    ChannelExit -->|是| ChannelUnexpected[标记：意外退出]
    ChannelError --> ChannelBackoff[等待并退避]
    ChannelUnexpected --> ChannelBackoff
    ChannelBackoff --> ChannelLoop

    HeartbeatLoop --> HeartbeatRun[收集任务 + Agent 运行]
    HeartbeatRun --> HeartbeatExit{正常退出？}
    HeartbeatExit -->|否| HeartbeatError[标记错误 + 日志]
    HeartbeatExit -->|是| HeartbeatUnexpected[标记：意外退出]
    HeartbeatError --> HeartbeatBackoff[等待并退避]
    HeartbeatUnexpected --> HeartbeatBackoff
    HeartbeatBackoff --> HeartbeatLoop

    SchedulerLoop --> SchedulerRun[cron::scheduler::run]
    SchedulerRun --> SchedulerExit{正常退出？}
    SchedulerExit -->|否| SchedulerError[标记错误 + 日志]
    SchedulerExit -->|是| SchedulerUnexpected[标记：意外退出]
    SchedulerError --> SchedulerBackoff[等待并退避]
    SchedulerUnexpected --> SchedulerBackoff
    SchedulerBackoff --> SchedulerLoop

    MarkChannelsOK --> Running[Daemon 运行中<br/>按 Ctrl+C 停止]
    MarkHeartbeatOK --> Running
    MarkSchedulerOK --> Running
    SpawnState --> Running

    Running --> StopRequest[收到 Ctrl+C]
    StopRequest --> AbortAll[中止所有任务]
    AbortAll --> JoinAll[等待任务完成]
    JoinAll --> Done[[Daemon 已停止]]

    classDef supervisor fill:#FDE8E8,stroke:#D0021B
    classDef running fill:#E8FDF5,stroke:#50C878
    classDef component fill:#E8F4FD,stroke:#4A90E2

    class SpawnGateway,SpawnChannelSup,SpawnHeartbeatWorker,SpawnSchedulerWorker,SpawnState supervisor
    class Running running
    class GatewayRun,ChannelRun,HeartbeatRun,SchedulerRun component
```

---

## 6. Gateway HTTP 端点

**网关的 HTTP API 结构：**

```mermaid
flowchart TB
    Client[HTTP 客户端] --> Gateway[ZeroClaw 网关]

    Gateway --> PairPOST[POST /pair<br/>交换一次性代码<br/>获取 bearer token]
    Gateway --> HealthGET[GET /health<br/>状态检查]
    Gateway --> WebhookPOST[POST /webhook<br/>主 agent 端点]
    Gateway --> WAVerify[GET /whatsapp<br/>Meta 验证]
    Gateway --> WAMessage[POST /whatsapp<br/>WhatsApp webhook]

    PairPOST --> PairLimiter[速率限制器<br/>配对请求/分钟]
    PairLimiter --> PairGuard[PairingGuard<br/>代码验证]
    PairGuard --> PairResponse["{paired, token, persisted}"]

    WebhookPOST --> WebhookLimiter[速率限制器<br/>webhook 请求/分钟]
    WebhookLimiter --> WebhookPairing{需要<br/>配对？}
    WebhookPairing -->|是| BearerAuth[Bearer token 检查]
    WebhookPairing -->|否| WebhookSecret{配置了<br/>密钥？}
    WebhookSecret -->|是| SecretCheck[X-Webhook-Secret<br/>HMAC-SHA256 验证]
    WebhookSecret -->|否| Idempotency[幂等性检查<br/>X-Idempotency-Key]
    BearerAuth --> Idempotency
    SecretCheck --> Idempotency

    Idempotency --> MemoryStore[自动保存到记忆]
    MemoryStore --> ProviderCall[Provider.simple_chat]
    ProviderCall --> WebhookResponse["{response, model}"]

    WAVerify --> TokenCheck[verify_token 检查<br/>恒定时间比较]
    TokenCheck --> Challenge[返回 hub.challenge]

    WAMessage --> SignatureCheck[X-Hub-Signature-256<br/>HMAC-SHA256 验证]
    SignatureCheck --> ParsePayload[解析消息]
    ParsePayload --> ForEach[对每条消息]
    ForEach --> WAMemory[自动保存到记忆]
    WAMemory --> WAProvider[Provider.simple_chat]
    WAProvider --> WASend[WhatsAppChannel.send]

    classDef auth fill:#FDE8E8,stroke:#D0021B
    classDef processing fill:#E8F4FD,stroke:#4A90E2
    classDef response fill:#E8FDF5,stroke:#50C878

    class PairLimiter,PairGuard,BearerAuth,SecretCheck auth
    class MemoryStore,ProviderCall,TokenCheck,ParsePayload,ForEach,WAMemory,WAProvider processing
    class PairResponse,WebhookResponse,Challenge,WASend response
```

---

## 7. 渠道消息分发

**渠道如何将消息路由到 agent：**

```mermaid
flowchart TB
    subgraph Channels[渠道监听器]
        TG[Telegram]
        DC[Discord]
        SL[Slack]
        IM[iMessage]
        MX[Matrix]
        SIG[Signal]
        WA[WhatsApp]
        Email[Email]
        IRC[IRC]
        Lark[Lark]
        DT[钉钉]
        QQ[QQ]
    end

    Channels --> MPSC[MPSC 通道<br/>100 缓冲区队列]

    MPSC --> Semaphore[信号量<br/>最大并发限制]
    Semaphore --> WorkerPool[工作池<br/>JoinSet]

    WorkerPool --> Process[process_channel_message]

    Process --> LogReceive[日志：💬 来自用户]
    LogReceive --> MemoryRecall[build_memory_context]
    MemoryRecall --> AutoSave[如果启用则自动保存]

    AutoSave --> StartTyping[channel.start_typing]
    StartTyping --> Timeout[300 秒超时保护]

    Timeout --> AgentCall[run_tool_call_loop<br/>静默模式]
    AgentCall --> StopTyping[channel.stop_typing]

    StopTyping --> Success{成功？}
    Success -->|是| LogReply[日志：🤖 回复时间]
    Success -->|否| LogError[日志：❌ LLM 错误]
    Success -->|超时| LogTimeout[日志：❌ 超时]

    LogReply --> SendReply[channel.send 回复]
    LogError --> SendError[channel.send 错误消息]
    LogTimeout --> SendTimeout[channel.send 超时消息]

    SendReply --> Done[消息完成]
    SendError --> Done
    SendTimeout --> Done

    Done --> NextWorker[加入下一个工作器]
    NextWorker --> WorkerPool

    classDef channel fill:#E8F4FD,stroke:#4A90E2
    classDef queue fill:#FFF4E6,stroke:#F5A623
    classDef process fill:#FDE8E8,stroke:#D0021B
    classDef success fill:#E8FDF5,stroke:#50C878

    class TG,DC,SL,IM,MX,SIG,WA,Email,IRC,Lark,DT,QQ channel
    class MPSC,Semaphore,WorkerPool queue
    class Process,LogReceive,MemoryRecall,AutoSave,StartTyping,Timeout,AgentCall,StopTyping process
    class LogReply,SendReply,Done,NextWorker success
```

---

## 8. 记忆系统架构

**存储后端和数据流：**

```mermaid
flowchart TB
    subgraph Frontend[记忆前端]
        AutoSave[自动保存钩子<br/>user_msg, assistant_resp]
        StoreTool[memory_store 工具]
        RecallTool[memory_recall 工具]
        ForgetTool[memory_forget 工具]
        GetTool[memory_get 工具]
        ListTool[memory_list 工具]
        CountTool[memory_count 工具]
    end

    subgraph Backends[记忆后端]
        Sqlite[(sqlite<br/>默认，本地文件)]
        Markdown[(markdown<br/>每日 .md 文件)]
        Lucid[(lucid<br/>云同步)]
        None[(none<br/>仅内存)]
    end

    subgraph Categories[记忆类别]
        Conv[对话<br/>聊天转录]
        Daily[每日<br/>会话摘要]
        Core[核心<br/>长期事实]
    end

    AutoSave --> MemoryTrait[Memory trait]
    StoreTool --> MemoryTrait
    RecallTool --> MemoryTrait
    ForgetTool --> MemoryTrait
    GetTool --> MemoryTrait
    ListTool --> MemoryTrait
    CountTool --> MemoryTrait

    MemoryTrait --> Factory[create_memory 工厂]
    Factory -->|config.memory.backend| BackendSelect{后端？}

    BackendSelect -->|sqlite| Sqlite
    BackendSelect -->|markdown| Markdown
    BackendSelect -->|lucid| Lucid
    BackendSelect -->|none| None

    Sqlite --> Categories
    Markdown --> Categories
    Lucid --> Categories

    Categories --> Storage[(Persistent Storage)]

    RAG[硬件 RAG] -.->|load_chunks| Markdown

    classDef frontend fill:#E8F4FD,stroke:#4A90E2
    classDef backend fill:#FFF4E6,stroke:#F5A623
    classDef category fill:#E8FDF5,stroke:#50C878
    classDef storage fill:#FDE8E8,stroke:#D0021B

    class AutoSave,StoreTool,RecallTool,ForgetTool,GetTool,ListTool,CountTool frontend
    class Sqlite,Markdown,Lucid,None backend
    class Conv,Daily,Core category
    class Storage storage
```

---

## 9. 提供商和模型路由

**LLM 提供商抽象和路由：**

```mermaid
flowchart TB
    subgraph Providers[支持的提供商]
        OR[OpenRouter]
        Anth[Anthropic]
        OAI[OpenAI]
        OpenRouter[openrouter]
        MiniMax[minimax]
        DeepSeek[deepseek]
        Kimi[kimi]
        Custom[自定义 URL]
    end

    subgraph Routing[模型路由]
        Routes[model_routes 配置<br/>模式 -> 提供商]
    end

    subgraph Factory[提供商工厂]
        Resilient[create_resilient_provider<br/>重试 + 超时]
        Routed[create_routed_provider<br/>基于模型的路由]
    end

    subgraph Traits[Provider Trait]
        ChatSystem[chat_with_system<br/>简单聊天]
        ChatHistory[chat_with_history<br/>多轮对话]
        ChatTools[chat_with_tools<br/>原生函数调用]
        Warmup[warmup<br/>连接池预热]
        SupportsNative[supports_native_tools<br/>能力检查]
    end

    Providers --> Factory
    Routes --> Factory

    Factory --> Traits

    ChatSystem --> LLM1[LLM API 调用]
    ChatHistory --> LLM2[LLM API 调用]
    ChatTools --> LLM3[LLM API 调用 + 函数]

    LLM1 --> Response[ChatMessage<br/>text + role]
    LLM2 --> Response
    LLM3 --> ToolResponse[ChatMessage + ToolCalls<br/>id, name, arguments]

    classDef provider fill:#E8F4FD,stroke:#4A90E2
    classDef routing fill:#FFF4E6,stroke:#F5A623
    classDef factory fill:#E8FDF5,stroke:#50C878
    classDef trait fill:#FDE8E8,stroke:#D0021B

    class OR,Anth,OAI,OpenRouter,MiniMax,DeepSeek,Kimi,Custom provider
    class Routes routing
    class Resilient,Routed factory
    class ChatSystem,ChatHistory,ChatTools,Warmup,SupportsNative trait
```

---

## 10. 工具执行架构

**工具注册表、执行和安全：**

```mermaid
flowchart TB
    subgraph ToolCategories[工具类别]
        Core[核心工具<br/>shell, file_read, file_write]
        Memory[记忆工具<br/>store, recall, forget]
        Schedule[调度工具<br/>cron_add, cron_list 等]
        Browser[浏览器<br/>Brave 集成]
        Composio[Composio<br/>1000+ 应用操作]
        Hardware[硬件<br/>gpio_read, gpio_write,<br/>arduino_upload 等]
        Delegate[委托<br/>子 agent 路由]
        Screenshot[screenshot<br/>屏幕捕获]
    end

    subgraph Registry[工具注册表]
        AllTools[all_tools_with_runtime<br/>工厂函数]
        DefaultTools[default_tools<br/>基础集]
        PeripheralTools[create_peripheral_tools<br/>硬件特定]
    end

    subgraph Security[安全策略]
        AllowedCmds[allowed_commands<br/>白名单]
        WorkspaceOnly[workspace_only<br/>路径限制]
        MaxActions[max_actions_per_hour<br/>速率限制]
        MaxCost[max_cost_per_day_cents<br/>成本上限]
        Approval[approval manager<br/>受监督的工具]
    end

    subgraph Execution[工具执行]
        Validate[输入验证<br/>架构检查]
        Approve{需要<br/>审批？}
        Execute[异步执行]
        Scrub[从输出中擦除凭据]
        Result[ToolResult<br/>success, output, error]
    end

    ToolCategories --> Registry
    Registry --> Security
    Security --> Execution

    Validate --> Approve
    Approve -->|是| Prompt[提示 CLI]
    Approve -->|否 / 已批准| Execute
    Approve -->|拒绝| Denied[返回拒绝]

    Prompt --> UserChoice{用户选择？}
    UserChoice -->|是| Execute
    UserChoice -->|否| Denied

    Execute --> Scrub
    Scrub --> Result
    Result --> Return[返回到 agent 循环]

    classDef tools fill:#E8F4FD,stroke:#4A90E2
    classDef registry fill:#FFF4E6,stroke:#F5A623
    classDef security fill:#FDE8E8,stroke:#D0021B
    classDef exec fill:#E8FDF5,stroke:#50C878

    class Core,Memory,Schedule,Browser,Composio,Hardware,Delegate,Screenshot tools
    class AllTools,DefaultTools,PeripheralTools registry
    class AllowedCmds,WorkspaceOnly,MaxActions,MaxCost,Approval security
    class Validate,Approve,Prompt,Execute,Scrub,Result,Return exec
```

---

## 11. 配置加载

**配置如何加载和合并：**

```mermaid
flowchart TB
    Start[Config::load_or_init] --> Exists{配置文件<br/>存在？}

    Exists -->|否| RunWizard[运行 onboard 向导]
    RunWizard --> Save[保存 config.toml]
    Save --> Load[从文件加载]

    Exists -->|是| Load

    Load --> Parse[TOML 解析]
    Parse --> Defaults[应用默认值<br/>Config::default]

    Defaults --> EnvOverrides[apply_env_overrides<br/>ZEROCLAW_* 环境变量]

    EnvOverrides --> Validate[架构验证]

    Validate --> Valid{有效？}
    Valid -->|否| Error[[错误：无效配置]]
    Valid -->|是| Complete[完整配置]

    Complete --> Paths[路径<br/>workspace_dir, config_path]
    Complete --> Providers[default_provider,<br/>api_key, api_url]
    Complete --> Model[default_model,<br/>default_temperature]
    Complete --> Gateway[网关配置<br/>port, host, pairing]
    Complete --> Channels[渠道配置<br/>telegram, discord 等]
    Complete --> Memory[记忆配置<br/>backend, auto_save]
    Complete --> Security[自主性配置<br/>level, allowed_commands]
    Complete --> Reliability[可靠性配置<br/>timeouts, retries]
    Complete --> Observability[可观测性<br/>backend, metrics]
    Complete --> Runtime[运行时配置<br/>kind, exec]
    Complete --> Peripherals[外设<br/>boards, datasheet_dir]
    Complete --> Cron[cron 配置<br/>enabled, db_path]
    Complete --> Composio[composio<br/>enabled, api_key]
    Complete --> Browser[browser<br/>enabled, allowlist]
    Complete --> Tunnel[tunnel<br/>provider, token]

    classDef config fill:#E8F4FD,stroke:#4A90E2
    classDef error fill:#FDE8E8,stroke:#D0021B
    classDef section fill:#FFF4E6,stroke:#F5A623

    class Load,Parse,Defaults,EnvOverrides,Validate,Complete config
    class Error error
    class Paths,Providers,Model,Gateway,Channels,Memory,Security,Reliability,Observability,Runtime,Peripherals,Cron,Composio,Browser,Tunnel section
```

---

## 12. 硬件外设集成

**硬件板支持和控制：**

```mermaid
flowchart TB
    subgraph Boards[支持的板子]
        Nucleo[Nucleo-F401RE<br/>STM32F401RETx]
        Uno[Arduino Uno<br/>ATmega328P]
        UnoQ[Uno Q<br/>ESP32 WiFi 桥接]
        RPi[RPi GPIO<br/>原生 Linux]
        ESP32[ESP32<br/>直接串口]
    end

    subgraph Transport[传输层]
        Serial[串口<br/>/dev/ttyACM0, /dev/ttyUSB0]
        USB[USB probe-rs<br/>ST-Link JTAG]
        Native[原生 GPIO<br/>Linux sysfs]
    end

    subgraph Peripherals[外设系统]
        Create[create_peripheral_tools<br/>工厂函数]
        GPIO[gpio_read/write<br/>数字 I/O]
        Upload[arduino_upload<br/>烧录程序]
        MemMap[hardware_memory_map<br/>地址范围]
        BoardInfo[hardware_board_info<br/>芯片识别]
        MemRead[hardware_memory_read<br/>寄存器转储]
        Capabilities[hardware_capabilities<br/>引脚枚举]
    end

    subgraph RAG[硬件 RAG]
        Datasheets[datasheet_dir<br/>.md 文档]
        Chunks[分块嵌入<br/>语义搜索]
        PinAliases[引脚别名映射<br/>red_led → 13]
    end

    Boards --> Transport
    Transport --> Peripherals

    RAG -.->|上下文注入| Peripherals

    Create --> ToolRegistry[工具注册表]
    GPIO --> ToolRegistry
    Upload --> ToolRegistry
    MemMap --> ToolRegistry
    BoardInfo --> ToolRegistry
    MemRead --> ToolRegistry
    Capabilities --> ToolRegistry

    ToolRegistry --> Agent[Agent 循环集成]

    classDef board fill:#E8F4FD,stroke:#4A90E2
    classDef transport fill:#FFF4E6,stroke:#F5A623
    classDef peripheral fill:#E8FDF5,stroke:#50C878
    classDef rag fill:#FDE8E8,stroke:#D0021B

    class Nucleo,Uno,UnoQ,RPi,ESP32 board
    class Serial,USB,Native transport
    class Create,GPIO,Upload,MemMap,BoardInfo,MemRead,Capabilities,ToolRegistry peripheral
    class Datasheets,Chunks,PinAliases rag
```

---

## 13. 可观测事件

**遥测和可观测性流：**

```mermaid
flowchart TB
    subgraph Observers[Observer 后端]
        Noop[NoopObserver<br/>无操作/测试]
        Console[ConsoleObserver<br/>标准输出日志]
        Metrics[MetricsObserver<br/>Prometheus 格式]
    end

    subgraph Events[可观测事件]
        AgentStart[AgentStart<br/>provider, model]
        LlmRequest[LlmRequest<br/>provider, model, msg_count]
        LlmResponse[LlmResponse<br/>duration, success, error]
        ToolCallStart[ToolCallStart<br/>tool name]
        ToolCall[ToolCall<br/>tool, duration, success]
        TurnComplete[TurnComplete<br/>agent 循环结束]
        AgentEnd[AgentEnd<br/>duration, tokens, cost]
    end

    subgraph Outputs[输出]
        Stdout[stdout 跟踪日志]
        MetricsFile[metrics.json<br/>JSON 行]
        Prometheus[Prometheus<br/>文本格式]
    end

    Events --> Observers
    Observers --> Outputs

    AgentStart --> Record[record_event]
    LlmRequest --> Record
    LlmResponse --> Record
    ToolCallStart --> Record
    ToolCall --> Record
    TurnComplete --> Record
    AgentEnd --> Record

    Record --> Dispatch[分发到后端]
    Dispatch --> Console
    Dispatch --> Metrics

    Console --> Stdout
    Metrics --> MetricsFile

    classDef observer fill:#E8F4FD,stroke:#4A90E2
    classDef event fill:#FFF4E6,stroke:#F5A623
    classDef output fill:#E8FDF5,stroke:#50C878

    class Noop,Console,Metrics observer
    class AgentStart,LlmRequest,LlmResponse,ToolCallStart,ToolCall,TurnComplete,AgentEnd,Record,Dispatch event
    class Stdout,MetricsFile,Prometheus output
```

---

## 总结图

**快速参考概览：**

```mermaid
mindmap
    root((ZeroClaw))
        模式
            Agent CLI
                交互式
                单次执行
            网关
                HTTP API
                Webhooks
            Daemon
                受监管
                多组件
            渠道
                12+ 平台
        组件
            Agent 循环
                工具调用
                记忆感知
            提供商
                50+ LLMs
                模型路由
            渠道
                实时
                受监管
            工具
                30+ 工具
                硬件控制
            记忆
                4 个后端
                RAG 能力
            安全
                配对
                审批
                策略
        集成
            Composio
                1000+ 应用
            浏览器
                Brave
            隧道
                Cloudflare
                boringproxy
        硬件
            STM32
            Arduino
            ESP32
            RPi GPIO
```

---

*为 ZeroClaw v0.1.0 生成 - 架构文档*
