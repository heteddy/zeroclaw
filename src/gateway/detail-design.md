# Gateway 模块详细设计

## 1. 模块概述

Gateway 模块是基于 Axum 的 HTTP 网关服务器，提供符合 HTTP/1.1 规范的 Webhook 接收、WebSocket 通信和 SSE 流式传输功能。作为 ZeroClaw 与外部系统 (如 Telegram、WhatsApp、Discord 等) 的桥梁。

## 2. 架构设计

### 2.1 模块结构

src/gateway/
- mod.rs          - 核心网关实现 (94.4KB)
- api.rs          - REST API 端点 (48.9KB)
- ws.rs           - WebSocket 处理 (5.3KB)
- sse.rs          - SSE 服务器发送事件 (4.9KB)
- static_files.rs - 静态文件服务 (1.7KB)

### 2.2 设计模式

- **中间件模式**: Tower 中间件链式处理
- **状态管理模式**: Arc 共享状态 + Axum State
- **工厂模式**: 路由处理器工厂
- **装饰器模式**: 限流、超时、认证中间件

## 3. 核心组件详解

### 3.1 滑动窗口限流器 (SlidingWindowRateLimiter)

```rust
pub struct SlidingWindowRateLimiter {
    limit_per_window: u32,      // 窗口内限制次数
    window: Duration,            // 窗口大小
    max_keys: usize,             // 最大跟踪键数
    requests: Mutex<(HashMap<String, Vec<Instant>>, Instant)>,
}

impl SlidingWindowRateLimiter {
    fn new(limit_per_window: u32, window: Duration, max_keys: usize) -> Self {
        Self {
            limit_per_window,
            window,
            max_keys: max_keys.max(1),
            requests: Mutex::new((HashMap::new(), Instant::now())),
        }
    }

    /// 检查请求是否允许
    fn allow(&self, key: &str) -> bool {
        if self.limit_per_window == 0 {
            return true;  // 无限制
        }

        let now = Instant::now();
        let cutoff = now.checked_sub(self.window).unwrap_or_else(Instant::now);

        let mut guard = self.requests.lock();
        let (requests, last_sweep) = &mut *guard;

        // 定期清理过期键
        if last_sweep.elapsed() >= Duration::from_secs(RATE_LIMITER_SWEEP_INTERVAL_SECS) {
            Self::prune_stale(requests, cutoff);
            *last_sweep = now;
        }

        // 容量保护：超出最大键数时淘汰最旧的键
        if !requests.contains_key(key) && requests.len() >= self.max_keys {
            Self::prune_stale(requests, cutoff);
            
            if requests.len() >= self.max_keys {
                let evict_key = requests
                    .iter()
                    .min_by_key(|(_, timestamps)| timestamps.last().copied().unwrap_or(cutoff))
                    .map(|(k, _)| k.clone());
                if let Some(evict_key) = evict_key {
                    requests.remove(&evict_key);
                }
            }
        }

        let entry = requests.entry(key.to_owned()).or_default();
        entry.retain(|instant| *instant > cutoff);

        if entry.len() >= self.limit_per_window as usize {
            return false;  // 超出限制
        }

        entry.push(now);
        true
    }
}
```

### 3.2 网关限流器 (GatewayRateLimiter)

```rust
pub struct GatewayRateLimiter {
    pair: SlidingWindowRateLimiter,     // 配对请求限流
    webhook: SlidingWindowRateLimiter,  // Webhook 请求限流
}

impl GatewayRateLimiter {
    fn new(pair_per_minute: u32, webhook_per_minute: u32, max_keys: usize) -> Self {
        let window = Duration::from_secs(RATE_LIMIT_WINDOW_SECS);
        Self {
            pair: SlidingWindowRateLimiter::new(pair_per_minute, window, max_keys),
            webhook: SlidingWindowRateLimiter::new(webhook_per_minute, window, max_keys),
        }
    }

    fn allow_pair(&self, key: &str) -> bool {
        self.pair.allow(key)
    }

    fn allow_webhook(&self, key: &str) -> bool {
        self.webhook.allow(key)
    }
}
```

