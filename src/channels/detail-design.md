# Channels 模块详细设计

## 1. 模块概述

Channels 模块实现了 ZeroClaw 与各种通信平台的集成层，负责消息的接收和发送。采用统一的 Trait 抽象，支持多种 messaging platform 的即插即用。

## 2. 架构设计

### 2.1 模块结构

```
src/channels/
├── mod.rs           # 模块导出和工厂注册 (238KB)
├── traits.rs        # Channel Trait 定义 (7.5KB)
├── telegram.rs      # Telegram Bot API (161KB - 最复杂)
├── discord.rs       # Discord API (50.7KB)
├── slack.rs         # Slack API (20.2KB)
├── whatsapp.rs      # WhatsApp Business API (37.3KB)
├── matrix.rs        # Matrix Protocol (36.0KB)
├── signal.rs        # Signal Protocol (28.7KB)
├── email_channel.rs # Email SMTP/IMAP (32.2KB)
├── imessage.rs      # iMessage AppleScript (32.8KB)
├── irc.rs           # IRC Protocol (35.0KB)
├── mqtt.rs          # MQTT IoT Protocol (8.5KB)
├── lark.rs          # 飞书/Lark (78.2KB)
├── dingtalk.rs      # 钉钉 (13.0KB)
├── qq.rs            # QQ (21.9KB)
├── mattermost.rs    # Mattermost (28.6KB)
├── nextcloud_talk.rs# Nextcloud Talk (15.2KB)
├── nostr.rs         # NoStr 协议 (14.3KB)
├── wati.rs          # Wati WhatsApp API (15.4KB)
├── clawdtalk.rs     # ClawD Talk (12.9KB)
├── linq.rs          # LinQ (25.1KB)
├── transcription.rs # 语音转文本 (6.7KB)
└── cli.rs           # CLI 通道 (3.7KB)
```

### 2.2 设计模式

**策略模式**: 每个 Channel 实现是独立的策略
**工厂模式**: `mod.rs` 中的工厂函数根据配置创建实例
**适配器模式**: 将各平台 API 适配到统一 Channel trait

## 3. 核心 Trait 定义 (traits.rs)

### 3.1 Channel Trait

```rust
#[async_trait]
pub trait Channel: Send + Sync {
    /// 人类可读的频道名称
    fn name(&self) -> &str;

    /// 发送消息
    async fn send(&self, message: &SendMessage) -> anyhow::Result<()>;

    /// 监听消息 (长运行任务)
    async fn listen(&self, tx: tokio::sync::mpsc::Sender<ChannelMessage>) 
        -> anyhow::Result<()>;

    /// 健康检查
    async fn health_check(&self) -> bool { true }

    /// 开始输入指示 (typing indicator)
    async fn start_typing(&self, recipient: &str) -> anyhow::Result<()> { Ok(()) }

    /// 停止输入指示
    async fn stop_typing(&self, recipient: &str) -> anyhow::Result<()> { Ok(()) }

    // ... 更多可选方法 (draft updates, reactions 等)
}
```

### 3.2 数据结构

#### ChannelMessage (接收消息)

```rust
pub struct ChannelMessage {
    pub id: String,              // 平台消息 ID
    pub sender: String,          // 发送者标识
    pub reply_target: String,    // 回复目标 (用于溯源)
    pub content: String,         // 消息内容
    pub channel: String,         // 频道名称
    pub timestamp: u64,          // Unix 时间戳
    pub thread_ts: Option<String>, // 线程 ID (支持 threaded replies)
}
```

#### SendMessage (发送消息)

```rust
pub struct SendMessage {
    pub content: String,
    pub recipient: String,
    pub subject: Option<String>,     // 邮件等需要主题的平台
    pub thread_ts: Option<String>,   // 回复到特定线程
}
```

### 3.3 Builder 模式

