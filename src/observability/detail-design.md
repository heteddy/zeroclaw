# Observability 模块详细设计

## 1. 模块概述

Observability 模块提供 ZeroClaw 的可观测性基础设施，包括日志记录、指标收集、分布式追踪和事件监控。支持多种后端 (Console、Prometheus、OpenTelemetry),实现统一的遥测数据输出。

## 2. 架构设计

### 2.1 模块结构

src/observability/
- mod.rs          - 模块导出和多路复用器 (5.5KB)
- traits.rs       - Observer Trait 定义 (6.5KB - 核心)
- log.rs          - 结构化日志记录 (6.2KB)
- prometheus.rs   - Prometheus 指标导出 (16.9KB)
- otel.rs         - OpenTelemetry 集成 (19.2KB)
- runtime_trace.rs - 运行时追踪 (12.2KB)
- multi.rs        - 多路观察者复用器 (4.6KB)
- noop.rs         - 空观察者 (2.3KB)
- verbose.rs      - 详细日志模式 (2.9KB)

### 2.2 设计模式

- **观察者模式**: 多个 Observer 监听同一事件流
- **装饰器模式**: 在基础 Observer 上增加功能
- **组合模式**: MultiObserver 组合多个后端
- **空对象模式**: NoopObserver 用于禁用可观测性

### 2.3 数据流

```
Agent Runtime
    ↓ (发出事件)
ObserverEvent / ObserverMetric
    ↓ (路由到)
MultiObserver
    ├→ ConsoleObserver (日志输出)
    ├→ PrometheusObserver (指标收集)
    ├→ OpenTelemetryObserver (分布式追踪)
    └→ CustomObserver (自定义后端)
```

## 3. 核心 Trait 详解 (traits.rs)

### 3.1 观察者事件枚举

```rust
/// Agent 运行时发出的离散事件
#[derive(Debug, Clone)]
pub enum ObserverEvent {
    /// Agent 编排循环启动新会话
    AgentStart { 
        provider: String,  // Provider 名称
        model: String,     // 模型名称
    },
    
    /// 即将向 LLM Provider 发送请求
    /// 
    /// 在 Provider 调用前立即发出，用于打印进度但不泄露 Prompt 内容
    LlmRequest {
        provider: String,
        model: String,
        messages_count: usize,  // 消息数量 (用于估算成本)
    },
    
    /// LLM Provider 调用结果
    LlmResponse {
        provider: String,
        model: String,
        duration: Duration,           // 耗时
        success: bool,                // 是否成功
        error_message: Option<String>, // 错误消息 (已脱敏)
        input_tokens: Option<u64>,    // 输入 token 数
        output_tokens: Option<u64>,   // 输出 token 数
    },
    
    /// Agent 会话结束
    /// 
    /// 携带聚合的使用量数据 (tokens、成本)
    AgentEnd {
        provider: String,
        model: String,
        duration: Duration,
        tokens_used: Option<u64>,
        cost_usd: Option<f64>,
    },
    
    /// 工具调用开始
    ToolCallStart { tool: String },
    
    /// 工具调用完成
    ToolCall {
        tool: String,
        duration: Duration,
        success: bool,
    },
    
    /// 一轮对话结束 (Turn-based)
    TurnComplete,
    
    /// 渠道消息收发
    ChannelMessage {
        channel: String,      // 渠道名称
        direction: String,    // "inbound" 或 "outbound"
    },
    
    /// 心跳检测
    HeartbeatTick,
    
    /// 组件错误
    Error {
        component: String,  // 子系统名称
        message: String,    // 错误描述 (不含敏感信息)
    },
}
```

### 3.2 观察者指标枚举

```rust
/// 运行时数值指标
#[derive(Debug, Clone)]
pub enum ObserverMetric {
    /// LLM 或工具请求的耗时
    RequestLatency(Duration),
    
    /// Token 消耗数
    TokensUsed(u64),
    
    /// 当前活跃并发会话数
    ActiveSessions(u64),
    
    /// 待处理消息队列深度
    QueueDepth(u64),
}
```

### 3.3 Observer Trait