### 3.3 幂等性存储 (IdempotencyStore)

```rust
pub struct IdempotencyStore {
    ttl: Duration,       // 键存活时间
    max_keys: usize,     // 最大键数
    keys: Mutex<HashMap<String, Instant>>,
}

impl IdempotencyStore {
    /// 记录新键，返回 true 表示是新的
    fn record_if_new(&self, key: &str) -> bool {
        let now = Instant::now();
        let mut keys = self.keys.lock();

        // 定期清理过期键
        if keys.len() > self.max_keys {
            keys.retain(|_, expiry| *expiry > now);
        }

        // 如果键不存在或已过期，记录新键
        if keys.get(key).map_or(true, |expiry| *expiry <= now) {
            keys.insert(key.to_string(), now + self.ttl);
            true
        } else {
            false  // 重复请求
        }
    }
}
```

### 3.4 网关服务 (GatewayServer)

```rust
pub struct GatewayServer {
    config: Config,
    channels: HashMap<String, Arc<dyn Channel>>,
    provider: Arc<dyn Provider>,
    memory: Arc<dyn Memory>,
    policy: Arc<SecurityPolicy>,
    pairing_guard: PairingGuard,
    rate_limiter: Arc<GatewayRateLimiter>,
    idempotency: Arc<IdempotencyStore>,
}

impl GatewayServer {
    pub fn new(config: Config) -> Result<Self> {
        // 初始化各个组件
        let channels = load_channels(&config)?;
        let provider = create_provider(&config)?;
        let memory = create_memory(&config)?;
        let policy = create_security_policy(&config)?;
        let pairing_guard = PairingGuard::new(
            config.gateway.require_pairing,
            &config.security.allowed_domains,
        );
        
        let rate_limiter = Arc::new(GatewayRateLimiter::new(
            config.gateway.pair_rate_limit,
            config.gateway.webhook_rate_limit,
            config.gateway.rate_limit_max_keys,
        ));
        
        let idempotency = Arc::new(IdempotencyStore::new(
            Duration::from_secs(config.gateway.idempotency_ttl_secs),
            config.gateway.idempotency_max_keys,
        ));

        Ok(Self {
            config,
            channels,
            provider,
            memory,
            policy,
            pairing_guard,
            rate_limiter,
            idempotency,
        })
    }

    /// 构建 Axum 路由
    pub fn build_router(&self) -> Router {
        Router::new()
            // 配对端点
            .route("/pair", post(handle_pair))
            
            // Webhook 端点 (按渠道分类)
            .route("/webhook/telegram", post(handle_telegram_webhook))
            .route("/webhook/discord", post(handle_discord_webhook))
            .route("/webhook/slack", post(handle_slack_webhook))
            .route("/webhook/whatsapp", post(handle_whatsapp_webhook))
            .route("/webhook/wati", post(handle_wati_webhook))
            .route("/webhook/lark", post(handle_lark_webhook))
            .route("/webhook/feishu", post(handle_feishu_webhook))
            .route("/webhook/nextcloud_talk", post(handle_nextcloud_talk_webhook))
            
            // API 端点
            .route("/api/v1/status", get(api::get_status))
            .route("/api/v1/channels", get(api::list_channels))
            .route("/api/v1/memory", get(api::query_memory))
            .route("/api/v1/tools", get(api::list_tools))
            
            // WebSocket 端点
            .route("/ws", get(ws::handle_ws))
            
            // SSE 端点
            .route("/sse", get(sse::handle_sse))
            
            // 静态文件
            .nest_service("/", static_files::serve_static())
            
            // 全局中间件
            .layer(RequestBodyLimitLayer::new(MAX_BODY_SIZE))
            .layer(TimeoutLayer::new(Duration::from_secs(REQUEST_TIMEOUT_SECS)))
            .layer(CorsLayer::permissive())
            .with_state(Arc::new(self.clone()))
    }

    /// 启动服务器
    pub async fn run(self, addr: SocketAddr) -> Result<()> {
        let app = self.build_router();
        
        info!("Starting gateway server on {}", addr);
        
        axum::Server::bind(&addr)
            .serve(app.into_make_service_with_connect_info::<SocketAddr>())
            .await?;
        
        Ok(())
    }
}
```