```rust
impl SendMessage {
    pub fn new(content: impl Into<String>, recipient: impl Into<String>) -> Self {
        Self {
            content: content.into(),
            recipient: recipient.into(),
            subject: None,
            thread_ts: None,
        }
    }

    pub fn with_subject(
        content: impl Into<String>,
        recipient: impl Into<String>,
        subject: impl Into<String>,
    ) -> Self {
        Self {
            content: content.into(),
            recipient: recipient.into(),
            subject: Some(subject.into()),
            thread_ts: None,
        }
    }

    pub fn in_thread(mut self, thread_ts: Option<String>) -> Self {
        self.thread_ts = thread_ts;
        self
    }
}
```

## 4. 典型实现分析：Telegram (telegram.rs - 161KB)

### 4.1 结构体定义

```rust
pub struct TelegramChannel {
    config: TelegramConfig,
    bot_token: Secret<String>,
    client: reqwest::Client,
    last_update_id: Arc<AtomicU64>,  // 断点续传
    allowlist: Arc<RwLock<HashSet<String>>>,  // 白名单控制
}

struct TelegramConfig {
    name: String,
    bot_token: String,
    allowed_users: Vec<String>,  // 用户名或用户 ID
    webhook_url: Option<String>, // webhook 模式 URL
}
```

### 4.2 核心方法实现

#### 4.2.1 send 方法

```rust
async fn send(&self, message: &SendMessage) -> Result<()> {
    let api_url = format!(
        "https://api.telegram.org/bot{}/sendMessage",
        self.bot_token.expose()
    );

    let payload = serde_json::json!({
        "chat_id": message.recipient,
        "text": message.content,
        "parse_mode": "Markdown",
        "reply_to_message_id": message.thread_ts
            .as_ref()
            .and_then(|id| id.parse::<i64>().ok())
    });

    let response = self.client
        .post(&api_url)
        .json(&payload)
        .send()
        .await?;

    if !response.status().is_success() {
        bail!("Telegram API error: {}", response.text().await?);
    }

    Ok(())
}
```

**设计要点**:
- 使用 `Secret` 封装敏感 token
- 支持 Markdown 格式化
- 支持回复到特定消息 (thread_ts)

#### 4.2.2 listen 方法 (长轮询)

```rust
async fn listen(&self, tx: mpsc::Sender<ChannelMessage>) -> Result<()> {
    let api_url = format!(
        "https://api.telegram.org/bot{}/getUpdates",
        self.bot_token.expose()
    );

    loop {
        let params = serde_json::json!({
            "offset": self.last_update_id.load(Ordering::SeqCst),
            "timeout": 30,  // long polling timeout
        });

        let response = self.client
            .post(&api_url)
            .json(&params)
            .send()
            .await?;

        let updates: TelegramUpdates = response.json().await?;

        for update in updates.result {
            // 更新 offset 避免重复处理
            self.last_update_id.store(
                update.update_id + 1,
                Ordering::SeqCst
            );

            // 白名单检查
            if let Some(ref message) = update.message {
                let sender = message.from.username
                    .unwrap_or_else(|| message.from.id.to_string());
                
                if !self.is_allowed(&sender) {
                    tracing::warn!("Blocked unauthorized user: {}", sender);
                    continue;
                }

                // 转换为统一的 ChannelMessage
                let channel_msg = ChannelMessage {
                    id: update.update_id.to_string(),
                    sender: sender.clone(),
                    reply_target: message.chat.id.to_string(),
                    content: message.text.clone().unwrap_or_default(),
                    channel: self.config.name.clone(),
                    timestamp: message.date as u64,
                    thread_ts: Some(message.message_id.to_string()),
                };

                // 发送到主循环
                if tx.send(channel_msg).await.is_err() {
                    bail!("Channel receiver dropped");
                }
            }
        }
    }
}
```

**关键特性**:
1. **断点续传**: 通过 `offset` 参数避免消息丢失
2. **长轮询**: 30 秒超时减少空轮询
3. **白名单**: 只响应授权用户
4. **错误恢复**: 网络错误后自动重试

#### 4.2.3 Typing Indicator

