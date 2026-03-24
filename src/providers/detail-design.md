# Providers 模块详细设计

## 1. 模块概述

Providers 模块实现与各大 AI 大模型提供商的集成，包括 OpenAI、Anthropic、Google Gemini、智谱 GLM、MiniMax、月之暗面 (Moonshot/Qwen) 等。提供统一的 Trait 抽象和 resilient wrapper，支持多 Provider 路由、重试、降级和成本优化。

## 2. 架构设计

### 2.1 模块结构

src/providers/
- mod.rs          - 工厂注册和配置 (99.9KB)
- traits.rs       - Provider Trait 定义 (29.5KB - 核心)
- reliable.rs     - Resilient Wrapper (70.4KB - 关键)
- router.rs       - Provider 路由器 (14.7KB)
- openai.rs       - OpenAI 集成 (27.7KB)
- openai_codex.rs - OpenAI Codex 专用 (32.0KB)
- anthropic.rs    - Anthropic 集成 (44.5KB)
- gemini.rs       - Google Gemini (75.3KB)
- glm.rs          - 智谱 GLM (10.5KB)
- compatible.rs   - OpenAI 兼容接口 (98.4KB)
- ollama.rs       - Ollama 本地部署 (37.3KB)
- openrouter.rs   - OpenRouter 聚合 (34.1KB)
- copilot.rs      - GitHub Copilot (24.2KB)
- bedrock.rs      - AWS Bedrock (58.6KB)
- telnyx.rs       - Telnyx AI (11.4KB)

### 2.2 设计模式

- **策略模式**: 不同 Provider 实现统一接口
- **装饰器模式**: ReliableProvider 包装基础 Provider 增加韧性
- **工厂模式**: 根据配置创建 Provider 实例
- **适配器模式**: CompatibleProvider 适配 OpenAI 兼容 API
- **责任链**: Router 按优先级尝试多个 Provider

### 2.3 Provider 层次

```
Provider (Trait)
├── OpenAIProvider
├── AnthropicProvider
├── GeminiProvider
├── GLMProvider
├── OllamaProvider
├── OpenRouterProvider
├── CopilotProvider
├── BedrockProvider
├── TelnyxProvider
├── CompatibleProvider (OpenAI 兼容)
└── ReliableProvider (Decorator)
    └── RouterProvider (多 Provider 路由)
```

## 3. 核心 Trait 详解 (traits.rs)

### 3.1 数据结构