### 3.5 Webhook 处理流程

#### 3.5.1 Telegram Webhook

```rust
async fn handle_telegram_webhook(
    State(state): State<Arc<GatewayState>>,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
    Json(payload): Json<TelegramWebhookPayload>,
) -> Result<Json<serde_json::Value>, StatusCode> {
    // 1. 速率限制检查
    let ip_key = format!("telegram:{}", addr.ip());
    if !state.rate_limiter.allow_webhook(&ip_key) {
        return Err(StatusCode::TOO_MANY_REQUESTS);
    }

    // 2. 验证来源 (可选)
    if let Some(expected_ip) = state.config.telegram.webhook_expected_ip {
        if addr.ip().to_string() != expected_ip {
            audit_log_violation("Telegram webhook from unexpected IP", &addr);
            return Err(StatusCode::FORBIDDEN);
        }
    }

    // 3. 幂等性检查
    let idem_key = format!("telegram:{}", payload.update_id);
    if !state.idempotency.record_if_new(&idem_key) {
        return Ok(Json(serde_json::json!({"status": "duplicate"})));
    }

    // 4. 转换为内部消息格式
    let msg = match telegram_to_channel_message(payload, &state.config) {
        Ok(m) => m,
        Err(e) => {
            error!("Failed to convert Telegram message: {}", e);
            return Err(StatusCode::BAD_REQUEST);
        }
    };

    // 5. 保存到内存 (用于 Agent 处理)
    let memory_key = format!("telegram_{}_{}", msg.sender, msg.id);
    state.memory.set(&memory_key, &msg).await.map_err(|_| {
        StatusCode::INTERNAL_SERVER_ERROR
    })?;

    // 6. 触发 Agent 处理 (异步)
    let agent_tx = state.agent_tx.clone();
    tokio::spawn(async move {
        if let Err(e) = agent_tx.send(AgentEvent::ChannelMessage(msg)).await {
            error!("Failed to send to agent: {}", e);
        }
    });

    Ok(Json(serde_json::json!({"status": "ok"})))
}
```

#### 3.5.2 WhatsApp Webhook

```rust
async fn handle_whatsapp_webhook(
    State(state): State<Arc<GatewayState>>,
    Query(params): Query<WhatsAppQueryParams>,
    Json(payload): Json<WhatsAppWebhookPayload>,
) -> Result<Json<serde_json::Value>, StatusCode> {
    // 1. 验证 Hub Challenge (Facebook 验证)
    if let Some(mode) = params.mode {
        if mode == "subscribe" {
            if params.verify_token.as_deref() == Some(&state.config.whatsapp.verify_token) {
                return Ok(Json(serde_json::json!(params.challenge.unwrap_or_default())));
            } else {
                return Err(StatusCode::FORBIDDEN);
            }
        }
    }

    // 2. 验证签名
    if let Some(signature) = params.signature {
        if !verify_whatsapp_signature(&payload, &signature, &state.config) {
            return Err(StatusCode::UNAUTHORIZED);
        }
    }

    // 3. 处理消息
    for entry in payload.entry {
        for change in entry.changes {
            if let Some(value) = change.value {
                if let Some(messages) = value.messages {
                    for msg in messages {
                        let channel_msg = whatsapp_to_channel_message(msg, &change);
                        
                        // 保存到内存
                        let memory_key = whatsapp_memory_key(&channel_msg);
                        state.memory.set(&memory_key, &channel_msg).await?;
                        
                        // 触发 Agent
                        let agent_tx = state.agent_tx.clone();
                        tokio::spawn(async move {
                            let _ = agent_tx.send(AgentEvent::ChannelMessage(channel_msg)).await;
                        });
                    }
                }
            }
        }
    }

    Ok(Json(serde_json::json!({"status": "ok"})))
}
```