```rust
async fn start_typing(&self, recipient: &str) -> Result<()> {
    let api_url = format!(
        "https://api.telegram.org/bot{}/sendChatAction",
        self.bot_token.expose()
    );

    let payload = serde_json::json!({
        "chat_id": recipient,
        "action": "typing"
    });

    // Telegram typing indicator 持续 ~5 秒，需要定期刷新
    let client = self.client.clone();
    let recipient = recipient.to_string();
    let token = self.bot_token.expose().clone();
    
    tokio::spawn(async move {
        let mut interval = tokio::time::interval(Duration::from_secs(4));
        for _ in 0..5 {  // 最多刷新 5 次 (20 秒)
            interval.tick().await;
            let _ = client
                .post(&format!("https://api.telegram.org/bot{token}/sendChatAction"))
                .json(&serde_json::json!({
                    "chat_id": recipient,
                    "action": "typing"
                }))
                .send()
                .await;
        }
    });

    Ok(())
}
```

### 4.3 Webhook 模式支持

除了 long polling，还支持 webhook:

```rust
pub async fn handle_webhook(
    &self,
    payload: &[u8],
    tx: mpsc::Sender<ChannelMessage>
) -> Result<()> {
    let update: Update = serde_json::from_slice(payload)?;
    
    // 处理逻辑与 long polling 相同
    // ...
    
    Ok(())
}
```

## 5. 其他 Channel 实现特点

### 5.1 Discord (discord.rs - 50.7KB)

**特色功能**:
- Gateway WebSocket 连接 (实时性更高)
- Embeds 富文本消息
- Reactions 表情回应
- Threads 线程支持

```rust
// Discord Gateway 连接
use serenity::prelude::*;
use serenity::model::prelude::*;

pub struct DiscordChannel {
    ctx: Client,  // Serenity client
    config: DiscordConfig,
}

// 事件处理器
struct DiscordEventHandler {
    tx: mpsc::Sender<ChannelMessage>,
}

#[serenity::async_trait]
impl EventHandler for DiscordEventHandler {
    async fn message(&self, ctx: Context, msg: Message) {
        if msg.author.bot { return; }  // 忽略机器人
        
        let channel_msg = ChannelMessage {
            id: msg.id.to_string(),
            sender: msg.author.name.clone(),
            reply_target: msg.channel_id.to_string(),
            content: msg.content.clone(),
            channel: "discord".to_string(),
            timestamp: msg.timestamp.timestamp() as u64,
            thread_ts: msg.thread_id.map(|id| id.to_string()),
        };
        
        let _ = self.tx.send(channel_msg).await;
    }
}
```

### 5.2 Slack (slack.rs - 20.2KB)

**特色功能**:
- Block Kit UI 组件
- Thread 回复
- Emoji Reactions
- App Mentions

```rust
// Slack Bolt SDK 集成
use slack_bolt::App;

pub struct SlackChannel {
    app: Arc<App>,
    config: SlackConfig,
}

// Block Kit 消息示例
async fn send(&self, message: &SendMessage) -> Result<()> {
    let blocks = serde_json::json!([{
        "type": "section",
        "text": {
            "type": "mrkdwn",
            "text": message.content
        }
    }]);
    
    // 发送带有 block 的消息
    // ...
}
```

### 5.3 Email (email_channel.rs - 32.2KB)

**双向协议**:
- SMTP 发送邮件
- IMAP 接收邮件

```rust
use lettre::{Transport, smtp::SmtpTransport};
use imap::Client;

pub struct EmailChannel {
    smtp_config: SmtpConfig,
    imap_config: ImapConfig,
}

// IMAP 监听
async fn listen(&self, tx: mpsc::Sender<ChannelMessage>) -> Result<()> {
    let mut client = Client::secure_connect(self.imap_config.server)?;
    client.login(&self.imap_config.user, &self.imap_config.password)?;
    client.select("INBOX")?;
    
    loop {
        let messages = client.search("UNSEEN")?;
        for uid in messages {
            let msg = client.fetch(uid, "BODY[]")?;
            // 解析 MIME 邮件
            let email = parse_email(msg)?;
            
            let channel_msg = ChannelMessage {
                id: uid.to_string(),
                sender: email.from,
                reply_target: email.to,
                content: email.body,
                channel: "email".to_string(),
                timestamp: email.date.timestamp() as u64,
                thread_ts: email.message_id,
            };
            
            let _ = tx.send(channel_msg).await;
        }
        
        tokio::time::sleep(Duration::from_secs(60)).await;  // 每分钟检查
    }
}
```