#### 3.1.1 ChatMessage

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ChatMessage {
    pub role: String,           // "system", "user", "assistant"
    pub content: String,        // 消息内容
    
    // 可选字段
    pub name: Option<String>,   // 消息名称 (用于区分用户)
    pub tool_calls: Option<Vec<ToolCall>>,  // 工具调用
    pub tool_call_id: Option<String>,       // 工具调用 ID (响应时)
    
    // 多模态支持
    pub images: Option<Vec<ImageAttachment>>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ImageAttachment {
    pub url: String,            // 图片 URL 或 base64
    pub mime_type: String,      // MIME 类型
    pub detail: Option<String>, // "low", "high", "auto"
}
```

#### 3.1.2 ToolCall

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolCall {
    pub id: String,             // 工具调用 ID
    pub name: String,           // 工具名称
    pub arguments: serde_json::Value,  // 参数 JSON
    
    // 执行结果 (响应时)
    #[serde(skip_serializing_if = "Option::is_none")]
    pub result: Option<String>,
}
```

#### 3.1.3 ChatResponse

```rust
#[derive(Debug, Clone, Default)]
pub struct ChatResponse {
    /// 文本回复
    pub text: Option<String>,
    
    /// 工具调用列表
    pub tool_calls: Vec<ToolCall>,
    
    /// Token 使用统计
    pub usage: Option<TokenUsage>,
    
    /// 推理过程内容 (如 DeepSeek 的 reasoning_content)
    pub reasoning_content: Option<String>,
    
    /// Finish reason
    pub finish_reason: Option<String>,
    
    /// 原始响应 (用于调试)
    pub raw_response: Option<serde_json::Value>,
}

#[derive(Debug, Clone, Default)]
pub struct TokenUsage {
    pub input_tokens: u64,
    pub output_tokens: u64,
    pub total_tokens: u64,
    
    // 缓存命中统计 (如 Anthropic)
    pub cache_read_input_tokens: Option<u64>,
    pub cache_creation_input_tokens: Option<u64>,
}
```

#### 3.1.4 ProviderCapabilities

```rust
#[derive(Debug, Clone)]
pub struct ProviderCapabilities {
    /// Provider 名称
    pub name: String,
    
    /// 是否支持工具调用
    pub supports_tools: bool,
    
    /// 是否支持视觉输入
    pub supports_vision: bool,
    
    /// 是否支持流式响应
    pub supports_streaming: bool,
    
    /// 是否支持系统 Prompt
    pub supports_system_prompt: bool,
    
    /// 最大上下文窗口 (tokens)
    pub max_context_window: u64,
    
    /// 最大输出 tokens
    pub max_output_tokens: u64,
    
    /// 每分钟的请求数限制
    pub requests_per_minute: Option<u64>,
    
    /// 每月的 token 数限制
    pub tokens_per_month: Option<u64>,
}
```

### 3.2 Provider Trait

```rust
/// AI 模型提供商核心 Trait
/// 
/// 实现此 Trait 以集成新的 LLM 提供商。所有 Provider 必须
/// 实现能力声明、工具转换和聊天方法。
/// 
/// 实现必须是 `Send + Sync` 因为在异步任务间共享。
#[async_trait]
pub trait Provider: Send + Sync {
    /// 返回 Provider 能力声明
    fn capabilities(&self) -> ProviderCapabilities;
    
    /// 将内部工具规格转换为 Provider 特定的工具格式
    fn convert_tools(&self, tools: &[ToolSpec]) -> ToolsPayload;
    
    /// 带系统 Prompt 的聊天 (完整功能)
    /// 
    /// # Arguments
    /// * `system_prompt` - 系统级指令
    /// * `messages` - 对话历史
    /// * `tools` - 可用工具列表
    /// * `temperature` - 温度参数
    /// 
    /// # Returns
    /// 返回 ChatResponse 包含文本回复和可能的工具调用
    async fn chat_with_system(
        &self,
        system_prompt: &str,
        messages: &[ChatMessage],
        tools: Option<&[ToolSpec]>,
        temperature: f64,
    ) -> anyhow::Result<ChatResponse>;
    
    /// 简化聊天 (无系统 Prompt 和工具)
    async fn chat(
        &self,
        messages: &[ChatMessage],
        temperature: f64,
    ) -> anyhow::Result<ChatResponse> {
        self.chat_with_system("", messages, None, temperature).await
    }
    
    /// 流式聊天 (可选实现)
    async fn chat_stream(
        &self,
        _system_prompt: &str,
        _messages: &[ChatMessage],
        _tools: Option<&[ToolSpec]>,
        _temperature: f64,
    ) -> anyhow::Result<tokio::sync::mpsc::Receiver<StreamChunk>> {
        bail!("Streaming not supported for this provider")
    }
    
    /// 健康检查
    async fn health_check(&self) -> bool {
        true
    }
    
    /// Provider 标识符
    fn id(&self) -> &str {
        &self.capabilities().name
    }
}
```

## 4. 主要 Provider 实现分析

### 4.1 OpenAI Provider (openai.rs)

#### 4.1.1 结构体定义

```rust
pub struct OpenAIProvider {
    client: reqwest::Client,
    api_key: Secret<String>,
    api_url: String,
    model: String,
    capabilities: ProviderCapabilities,
    timeout: Duration,
}

impl OpenAIProvider {
    pub fn new(config: &OpenAIConfig) -> Result<Self> {
        let client = reqwest::ClientBuilder::new()
            .timeout(Duration::from_secs(config.timeout_secs.unwrap_or(120)))
            .build()?;

        let capabilities = ProviderCapabilities {
            name: "openai".to_string(),
            supports_tools: true,
            supports_vision: config.model.contains("vision") || config.model.contains("gpt-4o"),
            supports_streaming: true,
            supports_system_prompt: true,
            max_context_window: get_openai_context_window(&config.model),
            max_output_tokens: config.max_tokens.unwrap_or(4096),
            requests_per_minute: None,
            tokens_per_month: None,
        };

        Ok(Self {
            client,
            api_key: Secret::new(config.api_key.clone()
                .context("API key required")?),
            api_url: config.api_url.clone()
                .unwrap_or_else(|| "https://api.openai.com/v1".to_string()),
            model: config.model.clone()
                .context("Model required")?,
            capabilities,
            timeout: Duration::from_secs(config.timeout_secs.unwrap_or(120)),
        })
    }
}
```

#### 4.1.2 工具调用实现

```rust
fn convert_tools(&self, tools: &[ToolSpec]) -> ToolsPayload {
    let openai_tools: Vec<OpenAITool> = tools.iter().map(|tool| {
        OpenAITool {
            r#type: "function".to_string(),
            function: OpenAIFunction {
                name: tool.name.clone(),
                description: tool.description.clone(),
                parameters: tool.parameters.clone(),
            },
        }
    }).collect();

    ToolsPayload {
        tools: openai_tools,
        tool_choice: Some(ToolChoice::Auto),
    }
}

#[derive(Debug, Serialize)]
struct OpenAITool {
    r#type: String,
    function: OpenAIFunction,
}

#[derive(Debug, Serialize)]
struct OpenAIFunction {
    name: String,
    description: String,
    parameters: serde_json::Value,
}
```

#### 4.1.3 聊天实现

```rust
async fn chat_with_system(
    &self,
    system_prompt: &str,
    messages: &[ChatMessage],
    tools: Option<&[ToolSpec]>,
    temperature: f64,
) -> anyhow::Result<ChatResponse> {
    // 构建请求体
    let mut payload = serde_json::json!({
        "model": self.model,
        "messages": self.build_messages(system_prompt, messages),
        "temperature": temperature,
    });

    // 添加工具
    if let Some(tools) = tools {
        let tools_payload = self.convert_tools(tools);
        payload["tools"] = serde_json::to_value(tools_payload.tools)?;
        payload["tool_choice"] = serde_json::to_value(tools_payload.tool_choice)?;
    }

    // 发送请求
    let response = self.client
        .post(format!("{}/chat/completions", self.api_url))
        .header("Authorization", format!("Bearer {}", self.api_key.expose()))
        .json(&payload)
        .send()
        .await?;

    // 解析响应
    let completion: OpenAICompletion = response.json().await?;
    
    let choice = completion.choices.first()
        .context("No choices in response")?;
    
    Ok(ChatResponse {
        text: choice.message.content.clone(),
        tool_calls: self.parse_tool_calls(&choice.message.tool_calls)?,
        usage: Some(TokenUsage {
            input_tokens: completion.usage.prompt_tokens as u64,
            output_tokens: completion.usage.completion_tokens as u64,
            total_tokens: completion.usage.total_tokens as u64,
        }),
        finish_reason: choice.finish_reason.clone(),
        raw_response: Some(serde_json::to_value(completion)?),
    })
}
```

### 4.2 Anthropic Provider (anthropic.rs)

#### 4.2.1 特性

- 使用 Messages API (非 Chat Completions)
- 支持 Prompt Caching (Claude 3.5+)
- 工具调用格式不同
- 不支持 system 字段在 messages 数组中

#### 4.2.2 实现要点

```rust
pub struct AnthropicProvider {
    client: reqwest::Client,
    api_key: Secret<String>,
    api_url: String,
    model: String,
    beta_version: String,  // "2023-06-01"
}

impl AnthropicProvider {
    fn build_request(
        &self,
        system_prompt: &str,
        messages: &[ChatMessage],
        tools: Option<&[ToolSpec]>,
        temperature: f64,
        max_tokens: u64,
    ) -> serde_json::Value {
        let mut payload = serde_json::json!({
            "model": self.model,
            "max_tokens": max_tokens,
            "temperature": temperature,
            "system": system_prompt,
            "messages": self.convert_messages(messages),
        });

        if let Some(tools) = tools {
            payload["tools"] = self.convert_tools(tools).0;
        }

        // Prompt caching
        if self.model.contains("claude-3") && self.should_enable_caching() {
            payload["extra_headers"] = serde_json::json!({
                "anthropic-beta": "prompt-caching-2024-07-31"
            });
            
            // 标记系统 prompt 用于缓存
            if let Some(obj) = payload.as_object_mut() {
                obj.insert("system".to_string(), serde_json::json!([
                    {"type": "text", "text": system_prompt, "cache_control": {"type": "ephemeral"}}
                ]));
            }
        }

        payload
    }

    fn convert_messages(&self, messages: &[ChatMessage]) -> Vec<serde_json::Value> {
        messages.iter().map(|msg| {
            let mut content = Vec::new();
            
            // 文本内容
            if !msg.content.is_empty() {
                content.push(serde_json::json!({
                    "type": "text",
                    "text": msg.content
                }));
            }
            
            // 图片内容
            if let Some(images) = &msg.images {
                for image in images {
                    content.push(serde_json::json!({
                        "type": "image",
                        "source": {
                            "type": "base64",
                            "media_type": image.mime_type,
                            "data": image.url  // 假设是 base64
                        }
                    }));
                }
            }
            
            serde_json::json!({
                "role": self.map_role(&msg.role),
                "content": if content.len() == 1 { 
                    content.remove(0) 
                } else { 
                    serde_json::Value::Array(content) 
                }
            })
        }).collect()
    }
}
```

### 4.3 Gemini Provider (gemini.rs)

#### 4.3.1 特性

- Google Generative AI API
- 支持多模态 (文本 + 图片)
- 工具调用称为 "Function Calling"
- 免费层级有速率限制

#### 4.3.2 实现要点

```rust
pub struct GeminiProvider {
    client: reqwest::Client,
    api_key: Secret<String>,
    model: String,
    api_version: String,  // "v1beta"
}

impl GeminiProvider {
    fn build_generative_language_client(&self) -> Result<reqwest::Client> {
        Ok(self.client.clone())
    }

    async fn generate_content(
        &self,
        messages: &[ChatMessage],
        tools: Option<&[ToolSpec]>,
        temperature: f64,
    ) -> Result<GeminiResponse> {
        let url = format!(
            "https://generativelanguage.googleapis.com/{}/models/{}:generateContent?key={}",
            self.api_version,
            self.model,
            self.api_key.expose()
        );

        let payload = serde_json::json!({
            "contents": self.build_contents(messages),
            "generationConfig": {
                "temperature": temperature,
            },
            "tools": tools.map(|t| self.convert_tools(t)),
        });

        let response = self.client
            .post(&url)
            .json(&payload)
            .send()
            .await?;

        Ok(response.json().await?)
    }

    fn build_contents(&self, messages: &[ChatMessage]) -> Vec<GeminiContent> {
        messages.iter().map(|msg| {
            let parts = vec![GeminiPart {
                text: Some(msg.content.clone()),
                inline_data: msg.images.iter().flatten().map(|img| {
                    GeminiBlob {
                        mime_type: img.mime_type.clone(),
                        data: img.url.clone(),
                    }
                }).next(),
                ..Default::default()
            }];

            GeminiContent {
                parts,
                role: if msg.role == "user" { "user" } else { "model" }.to_string(),
            }
        }).collect()
    }
}
```

### 4.4 Compatible Provider (compatible.rs)

#### 4.4.1 目的

适配任何 OpenAI 兼容的 API 端点，包括:
- Ollama
- vLLM
- LocalAI
- FastChat
- 其他兼容服务

#### 4.4.2 实现

```rust
pub struct CompatibleProvider {
    client: reqwest::Client,
    api_key: Option<Secret<String>>,
    api_url: String,
    model: String,
    capabilities: ProviderCapabilities,
}

impl CompatibleProvider {
    pub fn new(config: &CompatibleConfig) -> Result<Self> {
        let client = reqwest::ClientBuilder::new()
            .timeout(Duration::from_secs(config.timeout_secs.unwrap_or(300)))
            .no_proxy()  // 默认不使用代理
            .build()?;

        // 自动检测能力
        let capabilities = Self::detect_capabilities(&client, &config.api_url, &config.model).await?;

        Ok(Self {
            client,
            api_key: config.api_key.as_ref().map(|k| Secret::new(k.clone())),
            api_url: config.api_url.clone(),
            model: config.model.clone(),
            capabilities,
        })
    }

    /// 自动检测远端 API 能力
    async fn detect_capabilities(
        client: &reqwest::Client,
        api_url: &str,
        model: &str,
    ) -> Result<ProviderCapabilities> {
        // 尝试获取模型信息
        if let Ok(resp) = client
            .get(format!("{}/models/{}", api_url, model))
            .send()
            .await
        {
            if resp.status().is_success() {
                let model_info: serde_json::Value = resp.json().await?;
                
                // 提取上下文窗口等信息
                if let Some(context_length) = model_info.get("context_length").and_then(|v| v.as_u64()) {
                    return Ok(ProviderCapabilities {
                        max_context_window: context_length,
                        ..Default::default()
                    });
                }
            }
        }

        // 默认能力
        Ok(ProviderCapabilities {
            name: "compatible".to_string(),
            supports_tools: false,  // 保守假设
            supports_vision: false,
            supports_streaming: true,  // 大多数兼容
            supports_system_prompt: true,
            max_context_window: 4096,  // 保守值
            max_output_tokens: 2048,
            ..Default::default()
        })
    }
}
```

## 5. Resilient Wrapper (reliable.rs - 70.4KB)

### 5.1 装饰器模式实现

```rust
pub struct ReliableProvider<P: Provider> {
    inner: Arc<P>,
    retry_config: RetryConfig,
    fallback_provider: Option<Arc<dyn Provider>>,
    circuit_breaker: CircuitBreaker,
    rate_limiter: RateLimiter,
    cost_tracker: Arc<Mutex<CostTracker>>,
}

#[derive(Debug, Clone)]
pub struct RetryConfig {
    pub max_retries: u32,
    pub initial_delay_ms: u64,
    pub max_delay_ms: u64,
    pub exponential_base: u64,
    pub jitter: bool,
}

impl Default for RetryConfig {
    fn default() -> Self {
        Self {
            max_retries: 3,
            initial_delay_ms: 100,
            max_delay_ms: 10000,
            exponential_base: 2,
            jitter: true,
        }
    }
}
```

### 5.2 重试逻辑

```rust
impl<P: Provider> ReliableProvider<P> {
    /// 带重试的聊天调用
    async fn chat_with_retry(
        &self,
        system_prompt: &str,
        messages: &[ChatMessage],
        tools: Option<&[ToolSpec]>,
        temperature: f64,
    ) -> Result<ChatResponse> {
        let mut last_error = None;
        
        for attempt in 0..self.retry_config.max_retries {
            // 检查熔断器
            if !self.circuit_breaker.allow_request() {
                bail!("Circuit breaker is open");
            }

            match self.inner.chat_with_system(
                system_prompt,
                messages,
                tools,
                temperature,
            ).await {
                Ok(response) => {
                    // 成功：重置熔断器
                    self.circuit_breaker.record_success();
                    return Ok(response);
                }
                Err(e) => {
                    // 失败：记录错误
                    self.circuit_breaker.record_failure();
                    last_error = Some(e);

                    // 判断是否可重试
                    if !self.is_retryable_error(&last_error.unwrap()) {
                        break;  // 不可重试的错误直接返回
                    }

                    // 计算延迟
                    if attempt < self.retry_config.max_retries - 1 {
                        let delay = self.calculate_backoff(attempt);
                        tokio::time::sleep(delay).await;
                    }
                }
            }
        }

        // 所有重试失败
        if let Some(fallback) = &self.fallback_provider {
            warn!("Primary provider failed, using fallback");
            return fallback.chat_with_system(
                system_prompt,
                messages,
                tools,
                temperature,
            ).await;
        }

        Err(last_error.unwrap())
    }

    /// 判断错误是否可重试
    fn is_retryable_error(&self, error: &anyhow::Error) -> bool {
        let error_str = error.to_string().to_lowercase();
        
        // 可重试的错误
        error_str.contains("timeout") ||
        error_str.contains("rate limit") ||
        error_str.contains("service unavailable") ||
        error_str.contains("too many requests") ||
        error_str.contains("503") ||
        error_str.contains("429")
    }

    /// 计算退避延迟
    fn calculate_backoff(&self, attempt: u32) -> Duration {
        let delay = self.retry_config.initial_delay_ms as u64
            * self.retry_config.exponential_base.pow(attempt);
        
        let delay = delay.min(self.retry_config.max_delay_ms);
        
        // 添加抖动
        if self.retry_config.jitter {
            let jitter = rand::random::<u64>() % (delay / 2);
            Duration::from_millis(delay + jitter)
        } else {
            Duration::from_millis(delay)
        }
    }
}
```

### 5.3 熔断器

```rust
pub struct CircuitBreaker {
    state: RwLock<CircuitState>,
    failure_threshold: u32,
    success_threshold: u32,
    timeout: Duration,
}

enum CircuitState {
    Closed,      // 正常状态
    Open,        // 熔断状态
    HalfOpen,    // 半开状态 (尝试恢复)
}

impl CircuitBreaker {
    pub fn new(failure_threshold: u32, success_threshold: u32, timeout: Duration) -> Self {
        Self {
            state: RwLock::new(CircuitState::Closed),
            failure_threshold,
            success_threshold,
            timeout,
        }
    }

    pub fn allow_request(&self) -> bool {
        let mut state = self.state.write().unwrap();
        
        match *state {
            CircuitState::Closed => true,
            CircuitState::Open => {
                // 检查超时
                // TODO: 实际实现需要记录打开时间
                false
            }
            CircuitState::HalfOpen => true,
        }
    }

    pub fn record_success(&self) {
        let mut state = self.state.write().unwrap();
        
        match *state {
            CircuitState::HalfOpen => {
                // 连续成功达到阈值则关闭
                // TODO: 需要计数器
                *state = CircuitState::Closed;
            }
            _ => {}
        }
    }

    pub fn record_failure(&self) {
        let mut state = self.state.write().unwrap();
        
        match *state {
            CircuitState::Closed => {
                // 失败达到阈值则打开
                // TODO: 需要计数器
                *state = CircuitState::Open;
            }
            CircuitState::HalfOpen => {
                // 半开状态失败立即打开
                *state = CircuitState::Open;
            }
            _ => {}
        }
    }
}
```

## 6. Provider 路由器 (router.rs)

```rust
pub struct RouterProvider {
    routes: Vec<(String, Arc<dyn Provider>)>,  // hint -> provider
    default_provider: Arc<dyn Provider>,
}

impl RouterProvider {
    pub fn new(
        routes: Vec<(String, Arc<dyn Provider>)>,
        default_provider: Arc<dyn Provider>,
    ) -> Self {
        Self {
            routes,
            default_provider,
        }
    }

    /// 根据 hint 选择 Provider
    fn select_provider(&self, hint: Option<&str>) -> Arc<dyn Provider> {
        if let Some(hint) = hint {
            for (route_hint, provider) in &self.routes {
                if route_hint == hint {
                    return provider.clone();
                }
            }
        }
        
        self.default_provider.clone()
    }
}

#[async_trait]
impl Provider for RouterProvider {
    fn capabilities(&self) -> ProviderCapabilities {
        // 返回所有 Provider 能力的并集
        let mut caps = self.default_provider.capabilities();
        
        for (_, provider) in &self.routes {
            let provider_caps = provider.capabilities();
            caps.supports_tools |= provider_caps.supports_tools;
            caps.supports_vision |= provider_caps.supports_vision;
            caps.supports_streaming |= provider_caps.supports_streaming;
        }
        
        caps
    }

    fn convert_tools(&self, tools: &[ToolSpec]) -> ToolsPayload {
        // 使用默认 Provider 的转换
        self.default_provider.convert_tools(tools)
    }

    async fn chat_with_system(
        &self,
        system_prompt: &str,
        messages: &[ChatMessage],
        tools: Option<&[ToolSpec]>,
        temperature: f64,
    ) -> Result<ChatResponse> {
        // 根据消息内容自动选择 Provider
        let hint = self.classify_message(messages).await;
        let provider = self.select_provider(hint.as_deref());
        
        provider
            .chat_with_system(system_prompt, messages, tools, temperature)
            .await
    }

    async fn classify_message(&self, messages: &[ChatMessage]) -> Option<String> {
        // 简单的启发式分类
        let last_message = messages.last()?.content.to_lowercase();
        
        if last_message.contains("代码") || last_message.contains("programming") {
            return Some("coding-assistant".to_string());
        }
        
        if last_message.contains("图片") || last_message.contains("image") {
            return Some("vision-model".to_string());
        }
        
        None
    }
}
```

## 7. 配置选项

```toml
# Provider 配置示例
[model_providers.openai]
type = "openai"
api_key = "sk-..."
model = "gpt-4o"
api_url = "https://api.openai.com/v1"
max_tokens = 4096
timeout_secs = 120

[model_providers.anthropic]
type = "anthropic"
api_key = "sk-ant-..."
model = "claude-sonnet-4-6"
max_tokens = 8192

[model_providers.gemini]
type = "gemini"
api_key = "..."
model = "gemini-1.5-pro"
api_version = "v1beta"

[model_providers.ollama]
type = "compatible"
api_url = "http://localhost:11434/v1"
model = "llama3.1"
timeout_secs = 300

# 可靠性配置
[reliability]
max_retries = 3
initial_delay_ms = 100
max_delay_ms = 10000
exponential_base = 2
jitter = true

# 路由规则
[[model_routes]]
hint = "coding-assistant"
provider = "anthropic"
model = "claude-sonnet-4-6"

[[model_routes]]
hint = "vision-model"
provider = "openai"
model = "gpt-4o"

# 降级配置
[fallback]
enabled = true
primary = "openai"
fallback = "ollama"
```

## 8. 错误处理

### 8.1 Provider 特定错误

```rust
pub enum ProviderError {
    AuthenticationFailed(String),
    RateLimitExceeded { retry_after: Option<Duration> },
    ModelNotFound(String),
    ContextWindowExceeded { used: u64, limit: u64 },
    InvalidRequest(String),
    ServerError { code: u16, message: String },
    Timeout,
    NetworkError(anyhow::Error),
}

impl From<reqwest::Error> for ProviderError {
    fn from(err: reqwest::Error) -> Self {
        if err.is_timeout() {
            Self::Timeout
        } else if err.is_connect() {
            Self::NetworkError(err.into())
        } else {
            Self::NetworkError(err.into())
        }
    }
}
```