### 3.6 配对端点

```rust
async fn handle_pair(
    State(state): State<Arc<GatewayState>>,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
    Json(payload): Json<PairRequest>,
) -> Result<Json<PairResponse>, StatusCode> {
    // 1. 速率限制 (更严格的配对限制)
    let ip_key = format!("pair:{}", addr.ip());
    if !state.rate_limiter.allow_pair(&ip_key) {
        audit_log_attempt("Pair request rate limited", &addr);
        return Err(StatusCode::TOO_MANY_REQUESTS);
    }

    // 2. 检查是否需要配对
    if !state.pairing_guard.require_pairing() {
        return Ok(Json(PairResponse {
            success: true,
            message: "Pairing not required".to_string(),
        }));
    }

    // 3. 验证配对码
    match state.pairing_guard.verify_pairing_code(&payload.code) {
        Ok(device_id) => {
            // 记录可信设备
            state.trusted_devices.lock().insert(device_id);
            
            audit_log_success("Device paired successfully", &addr);
            
            Ok(Json(PairResponse {
                success: true,
                message: "Device paired successfully".to_string(),
            }))
        }
        Err(e) => {
            audit_log_failure("Invalid pairing code", &addr, &e);
            
            Err(StatusCode::UNAUTHORIZED)
        }
    }
}
```

### 3.7 API 端点 (api.rs)

#### 3.7.1 状态查询

```rust
pub async fn get_status(
    State(state): State<Arc<GatewayState>>,
) -> Json<StatusResponse> {
    let channels_status: Vec<ChannelStatus> = state.channels.iter().map(|(id, ch)| {
        ChannelStatus {
            id: id.clone(),
            name: ch.name().to_string(),
            connected: ch.is_connected(),
            last_seen: ch.last_seen(),
        }
    }).collect();

    Json(StatusResponse {
        status: "healthy".to_string(),
        uptime_secs: state.start_time.elapsed().as_secs(),
        channels: channels_status,
        active_sessions: state.session_count.load(Ordering::Relaxed),
        total_requests: state.request_count.load(Ordering::Relaxed),
    })
}
```

#### 3.7.2 记忆查询

```rust
pub async fn query_memory(
    State(state): State<Arc<GatewayState>>,
    Query(params): Query<MemoryQueryParams>,
) -> Result<Json<Vec<MemoryEntry>>, StatusCode> {
    let entries = state.memory.search(
        &params.query,
        params.limit.unwrap_or(10),
        params.category.as_ref().map(|s| s.as_str()),
    ).await.map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(Json(entries))
}
```

### 3.8 WebSocket 处理 (ws.rs)

```rust
pub async fn handle_ws(
    State(state): State<Arc<GatewayState>>,
    ws: WebSocketUpgrade,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
) -> impl IntoResponse {
    ws.on_upgrade(move |socket| async move {
        handle_ws_connection(socket, state, addr).await;
    })
}

async fn handle_ws_connection(
    socket: WebSocket,
    state: Arc<GatewayState>,
    addr: SocketAddr,
) {
    let (mut tx, mut rx) = socket.split();
    
    // 发送通道：接收 Agent 事件并转发给客户端
    let mut event_rx = state.event_bus.subscribe();
    
    let tx_task = tokio::spawn(async move {
        while let Ok(event) = event_rx.recv().await {
            let msg = serde_json::to_string(&event).unwrap();
            if tx.send(Message::Text(msg)).await.is_err() {
                break;  // 客户端断开
            }
        }
    });

    // 接收通道：处理客户端消息
    let rx_task = tokio::spawn(async move {
        while let Some(msg) = rx.next().await {
            match msg {
                Ok(Message::Text(text)) => {
                    // 处理客户端消息
                    handle_ws_message(&state, &text, addr).await;
                }
                Ok(Message::Close(_)) => {
                    break;
                }
                _ => {}
            }
        }
    });

    // 等待任一任务结束
    tokio::select! {
        _ = tx_task => {},
        _ = rx_task => {},
    }
}
```