### 5.4 WhatsApp (whatsapp.rs - 37.3KB)

**Cloud API 集成**:

```rust
pub struct WhatsAppChannel {
    config: WhatsAppConfig,
    access_token: Secret<String>,
    phone_number_id: String,
    client: reqwest::Client,
    webhook_verify_token: String,
}

// Webhook 验证 (Meta 要求)
pub async fn verify_webhook(
    &self,
    mode: &str,
    token: &str,
    challenge: &str
) -> Option<String> {
    if mode == "subscribe" && token == self.webhook_verify_token {
        Some(challenge.to_string())
    } else {
        None
    }
}

// 发送模板消息 (商业 API 限制)
async fn send(&self, message: &SendMessage) -> Result<()> {
    // WhatsApp 商业 API 需要先使用模板消息发起对话
    let payload = serde_json::json!({
        "messaging_product": "whatsapp",
        "recipient_type": "individual",
        "to": message.recipient,
        "type": "template",
        "template": {
            "name": "generic_template",
            "language": {"code": "en"},
            "components": [{
                "type": "body",
                "parameters": [{
                    "type": "text",
                    "text": message.content
                }]
            }]
        }
    });
    
    // 发送请求...
}
```

## 6. 工厂模式实现 (mod.rs)

### 6.1 Channel Factory

```rust
pub fn create_channel(
    channel_type: &str,
    config: &serde_json::Value,
) -> Result<Arc<dyn Channel>> {
    match channel_type.to_lowercase().as_str() {
        "telegram" => {
            let cfg: TelegramConfig = serde_json::from_value(config.clone())?;
            Ok(Arc::new(TelegramChannel::new(cfg)?))
        }
        "discord" => {
            let cfg: DiscordConfig = serde_json::from_value(config.clone())?;
            Ok(Arc::new(DiscordChannel::new(cfg)?))
        }
        "slack" => {
            let cfg: SlackConfig = serde_json::from_value(config.clone())?;
            Ok(Arc::new(SlackChannel::new(cfg)?))
        }
        // ... 其他 channel 类型
        _ => bail!("Unknown channel type: {}", channel_type),
    }
}
```

### 6.2 多 Channel 并发启动

```rust
pub async fn start_channels(
    config: Config,
) -> Result<Vec<tokio::task::JoinHandle<Result<()>>>> {
    let mut handles = Vec::new();
    let (tx, mut rx) = mpsc::channel::<ChannelMessage>(100);

    // 为每个配置的 channel 创建实例
    for (channel_name, channel_config) in config.channels {
        let channel = create_channel(&channel_name, &channel_config)?;
        let tx = tx.clone();
        
        let handle = tokio::spawn(async move {
            tracing::info!("Starting channel: {}", channel_name);
            channel.listen(tx).await
        });
        
        handles.push(handle);
    }

    // 主循环：从所有 channel 接收消息
    let agent_handle = tokio::spawn(async move {
        while let Some(msg) = rx.recv().await {
            // 转发给 Agent 处理
            process_message(msg).await?;
        }
        Ok::<_, anyhow::Error>(())
    });

    handles.push(agent_handle);
    Ok(handles)
}
```

## 7. 高级特性

### 7.1 Draft Updates (渐进式消息更新)

某些平台 (如 Telegram, Discord) 支持编辑已发送消息，用于流式输出:

```rust
pub trait Channel {
    // ... 基础方法
    
    /// 是否支持草稿更新
    fn supports_draft_updates(&self) -> bool { false }

    /// 发送初始草稿
    async fn send_draft(&self, message: &SendMessage) 
        -> Result<Option<String>> { Ok(None) }

    /// 更新草稿内容
    async fn update_draft(
        &self,
        recipient: &str,
        message_id: &str,
        text: &str,
    ) -> Result<()> { Ok(()) }

    /// 完成最终版本
    async fn finalize_draft(
        &self,
        recipient: &str,
        message_id: &str,
        text: &str,
    ) -> Result<()> { Ok(()) }

    /// 取消草稿
    async fn cancel_draft(
        &self,
        recipient: &str,
        message_id: &str,
    ) -> Result<()> { Ok(()) }
}
```