```rust
/// 记录 Agent 运行时遥测数据的核心 Trait
/// 
/// 实现此 Trait 以集成任意监控后端 (结构化日志、Prometheus、
/// OpenTelemetry 等)。Agent 运行时持有一个或多个 Observer 实例，
/// 在关键生命周期点调用 record_event 和 record_metric。
/// 
/// 实现必须是 `Send + Sync + 'static` 因为 Observer 通过 Arc 在
/// 异步任务间共享。
pub trait Observer: Send + Sync + 'static {
    /// 记录离散的生命周期事件
    /// 
    /// 在热路径上同步调用；实现应避免阻塞 I/O。
    /// 应在内部缓冲事件并异步刷新。
    fn record_event(&self, event: &ObserverEvent);

    /// 记录数值指标样本
    /// 
    /// 同步调用；同样应非阻塞。
    fn record_metric(&self, metric: &ObserverMetric);

    /// 刷新任何缓冲的遥测数据到后端
    /// 
    /// 运行时的优雅关闭期间调用。
    /// 默认不操作，适用于同步写入的后端。
    fn flush(&self) {}

    /// 返回观察者后端的可读名称
    fn name(&self) -> &str;

    /// 下转到 Any 类型，用于后端特定操作
    fn as_any(&self) -> &dyn std::any::Any;
}
```

## 4. 后端实现详解

### 4.1 控制台日志 (log.rs)

```rust
pub struct ConsoleObserver {
    verbose: bool,
    start_time: Instant,
}

impl ConsoleObserver {
    pub fn new(verbose: bool) -> Self {
        Self {
            verbose,
            start_time: Instant::now(),
        }
    }

    fn format_duration(&self, duration: Duration) -> String {
        let secs = duration.as_secs();
        if secs < 1 {
            format!("{}ms", duration.as_millis())
        } else if secs < 60 {
            format!("{:.1}s", secs as f64)
        } else {
            format!("{}m {}s", secs / 60, secs % 60)
        }
    }
}

impl Observer for ConsoleObserver {
    fn record_event(&self, event: &ObserverEvent) {
        let elapsed = self.start_time.elapsed();
        
        match event {
            ObserverEvent::AgentStart { provider, model } => {
                info!(
                    "[{:.3}s] 🚀 Agent started with {}/{}",
                    elapsed.as_secs_f64(),
                    provider,
                    model
                );
            }
            
            ObserverEvent::LlmRequest { provider, model, messages_count } => {
                if self.verbose {
                    info!(
                        "[{:.3}s] 💬 Sending request to {}/{} ({} messages)",
                        elapsed.as_secs_f64(),
                        provider,
                        model,
                        messages_count
                    );
                }
            }
            
            ObserverEvent::LlmResponse {
                provider,
                model,
                duration,
                success,
                input_tokens,
                output_tokens,
                ..
            } => {
                if *success {
                    let tokens = match (input_tokens, output_tokens) {
                        (Some(input), Some(output)) => {
                            format!(" {}t+{}t", input, output)
                        }
                        _ => String::new(),
                    };
                    
                    info!(
                        "[{:.3}s] ✅ {}/{} in {}{}",
                        elapsed.as_secs_f64(),
                        provider,
                        model,
                        self.format_duration(*duration),
                        tokens
                    );
                } else {
                    error!(
                        "[{:.3}s] ❌ {}/{} failed after {}",
                        elapsed.as_secs_f64(),
                        provider,
                        model,
                        self.format_duration(*duration)
                    );
                }
            }
            
            ObserverEvent::ToolCallStart { tool } => {
                if self.verbose {
                    info!("[{:.3}s] 🔧 Executing tool: {}", elapsed.as_secs_f64(), tool);
                }
            }
            
            ObserverEvent::ToolCall { tool, duration, success } => {
                if *success {
                    debug!(
                        "[{:.3}s] ✓ {} completed in {}",
                        elapsed.as_secs_f64(),
                        tool,
                        self.format_duration(*duration)
                    );
                } else {
                    warn!(
                        "[{:.3}s] ✗ {} failed after {}",
                        elapsed.as_secs_f64(),
                        tool,
                        self.format_duration(*duration)
                    );
                }
            }
            
            ObserverEvent::TurnComplete => {
                info!("[{:.3}s] 📝 Turn complete", elapsed.as_secs_f64());
            }
            
            ObserverEvent::Error { component, message } => {
                error!(
                    "[{:.3}s] ⚠️  Error in {}: {}",
                    elapsed.as_secs_f64(),
                    component,
                    message
                );
            }
            
            _ => {
                if self.verbose {
                    trace!("[{:.3}s] {:?}", elapsed.as_secs_f64(), event);
                }
            }
        }
    }