### 3.9 SSE 处理 (sse.rs)

```rust
pub async fn handle_sse(
    State(state): State<Arc<GatewayState>>,
) -> impl IntoResponse {
    let stream = async_stream::stream! {
        let mut event_rx = state.event_bus.subscribe();
        
        loop {
            tokio::select! {
                event = event_rx.recv() => {
                    if let Ok(event) = event {
                        yield Ok(Event::default()
                            .event("message")
                            .data(serde_json::to_string(&event).unwrap()));
                    }
                }
                _ = tokio::time::sleep(Duration::from_secs(30)) => {
                    // 心跳
                    yield Ok(Event::default()
                        .event("ping")
                        .data("{\"type\":\"ping\"}"));
                }
            }
        }
    };

    Sse::new(stream).keep_alive(
        KeepAlive::new()
            .interval(Duration::from_secs(15))
            .text("keep-alive-text"),
    )
}
```

## 4. 安全机制

### 4.1 请求体限制

```rust
// 最大 64KB 请求体
pub const MAX_BODY_SIZE: usize = 65_536;

// 应用限制
.layer(RequestBodyLimitLayer::new(MAX_BODY_SIZE))
```

### 4.2 请求超时

```rust
// 30 秒超时
pub const REQUEST_TIMEOUT_SECS: u64 = 30;

// 应用超时
.layer(TimeoutLayer::new(Duration::from_secs(REQUEST_TIMEOUT_SECS)))
```

### 4.3 来源验证

```rust
// 验证 Telegram webhook 来源 IP
if let Some(expected_ip) = config.telegram.webhook_expected_ip {
    if addr.ip().to_string() != expected_ip {
        return Err(StatusCode::FORBIDDEN);
    }
}

// 验证 WhatsApp 签名
fn verify_whatsapp_signature(
    payload: &WhatsAppWebhookPayload,
    signature: &str,
    config: &Config,
) -> bool {
    use hmac::{Hmac, Mac};
    use sha2::Sha256;

    type HmacSha256 = Hmac<Sha256>;

    let mut mac = HmacSha256::new_from_slice(&config.whatsapp.app_secret)
        .expect("HMAC can take key of any size");
    mac.update(payload.raw_body.as_bytes());
    
    mac.verify_slice(signature.as_bytes()).is_ok()
}
```

### 4.4 审计日志

```rust
fn audit_log_violation(reason: &str, addr: &SocketAddr) {
    let event = AuditEvent {
        event_type: AuditEventType::SecurityViolation {
            reason: reason.to_string(),
            source: addr.to_string(),
        },
        timestamp: Utc::now(),
        severity: Severity::Warning,
        ..Default::default()
    };
    
    AUDIT_LOGGER.log(event);
}
```

## 5. 配置选项

```toml
[gateway]
host = "0.0.0.0"
port = 8080
require_pairing = true
pair_rate_limit = 5       # 每分钟 5 次配对请求
webhook_rate_limit = 100  # 每分钟 100 次 webhook
rate_limit_max_keys = 10000
idempotency_ttl_secs = 3600
idempotency_max_keys = 10000

[gateway.ssl]
enabled = false
cert_path = "/path/to/cert.pem"
key_path = "/path/to/key.pem"

[telegram]
webhook_enabled = true
webhook_port = 8443
webhook_url = "https://your-domain.com/webhook/telegram"
webhook_expected_ip = "149.154.167.197"  # Telegram webhook IPs

[whatsapp]
webhook_verify_token = "your_verify_token"
app_secret = "your_app_secret"
verify_signatures = true

[discord]
webhook_public_key = "your_discord_public_key"  # 用于验证 Discord 签名
```