**Telegram 实现示例**:

```rust
async fn update_draft(
    &self,
    recipient: &str,
    message_id: &str,
    text: &str,
) -> Result<()> {
    let api_url = format!(
        "https://api.telegram.org/bot{}/editMessageText",
        self.bot_token.expose()
    );

    let payload = serde_json::json!({
        "chat_id": recipient,
        "message_id": message_id.parse::<i64>()?,
        "text": text,
        "parse_mode": "Markdown"
    });

    self.client.post(&api_url).json(&payload).send().await?;
    Ok(())
}
```

### 7.2 Reactions (表情回应)

```rust
pub trait Channel {
    // 添加表情回应
    async fn add_reaction(
        &self,
        channel_id: &str,
        message_id: &str,
        emoji: &str,
    ) -> Result<()> { Ok(()) }

    // 移除表情回应
    async fn remove_reaction(
        &self,
        channel_id: &str,
        message_id: &str,
        emoji: &str,
    ) -> Result<()> { Ok(()) }
}
```

**Discord 实现**:

```rust
async fn add_reaction(
    &self,
    channel_id: &str,
    message_id: &str,
    emoji: &str,
) -> Result<()> {
    // 解析 message_id (格式："discord_<snowflake>")
    let parts: Vec<&str> = message_id.split('_').collect();
    if parts.len() != 2 || parts[0] != "discord" {
        bail!("Invalid Discord message ID format");
    }
    
    let msg_id = parts[1];
    
    // 调用 Discord API 添加反应
    self.ctx
        .http
        .create_reaction(
            ChannelId::new(channel_id.parse()?),
            MessageId::new(msg_id.parse()?),
            ReactionType::Unicode(emoji.to_string()),
        )?
        .await?;
    
    Ok(())
}
```

### 7.3 语音转文本 (transcription.rs)

```rust
pub async fn transcribe_audio(
    audio_data: &[u8],
    language: Option<&str>,
) -> Result<String> {
    // 使用 Whisper API 或其他 STT 服务
    let form = reqwest::multipart::Form::new()
        .part("file", reqwest::multipart::Part::bytes(audio_data.to_vec()))
        .text("model", "whisper-1");

    if let Some(lang) = language {
        form.text("language", lang.to_string());
    }

    let response = client
        .post("https://api.openai.com/v1/audio/transcriptions")
        .multipart(form)
        .send()
        .await?;

    let result: WhisperResponse = response.json().await?;
    Ok(result.text)
}
```

## 8. 健康检查

每个 Channel 实现健康检查:

```rust
impl Channel for TelegramChannel {
    async fn health_check(&self) -> bool {
        let api_url = format!(
            "https://api.telegram.org/bot{}/getMe",
            self.bot_token.expose()
        );

        match self.client.get(&api_url).send().await {
            Ok(resp) => resp.status().is_success(),
            Err(_) => false,
        }
    }
}
```

**Doctor 命令集成**:

```bash
zeroclaw channel doctor
```

会并行检查所有 configured channels 的健康状态。

## 9. 配置管理

### 9.1 Channel 配置 Schema

```json
{
  "channels": {
    "telegram": {
      "bot_token": "${TELEGRAM_BOT_TOKEN}",
      "name": "my-telegram-bot",
      "allowed_users": ["user1", "user2"]
    },
    "discord": {
      "bot_token": "${DISCORD_BOT_TOKEN}",
      "name": "my-discord-bot",
      "guild_id": "123456789"
    }
  }
}
```

### 9.2 CLI 管理

```bash
# 添加 channel
zeroclaw channel add telegram '{"bot_token":"...","name":"my-bot"}'

# 列出 channels
zeroclaw channel list

# 移除 channel
zeroclaw channel remove my-bot

# 绑定 Telegram 用户到白名单
zeroclaw channel bind-telegram zeroclaw_user
```

## 10. 错误处理与恢复

### 10.1 错误分类