    fn record_metric(&self, metric: &ObserverMetric) {
        if !self.verbose {
            return;
        }

        match metric {
            ObserverMetric::TokensUsed(n) => {
                debug!("Total tokens used: {}", n);
            }
            ObserverMetric::ActiveSessions(n) => {
                debug!("Active sessions: {}", n);
            }
            _ => {}
        }
    }

    fn name(&self) -> &str {
        "console"
    }

    fn as_any(&self) -> &dyn std::any::Any {
        self
    }
}
```

### 4.2 Prometheus 指标 (prometheus.rs)

```rust
use prometheus::{Registry, Counter, Gauge, Histogram, register_counter_with_registry};

pub struct PrometheusObserver {
    registry: Registry,
    
    // 计数器
    llm_requests_total: Counter,
    llm_tokens_total: Counter,
    tool_calls_total: Counter,
    
    // 直方图 (延迟分布)
    llm_request_duration: Histogram,
    tool_call_duration: Histogram,
    
    // 仪表盘
    active_sessions: Gauge,
    queue_depth: Gauge,
}

impl PrometheusObserver {
    pub fn new() -> Result<Self> {
        let registry = Registry::new();

        // 注册指标
        let llm_requests_total = register_counter_with_registry!(
            "zeroclaw_llm_requests_total",
            "Total number of LLM requests made",
            registry
        )?;

        let llm_tokens_total = register_counter_with_registry!(
            "zeroclaw_llm_tokens_total",
            "Total number of tokens consumed",
            registry
        )?;

        let tool_calls_total = Counter::new(
            "zeroclaw_tool_calls_total",
            "Total number of tool calls"
        )?.register_with_registry(&registry)?;

        let llm_request_duration = Histogram::with_opts(
            prometheus::HistogramOpts::new(
                "zeroclaw_llm_request_duration_seconds",
                "LLM request duration in seconds"
            ).buckets(vec![0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0])
        )?.register_with_registry(&registry)?;

        let tool_call_duration = Histogram::with_opts(
            prometheus::HistogramOpts::new(
                "zeroclaw_tool_call_duration_seconds",
                "Tool call duration in seconds"
            ).buckets(vec![0.01, 0.05, 0.1, 0.5, 1.0, 5.0])
        )?.register_with_registry(&registry)?;

        let active_sessions = register_gauge_with_registry!(
            "zeroclaw_active_sessions",
            "Number of active concurrent sessions",
            registry
        )?;

        let queue_depth = register_gauge_with_registry!(
            "zeroclaw_queue_depth",
            "Current depth of the inbound message queue",
            registry
        )?;

        Ok(Self {
            registry,
            llm_requests_total,
            llm_tokens_total,
            tool_calls_total,
            llm_request_duration,
            tool_call_duration,
            active_sessions,
            queue_depth,
        })
    }

    /// 获取 Prometheus 注册表 (用于 HTTP 暴露)
    pub fn registry(&self) -> &Registry {
        &self.registry
    }

    /// 收集指标为 Prometheus 格式
    pub fn gather(&self) -> Vec<prometheus::proto::MetricFamily> {
        self.registry.gather()
    }
}

