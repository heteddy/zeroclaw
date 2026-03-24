# Agent 模块详细设计

## 1. 模块概述

Agent 模块是 ZeroClaw 系统的核心编排引擎，负责 AI 代理的完整生命周期管理，包括消息处理、上下文管理、工具调度、记忆加载和执行控制。

## 2. 架构设计

### 2.1 模块结构

```
src/agent/
├── mod.rs           # 模块导出
├── agent.rs         # Agent 核心结构体和构建器
├── loop_.rs         # 代理主循环（200KB+ 核心逻辑）
├── classifier.rs    # 意图分类器
├── dispatcher.rs    # 请求分发器
├── prompt.rs        # Prompt 工程管理
├── memory_loader.rs # 记忆加载策略
└── tests.rs         # 集成测试
```

### 2.2 核心组件关系

```
用户消息 → Classifier(意图识别) → Dispatcher(路由) → Agent Loop(执行循环)
                                    ↓
                              Memory Loader(上下文加载)
                                    ↓
                              Provider(LLM 调用)
                                    ↓
                              Tool Executor(工具执行)
                                    ↓
                              Response(响应生成)
```

## 3. 核心数据结构

### 3.1 Agent 结构体 (agent.rs)

```rust
pub struct Agent {
    // 配置
    provider: Arc<dyn Provider>,
    memory: Arc<dyn Memory>,
    tools: Vec<Arc<dyn Tool>>,
    
    // 状态管理
    conversation_history: Vec<ConversationMessage>,
    session_id: String,
    
    // 性能参数
    temperature: f64,
    max_iterations: usize,
}
```

**职责**:
- 封装 LLM Provider、Memory、Tools 三大核心依赖
- 维护对话历史和会话状态
- 提供配置参数（temperature、max_iterations）

### 3.2 AgentBuilder

采用 Builder 模式构建 Agent:

```rust
AgentBuilder::new()
    .provider(provider)
    .memory(memory)
    .tools(tools)
    .temperature(0.7)
    .max_iterations(10)
    .build()
```

## 4. Agent Loop 核心流程 (loop_.rs)

### 4.1 主循环伪代码

```rust
pub async fn run(agent: &mut Agent, initial_message: &str) -> Result<String> {
    let mut iteration = 0;
    
    loop {
        // 1. 检查迭代次数限制
        if iteration >= agent.max_iterations {
            bail!("Maximum iterations exceeded");
        }
        
        // 2. 加载相关记忆到上下文
        let relevant_memories = agent.memory
            .recall(initial_message, 5, Some(&agent.session_id))
            .await?;
        
        // 3. 构建 Prompt
        let messages = build_prompt_with_context(
            &agent.conversation_history,
            &relevant_memories,
            &agent.tools
        );
        
        // 4. 调用 LLM Provider
        let response = agent.provider.chat(messages, model, temperature).await?;
        
        // 5. 处理响应
        if response.has_tool_calls() {
            // 5.1 执行工具调用
            for tool_call in response.tool_calls {
                let result = execute_tool(&tool_call, &agent.tools).await?;
                
                // 5.2 将工具结果添加到历史
                agent.conversation_history.push(
                    ConversationMessage::ToolResults(vec![result])
                );
            }
            // 5.3 继续下一轮迭代
            iteration += 1;
            continue;
        }
        
        // 6. 最终响应 - 无需更多工具调用
        if let Some(text) = response.text {
            // 保存到对话历史
            agent.conversation_history.push(
                ConversationMessage::Chat(ChatMessage::assistant(&text))
            );
            
            // 自动保存到记忆
            if agent.config.auto_save_memory {
                agent.memory.store(
                    &generate_key(),
                    &text,
                    MemoryCategory::Conversation,
                    Some(&agent.session_id)
                ).await?;
            }
            
            return Ok(text);
        }
    }
}
```

### 4.2 关键设计决策

#### 4.2.1 迭代限制
- **目的**: 防止无限工具调用循环
- **默认值**: 通常在配置中设定 (如 10-20 次)
- **错误处理**: 超过限制时返回明确错误

#### 4.2.2 记忆加载策略
- **时机**: 每次迭代开始时加载
- **相关性**: 基于当前消息内容进行关键词召回
- **作用域**: 支持 session 级别的隔离

#### 4.2.3 工具执行反馈循环
```
LLM 请求工具调用 → 执行工具 → 结果反馈给 LLM → LLM 决定下一步
```

这个循环会持续直到:
1. LLM 给出最终文本响应
2. 达到最大迭代次数

## 5. 分类器 (classifier.rs)

### 5.1 职责

对输入消息进行意图分类，决定如何处理:

```rust
pub enum Intent {
    Chat,              // 普通聊天
    ToolUse,          // 需要使用工具
    MemoryQuery,      // 记忆查询
    SystemCommand,    // 系统命令
}

pub async fn classify(message: &str) -> Result<Intent> {
    // 实现可能使用轻量级模型或规则匹配
}
```

### 5.2 优化策略

- **快速路径**: 简单消息直接规则匹配
- **慢速路径**: 复杂消息调用小型 LLM 分类

## 6. 分发器 (dispatcher.rs)

### 6.1 路由逻辑

根据分类结果分发到不同处理器:

```rust
pub async fn dispatch(
    intent: Intent,
    agent: &mut Agent,
    message: &str
) -> Result<String> {
    match intent {
        Intent::Chat => agent.chat(message).await,
        Intent::ToolUse => agent.execute_with_tools(message).await,
        Intent::MemoryQuery => agent.query_memory(message).await,
        Intent::SystemCommand => handle_system_command(message).await,
    }
}
```

## 7. Prompt 工程 (prompt.rs)

### 7.1 Prompt 模板结构