```rust
pub enum ChannelError {
    AuthenticationFailed(String),
    RateLimited { retry_after: Duration },
    NetworkError(reqwest::Error),
    InvalidMessageFormat(String),
    UnauthorizedUser(String),
    PlatformSpecific { platform: String, error: String },
}
```

### 10.2 重试策略

```rust
use tokio_retry::{Retry, ExponentialBackoff};

async fn send_with_retry(
    channel: &dyn Channel,
    message: &SendMessage,
) -> Result<()> {
    let retry_strategy = ExponentialBackoff::from_millis(100)
        .max_delay(Duration::from_secs(30))
        .take(5);

    Retry::spawn(retry_strategy, || {
        channel.send(message)
    }).await
}
```

### 10.3 降级策略

当某个 Channel 失败时:
1. 记录错误日志
2. 尝试通过其他可用 Channel 通知
3. 保存失败消息到队列供后续重试

## 11. 性能优化

### 11.1 连接池

```rust
// 使用共享的 reqwest Client (内部有连接池)
let client = reqwest::Client::builder()
    .pool_max_idle_per_host(10)
    .timeout(Duration::from_secs(30))
    .build()?;
```

### 11.2 消息批处理

某些平台支持批量发送:

```rust
// Discord 可以一次发送多个 embed
async fn send_batch(
    &self,
    messages: &[SendMessage],
) -> Result<()> {
    // 合并消息以减少 API 调用
    // ...
}
```

### 11.3 异步并发

所有 Channel 操作都是异步的，支持高并发场景。

## 12. 安全考虑

### 12.1 Token 管理

```rust
// 使用 Secret 封装敏感信息
use secrecy::Secret;

pub struct TelegramChannel {
    bot_token: Secret<String>,  // 自动擦除内存
}
```

### 12.2 白名单机制

```rust
fn is_allowed(&self, sender: &str) -> bool {
    self.allowlist.read().contains(sender)
}
```

### 12.3 输入验证

- 验证 webhook 签名 (防止伪造)
- 过滤恶意内容
- 限制消息长度

## 13. 测试策略

### 13.1 Mock Channel

```rust
struct MockChannel {
    sent_messages: Arc<Mutex<Vec<SendMessage>>>,
}

#[async_trait]
impl Channel for MockChannel {
    async fn send(&self, message: &SendMessage) -> Result<()> {
        self.sent_messages.lock().push(message.clone());
        Ok(())
    }
    
    // ... 其他方法
}
```

### 13.2 集成测试

```rust
#[tokio::test]
async fn test_telegram_channel_send() {
    let channel = create_test_telegram_channel().await;
    let msg = SendMessage::new("Hello", "test_user");
    
    assert!(channel.send(&msg).await.is_ok());
}
```

## 14. 扩展指南

### 14.1 添加新 Channel

1. 创建新文件 `src/channels/mychannel.rs`
2. 实现 `Channel` trait
3. 在 `mod.rs` 中注册到 factory
4. 添加配置 schema
5. 编写测试

### 14.2 最小实现示例

```rust
use crate::channels::traits::{Channel, SendMessage, ChannelMessage};

pub struct MyChannel {
    config: MyConfig,
}

#[async_trait]
impl Channel for MyChannel {
    fn name(&self) -> &str { "mychannel" }

    async fn send(&self, message: &SendMessage) -> Result<()> {
        // 实现发送逻辑
        Ok(())
    }

    async fn listen(&self, tx: mpsc::Sender<ChannelMessage>) -> Result<()> {
        // 实现监听逻辑
        loop {
            // 接收消息并发送
            let msg = receive_message().await?;
            let channel_msg = ChannelMessage { /* ... */ };
            tx.send(channel_msg).await?;
        }
    }
}
```

## 15. 监控指标

### 15.1 关键指标

- 消息发送成功率
- 消息接收延迟
- API 调用频率
- 错误分布统计

### 15.2 日志记录

```rust
tracing::info!(
    channel = %self.name(),
    recipient = %message.recipient,
    "Message sent successfully"
);

tracing::error!(
    channel = %self.name(),
    error = %e,
    "Failed to send message"
);
```