impl Observer for PrometheusObserver {
    fn record_event(&self, event: &ObserverEvent) {
        match event {
            ObserverEvent::LlmRequest { .. } => {
                self.llm_requests_total.inc();
            }
            
            ObserverEvent::LlmResponse {
                duration,
                input_tokens,
                output_tokens,
                ..
            } => {
                // 记录延迟
                self.llm_request_duration.observe(duration.as_secs_f64());
                
                // 记录 token 数
                if let Some(tokens) = input_tokens {
                    self.llm_tokens_total.inc_by(*tokens as f64);
                }
                if let Some(tokens) = output_tokens {
                    self.llm_tokens_total.inc_by(*tokens as f64);
                }
            }
            
            ObserverEvent::ToolCall { duration, .. } => {
                self.tool_calls_total.inc();
                self.tool_call_duration.observe(duration.as_secs_f64());
            }
            
            ObserverEvent::AgentStart { .. } => {
                self.active_sessions.inc();
            }
            
            ObserverEvent::AgentEnd { .. } => {
                self.active_sessions.dec();
            }
            
            _ => {}
        }
    }

    fn record_metric(&self, metric: &ObserverMetric) {
        match metric {
            ObserverMetric::ActiveSessions(n) => {
                self.active_sessions.set(*n as f64);
            }
            ObserverMetric::QueueDepth(n) => {
                self.queue_depth.set(*n as f64);
            }
            _ => {}
        }
    }

    fn flush(&self) {
        // Prometheus 指标是即时更新的，无需刷新
    }

    fn name(&self) -> &str {
        "prometheus"
    }

    fn as_any(&self) -> &dyn std::any::Any {
        self
    }
}
```

### 4.3 OpenTelemetry 追踪 (otel.rs - 简化版)

```rust
use opentelemetry::{
    global,
    trace::{Span, Tracer},
    KeyValue,
};

pub struct OpenTelemetryObserver {
    tracer: opentelemetry::sdk::trace::Tracer,
}

impl OpenTelemetryObserver {
    pub fn new(service_name: &str, endpoint: &str) -> Result<Self> {
        // 配置 OTLP 导出器
        let exporter = opentelemetry_otlp::new_exporter()
            .tonic()
            .with_endpoint(endpoint);

        let tracer = opentelemetry_otlp::new_pipeline()
            .tracing()
            .with_exporter(exporter)
            .with_trace_config(
                opentelemetry::sdk::trace::config()
                    .with_resource(Resource::new(vec![KeyValue::new(
                        "service.name",
                        service_name,
                    )]))
                    .with_sampler(Sampler::AlwaysOn)
            )
            .install_batch(opentelemetry::runtime::Tokio)?;

        Ok(Self { tracer })
    }

    fn start_span(&self, name: &str) -> Box<dyn Span + Send + Sync> {
        Box::new(self.tracer.start(name))
    }
}

impl Observer for OpenTelemetryObserver {
    fn record_event(&self, event: &ObserverEvent) {
        match event {
            ObserverEvent::AgentStart { provider, model } => {
                let span = self.start_span("agent_session");
                span.set_attribute(KeyValue::new("provider", provider.clone()));
                span.set_attribute(KeyValue::new("model", model.clone()));
            }
            
            ObserverEvent::LlmRequest { provider, model, .. } => {
                let span = self.start_span("llm_request");
                span.set_attribute(KeyValue::new("provider", provider.clone()));
                span.set_attribute(KeyValue::new("model", model.clone()));
            }
            
            ObserverEvent::LlmResponse {
                duration,
                input_tokens,
                output_tokens,
                ..
            } => {
                if let Some(span) = self.tracer.in_span_context() {
                    span.set_attribute(KeyValue::new(
                        "llm.duration_ms",
                        duration.as_millis() as i64,
                    ));
                    if let Some(tokens) = input_tokens {
                        span.set_attribute(KeyValue::new("llm.input_tokens", *tokens as i64));
                    }
                    if let Some(tokens) = output_tokens {
                        span.set_attribute(KeyValue::new("llm.output_tokens", *tokens as i64));
                    }
                    span.end();
                }
            }
            
            ObserverEvent::ToolCall { tool, duration, success } => {
                let span = self.start_span("tool_call");
                span.set_attribute(KeyValue::new("tool.name", tool.clone()));
                span.set_attribute(KeyValue::new(
                    "tool.duration_ms",
                    duration.as_millis() as i64,
                ));
                span.set_attribute(KeyValue::new("tool.success", *success));
                span.end();
            }
            
            _ => {}
        }
    }