```rust
pub fn build_prompt(
    system: Option<&str>,
    history: &[ConversationMessage],
    tools: &[ToolSpec],
    memories: &[MemoryEntry]
) -> Vec<ChatMessage> {
    let mut messages = Vec::new();
    
    // 1. System prompt (最高优先级)
    if let Some(system_prompt) = system {
        messages.push(ChatMessage::system(system_prompt));
    }
    
    // 2. Tool instructions (如果支持工具调用)
    if !tools.is_empty() {
        messages.push(ChatMessage::system(&build_tool_instructions(tools)));
    }
    
    // 3. Relevant memories (上下文增强)
    for memory in memories {
        messages.push(ChatMessage::system(&format!(
            "Relevant memory: {}", memory.content
        )));
    }
    
    // 4. Conversation history
    for msg in history {
        match msg {
            ConversationMessage::Chat(chat_msg) => {
                messages.push(chat_msg.clone());
            }
            ConversationMessage::AssistantToolCalls { text, tool_calls, .. } => {
                if let Some(text) = text {
                    messages.push(ChatMessage::assistant(text));
                }
                // 工具调用也会以特定格式加入
            }
            ConversationMessage::ToolResults(results) => {
                for result in results {
                    messages.push(ChatMessage::tool(&result.content));
                }
            }
        }
    }
    
    messages
}
```

### 7.2 设计原则

1. **优先级顺序**: System > Tools > Memories > History > Current
2. **Token 经济**: 只加载最相关的上下文
3. **格式一致性**: 所有消息统一为 `ChatMessage` 类型

## 8. 记忆加载器 (memory_loader.rs)

### 8.1 加载策略

```rust
pub struct MemoryLoader {
    memory_backend: Arc<dyn Memory>,
    relevance_threshold: f64,
    max_memories: usize,
}

impl MemoryLoader {
    pub async fn load_relevant_memories(
        &self,
        query: &str,
        session_id: Option<&str>,
    ) -> Result<Vec<MemoryEntry>> {
        // 1. 从后端召回
        let mut memories = self.memory_backend
            .recall(query, self.max_memories, session_id)
            .await?;
        
        // 2. 过滤低相关性
        memories.retain(|m| {
            m.score.unwrap_or(0.0) >= self.relevance_threshold
        });
        
        // 3. 按相关性排序
        memories.sort_by(|a, b| {
            b.score.partial_cmp(&a.score).unwrap()
        });
        
        Ok(memories)
    }
}
```

### 8.2 优化技术

- **缓存**: 高频查询结果缓存
- **预取**: 基于历史模式预测性加载
- **分层**: Core > Daily > Conversation 的优先级

## 9. 错误处理策略

### 9.1 错误类型

```rust
pub enum AgentError {
    ProviderError(anyhow::Error),
    MemoryError(anyhow::Error),
    ToolExecutionError { tool: String, error: String },
    MaxIterationsExceeded,
    InvalidToolCall { call_id: String, reason: String },
}
```

### 9.2 恢复机制

1. **工具执行失败**: 将错误反馈给 LLM，让其重试或回退
2. **Provider 失败**: 根据配置决定是否切换到备用 Provider
3. **记忆加载失败**: 降级为无记忆模式继续

## 10. 性能优化

### 10.1 并发策略

```rust
// 并行加载记忆和工具预热
let (memories, _) = tokio::join!(
    memory_loader.load_relevant_memories(&query, session_id),
    provider.warmup()
);
```

### 10.2 Token 优化

- **历史截断**: 保留最近 N 轮 + 关键记忆
- **工具描述压缩**: 精简工具文档
- **流式响应**: 减少首字节延迟

## 11. 测试策略 (tests.rs)

### 11.1 单元测试

```rust
#[tokio::test]
async fn test_agent_loop_basic_flow() {
    let mut agent = create_test_agent().await;
    let result = agent.run("What is the weather?").await.unwrap();
    assert!(!result.is_empty());
}
```

### 11.2 集成测试

- 端到端消息处理流程
- 多轮对话状态保持
- 工具调用链正确性

### 11.3 边界测试

- 超长上下文处理
- 恶意输入鲁棒性
- 资源限制触发

## 12. 扩展点

### 12.1 自定义分类器

实现 `Classifier` trait 替换默认逻辑

### 12.2 自定义记忆策略

调整 `MemoryLoader` 的相关性算法

### 12.3 自定义迭代控制

重写 `run` 函数添加业务特定的循环逻辑

## 13. 与其他模块的接口

### 13.1 依赖注入

```rust
// 通过 trait object 实现松耦合
pub struct Agent {
    provider: Arc<dyn Provider>,  // 来自 providers 模块
    memory: Arc<dyn Memory>,      // 来自 memory 模块
    tools: Vec<Arc<dyn Tool>>,    // 来自 tools 模块
}
```

### 13.2 输出接口

Agent 处理结果通过 `Channel` trait 发送到各通信平台

## 14. 配置项

```toml
[agent]
max_iterations = 10
temperature = 0.7
auto_save_memory = true
memory_relevance_threshold = 0.6
max_conversation_history = 50  # 轮数
```

## 15. 监控与可观测性

### 15.1 关键指标

- 平均迭代次数
- 工具调用成功率
- 记忆加载延迟
- Token 消耗统计

### 15.2 日志记录

```rust
tracing::info!(
    iteration = %iteration,
    tool_calls = %response.tool_calls.len(),
    "Agent loop iteration completed"
);
```

## 16. 安全考虑

### 16.1 输入验证

- 过滤恶意指令注入
- 限制输入长度

### 16.2 工具调用审计

- 记录所有工具调用
- 敏感操作需要审批

### 16.3 记忆隔离

- Session 级别的数据隔离
- 敏感信息加密存储