## 6. 错误处理

### 6.1 错误类型

```rust
pub enum GatewayError {
    RateLimitExceeded,
    IdempotencyViolation,
    AuthenticationFailed(String),
    SignatureMismatch,
    InvalidPayload(String),
    ChannelNotFound(String),
    InternalError(anyhow::Error),
}

impl IntoResponse for GatewayError {
    fn into_response(self) -> Response {
        match self {
            Self::RateLimitExceeded => {
                (StatusCode::TOO_MANY_REQUESTS, "Rate limit exceeded").into_response()
            }
            Self::AuthenticationFailed(msg) => {
                (StatusCode::UNAUTHORIZED, msg).into_response()
            }
            Self::InvalidPayload(msg) => {
                (StatusCode::BAD_REQUEST, msg).into_response()
            }
            _ => (StatusCode::INTERNAL_SERVER_ERROR, "Internal error").into_response(),
        }
    }
}
```

## 7. 性能优化

### 7.1 连接池

```rust
// 复用 HTTP 客户端
lazy_static! {
    static ref HTTP_CLIENT: reqwest::Client = reqwest::ClientBuilder::new()
        .timeout(Duration::from_secs(30))
        .pool_max_idle_per_host(10)
        .build()
        .unwrap();
}
```

### 7.2 异步处理

所有 webhook 处理都是异步的，Agent 触发通过 `tokio::spawn` 后台执行，不阻塞响应。

### 7.3 内存管理

- 限流器和幂等性存储都有最大键数限制
- 定期清理过期条目
- 使用 `Arc<Mutex>` 最小化锁竞争

## 8. 测试策略

### 8.1 单元测试

```rust
#[test]
fn test_rate_limiter_allows_under_limit() {
    let limiter = SlidingWindowRateLimiter::new(
        10,  // 10 requests per window
        Duration::from_secs(60),
        1000,
    );

    for _ in 0..10 {
        assert!(limiter.allow("test_key"));
    }

    assert!(!limiter.allow("test_key"));  // Should be blocked
}
```

### 8.2 集成测试

使用 `axum::TestClient` 测试完整 HTTP 请求流程。

## 9. 监控指标

### 9.1 关键指标

- 请求总数和成功率
- 各渠道 webhook 数量
- 平均响应时间 (P50/P95/P99)
- 限流触发次数
- WebSocket 活跃连接数
- SSE 客户端数

### 9.2 日志记录

```rust
// 结构化日志
info!(
    target = "gateway",
    channel = "telegram",
    update_id = payload.update_id,
    duration_ms = start.elapsed().as_millis(),
    "Processed Telegram webhook"
);
```

## 10. 扩展指南

### 10.1 添加新渠道 Webhook

1. 在 `mod.rs` 添加路由
2. 创建转换函数 (`xxx_to_channel_message`)
3. 实现签名验证 (如果需要)
4. 更新配置 schema

### 10.2 示例：添加 LINE Webhook

```rust
// 1. 添加路由
.route("/webhook/line", post(handle_line_webhook))

// 2. 实现处理器
async fn handle_line_webhook(
    State(state): State<Arc<GatewayState>>,
    Json(payload): Json<LineWebhookPayload>,
) -> Result<Json<serde_json::Value>, StatusCode> {
    // 验证签名
    if !verify_line_signature(&payload, &state.config) {
        return Err(StatusCode::UNAUTHORIZED);
    }

    // 转换消息
    for event in payload.events {
        let msg = line_to_channel_message(event, &state.config);
        
        // 保存并触发 Agent
        let memory_key = format!("line_{}_{}", msg.sender, msg.id);
        state.memory.set(&memory_key, &msg).await?;
        
        let agent_tx = state.agent_tx.clone();
        tokio::spawn(async move {
            let _ = agent_tx.send(AgentEvent::ChannelMessage(msg)).await;
        });
    }

    Ok(Json(serde_json::json!({"status": "ok"})))
}
```