    fn record_metric(&self, _metric: &ObserverMetric) {
        // OpenTelemetry Metrics 可通过单独的 API 记录
    }

    fn flush(&self) {
        // 强制刷新所有挂起的 spans
        opentelemetry::global::shutdown_tracer_provider();
    }

    fn name(&self) -> &str {
        "opentelemetry"
    }

    fn as_any(&self) -> &dyn std::any::Any {
        self
    }
}
```

### 4.4 多路复用器 (multi.rs)

```rust
pub struct MultiObserver {
    observers: Vec<Arc<dyn Observer>>,
}

impl MultiObserver {
    pub fn new() -> Self {
        Self {
            observers: Vec::new(),
        }
    }

    /// 添加观察者
    pub fn add(&mut self, observer: Arc<dyn Observer>) {
        self.observers.push(observer);
    }

    /// 移除观察者
    pub fn remove(&mut self, name: &str) {
        self.observers.retain(|obs| obs.name() != name);
    }

    /// 获取所有观察者名称
    pub fn names(&self) -> Vec<&str> {
        self.observers.iter().map(|o| o.name()).collect()
    }
}

impl Observer for MultiObserver {
    fn record_event(&self, event: &ObserverEvent) {
        // 并行记录到所有观察者
        self.observers.iter().for_each(|observer| {
            observer.record_event(event);
        });
    }

    fn record_metric(&self, metric: &ObserverMetric) {
        self.observers.iter().for_each(|observer| {
            observer.record_metric(metric);
        });
    }

    fn flush(&self) {
        // 并行刷新所有观察者
        self.observers.iter().for_each(|observer| {
            observer.flush();
        });
    }

    fn name(&self) -> &str {
        "multi"
    }

    fn as_any(&self) -> &dyn std::any::Any {
        self
    }
}
```

## 5. 运行时追踪 (runtime_trace.rs)

```rust
/// 运行时追踪上下文
pub struct TraceContext {
    pub trace_id: String,
    pub span_id: String,
    pub parent_span_id: Option<String>,
    pub attributes: HashMap<String, String>,
}

impl TraceContext {
    pub fn new() -> Self {
        Self {
            trace_id: generate_trace_id(),
            span_id: generate_span_id(),
            parent_span_id: None,
            attributes: HashMap::new(),
        }
    }

    pub fn child(&self) -> Self {
        Self {
            trace_id: self.trace_id.clone(),
            span_id: generate_span_id(),
            parent_span_id: Some(self.span_id.clone()),
            attributes: self.attributes.clone(),
        }
    }

    pub fn with_attribute(mut self, key: &str, value: &str) -> Self {
        self.attributes.insert(key.to_string(), value.to_string());
        self
    }
}

/// 追踪 Guard - RAII 模式管理 Span 生命周期
pub struct TraceGuard {
    context: TraceContext,
    observer: Arc<dyn Observer>,
    start_time: Instant,
}

impl TraceGuard {
    pub fn new(context: TraceContext, observer: Arc<dyn Observer>) -> Self {
        Self {
            context,
            observer,
            start_time: Instant::now(),
        }
    }

    pub fn record_event(&self, event: ObserverEvent) {
        self.observer.record_event(&event);
    }
}

impl Drop for TraceGuard {
    fn drop(&mut self) {
        let duration = self.start_time.elapsed();
        // 自动记录 Span 结束事件
        self.observer.record_event(&ObserverEvent::TurnComplete);
    }
}

/// 创建新的追踪 Span
pub fn start_span(
    name: &str,
    observer: Arc<dyn Observer>,
) -> TraceGuard {
    let context = TraceContext::new()
        .with_attribute("span.name", name);
    
    TraceGuard::new(context, observer)
}
```

## 6. 配置选项

```toml
[observability]
enabled = true
verbose = false  # 详细日志模式

[observability.console]
enabled = true
verbose_events = false  # 只显示摘要事件

[observability.prometheus]
enabled = true
port = 9090  # 指标暴露端口
path = "/metrics"

[observability.opentelemetry]
enabled = false
endpoint = "http://localhost:4317"  # OTLP gRPC 端点
service_name = "zeroclaw-agent"
sampling_rate = 1.0  # 100% 采样

[observability.log]
level = "info"  # trace/debug/info/warn/error
format = "json"  # json/text
output = "stdout"  # stdout/file
file_path = "~/.zeroclaw/agent.log"
max_size_mb = 100
rotation = "daily"
```

## 7. 使用示例

```rust
// 创建多路观察者
let mut multi_observer = MultiObserver::new();

// 添加控制台观察者
multi_observer.add(Arc::new(ConsoleObserver::new(false)));

// 添加 Prometheus 观察者
if config.observability.prometheus.enabled {
    let prom_observer = PrometheusObserver::new()?;
    
    // 启动 HTTP 服务器暴露指标
    let registry = prom_observer.registry().clone();
    tokio::spawn(async move {
        serve_metrics(registry, config.observability.prometheus.port).await;
    });
    
    multi_observer.add(Arc::new(prom_observer));
}

// 添加 OpenTelemetry 观察者
if config.observability.opentelemetry.enabled {
    let otel_observer = OpenTelemetryObserver::new(
        "zeroclaw-agent",
        &config.observability.opentelemetry.endpoint,
    )?;
    multi_observer.add(Arc::new(otel_observer));
}

// 设置全局观察者
let observer = Arc::new(multi_observer);

// 在 Agent 运行时中使用
observer.record_event(&ObserverEvent::AgentStart {
    provider: "openai".to_string(),
    model: "gpt-4".to_string(),
});

// 使用追踪 Span
let _span = start_span("agent_turn", observer.clone());
// ... 执行 Agent 逻辑 ...
// Drop 时自动记录结束
```

## 8. 性能考虑

### 8.1 异步批量处理

```rust
pub struct BatchedObserver {
    inner: Arc<dyn Observer>,
    tx: mpsc::Sender<BatchedEvent>,
}

impl BatchedObserver {
    pub fn new(inner: Arc<dyn Observer>, batch_size: usize, flush_interval: Duration) -> Self {
        let (tx, mut rx) = mpsc::channel::<BatchedEvent>(1000);

        // 后台刷新任务
        tokio::spawn(async move {
            let mut buffer = Vec::with_capacity(batch_size);
            let mut interval = tokio::time::interval(flush_interval);

            loop {
                tokio::select! {
                    event = rx.recv() => {
                        match event {
                            Some(e) => {
                                buffer.push(e);
                                if buffer.len() >= batch_size {
                                    flush_buffer(&mut buffer, &inner).await;
                                }
                            }
                            None => break,  // 通道关闭
                        }
                    }
                    _ = interval.tick() => {
                        if !buffer.is_empty() {
                            flush_buffer(&mut buffer, &inner).await;
                        }
                    }
                }
            }
        });

        Self { inner, tx }
    }
}

async fn flush_buffer(buffer: &mut Vec<BatchedEvent>, inner: &Arc<dyn Observer>) {
    for event in buffer.drain(..) {
        inner.record_event(&event.event);
        inner.record_metric(&event.metric);
    }
}
```

## 9. 测试策略

### 9.1 Mock 观察者

```rust
pub struct MockObserver {
    pub events: Mutex<Vec<ObserverEvent>>,
    pub metrics: Mutex<Vec<ObserverMetric>>,
}

impl Observer for MockObserver {
    fn record_event(&self, event: &ObserverEvent) {
        self.events.lock().push(event.clone());
    }

    fn record_metric(&self, metric: &ObserverMetric) {
        self.metrics.lock().push(metric.clone());
    }

    fn name(&self) -> &str {
        "mock"
    }

    fn as_any(&self) -> &dyn std::any::Any {
        self
    }
}

#[test]
fn test_agent_emits_start_and_end_events() {
    let observer = Arc::new(MockObserver::default());
    
    // 运行 Agent
    run_agent_with_observer(observer.clone()).await;
    
    // 验证事件
    let events = observer.events.lock();
    assert!(events.iter().any(|e| matches!(e, ObserverEvent::AgentStart { .. })));
    assert!(events.iter().any(|e| matches!(e, ObserverEvent::AgentEnd { .. })));
}
```
