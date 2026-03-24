# Config 模块详细设计

## 1. 模块概述

Config 模块负责 ZeroClaw 的配置管理，包括配置加载、验证、合并和运行时更新。提供类型安全的配置访问接口和 Schema 生成能力。

## 2. 架构设计

### 2.1 模块结构

src/config/
- mod.rs          - 模块导出和配置加载 (3.5KB)
- schema.rs       - 配置 Schema 定义 (259.4KB - 最核心)
- traits.rs       - ChannelConfig Trait (0.4KB)

### 2.2 配置层次结构

```
ZeroClaw Configuration
├── 基础配置 (Top-level)
│   ├── api_key / api_url
│   ├── default_provider / default_model
│   └── model_providers (命名配置集)
│
├── 子系统配置
│   ├── autonomy (自主性策略)
│   ├── security (安全策略)
│   ├── runtime (运行时适配器)
│   ├── reliability (可靠性设置)
│   ├── scheduler (任务调度器)
│   ├── agent (Agent 编排)
│   └── skills (技能管理)
│
├── 集成配置
│   ├── channels_config (通信渠道)
│   ├── memory (记忆后端)
│   ├── storage (持久化存储)
│   ├── tunnel (隧道服务)
│   ├── gateway (网关服务器)
│   └── proxy (代理配置)
│
├── 工具配置
│   ├── composio (OAuth 工具)
│   ├── browser (浏览器自动化)
│   ├── http_request (HTTP 请求)
│   └── web_fetch / web_search
│
└── 高级配置
    ├── observability (可观测性)
    ├── heartbeat (心跳检测)
    ├── cron (定时任务)
    ├── cost (成本跟踪)
    └── identity (身份格式)
```

### 2.3 设计模式

- **构建器模式**: ConfigBuilder 逐步构建配置
- **策略模式**: 多种配置源 (文件/环境变量/CLI)
- **单例模式**: RUNTIME_PROXY_CONFIG 全局唯一
- **工厂模式**: 从配置创建各组件实例

## 3. 核心数据结构详解

### 3.1 顶层配置 (Config)

```rust
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct Config {
    // === 计算字段 (不序列化) ===
    #[serde(skip)]
    pub workspace_dir: PathBuf,      // 工作目录
    
    #[serde(skip)]
    pub config_path: PathBuf,        // 配置文件路径

    // === 基础配置 ===
    /// API Key (被 ZEROCLAW_API_KEY 或 API_KEY 覆盖)
    pub api_key: Option<String>,
    
    /// API Base URL 覆盖 (如 "http://10.0.0.1:11434")
    pub api_url: Option<String>,
    
    /// 默认 Provider ID (如 "openrouter", "ollama")
    #[serde(alias = "model_provider")]
    pub default_provider: Option<String>,
    
    /// 默认模型 (如 "anthropic/claude-sonnet-4-6")
    #[serde(alias = "model")]
    pub default_model: Option<String>,
    
    /// 命名 Provider 配置集
    #[serde(default)]
    pub model_providers: HashMap<String, ModelProviderConfig>,
    
    /// 默认温度 (0.0-2.0), 默认 0.7
    pub default_temperature: f64,

    // === 子系统配置 ===
    #[serde(default)]
    pub observability: ObservabilityConfig,
    
    #[serde(default)]
    pub autonomy: AutonomyConfig,
    
    #[serde(default)]
    pub security: SecurityConfig,
    
    #[serde(default)]
    pub runtime: RuntimeConfig,
    
    #[serde(default)]
    pub reliability: ReliabilityConfig,
    
    #[serde(default)]
    pub scheduler: SchedulerConfig,
    
    #[serde(default)]
    pub agent: AgentConfig,
    
    #[serde(default)]
    pub skills: SkillsConfig,

    // === 路由配置 ===
    #[serde(default)]
    pub model_routes: Vec<ModelRouteConfig>,
    
    #[serde(default)]
    pub embedding_routes: Vec<EmbeddingRouteConfig>,
    
    #[serde(default)]
    pub query_classification: QueryClassificationConfig,

    // === 集成配置 ===
    #[serde(default)]
    pub heartbeat: HeartbeatConfig,
    
    #[serde(default)]
    pub cron: CronConfig,
    
    #[serde(default)]
    pub channels_config: ChannelsConfig,
    
    #[serde(default)]
    pub memory: MemoryConfig,
    
    #[serde(default)]
    pub storage: StorageConfig,
    
    #[serde(default)]
    pub tunnel: TunnelConfig,
    
    #[serde(default)]
    pub gateway: GatewayConfig,
    
    #[serde(default)]
    pub composio: ComposioConfig,
    
    #[serde(default)]
    pub secrets: SecretsConfig,
    
    #[serde(default)]
    pub browser: BrowserConfig,
    
    #[serde(default)]
    pub http_request: HttpRequestConfig,
    
    #[serde(default)]
    pub multimodal: MultimodalConfig,
    
    #[serde(default)]
    pub web_fetch: WebFetchConfig,
    
    #[serde(default)]
    pub web_search: WebSearchConfig,
    
    #[serde(default)]
    pub proxy: ProxyConfig,
    
    #[serde(default)]
    pub identity: IdentityConfig,
    
    #[serde(default)]
    pub cost: CostConfig,
}
```

### 3.2 Provider 配置

```rust
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct ModelProviderConfig {
    /// Provider 类型标识符 (如 "openai", "anthropic")
    pub r#type: String,
    
    /// API Key (可被环境变量覆盖)
    pub api_key: Option<String>,
    
    /// API Base URL 覆盖
    pub api_url: Option<String>,
    
    /// 默认模型名称
    pub model: Option<String>,
    
    /// 模型温度 (0.0-2.0)
    pub temperature: Option<f64>,
    
    /// 最大 token 数
    pub max_tokens: Option<u32>,
    
    /// 是否启用流式响应
    pub stream: Option<bool>,
    
    /// 超时设置 (秒)
    pub timeout_secs: Option<u64>,
    
    /// 重试次数
    pub retries: Option<u32>,
    
    /// 提供者特定配置 (JSON)
    #[serde(default)]
    pub extra: serde_json::Value,
}
```

### 3.3 安全配置

```rust
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct SecurityConfig {
    /// 自主性级别 (manual/supervised/autonomous)
    #[serde(default = "default_autonomy_level")]
    pub autonomy: AutonomyLevel,
    
    /// 限制在工作目录内
    #[serde(default = "default_true")]
    pub workspace_only: bool,
    
    /// 允许的根目录列表
    #[serde(default)]
    pub allowed_roots: Vec<PathBuf>,
    
    /// 允许的命令白名单
    #[serde(default)]
    pub allowed_commands: Vec<String>,
    
    /// 每小时最大操作数
    #[serde(default = "default_max_actions_per_hour")]
    pub max_actions_per_hour: u32,
    
    /// 每日最大成本 (美分)
    #[serde(default = "default_max_cost_per_day_cents")]
    pub max_cost_per_day_cents: u32,
    
    /// 是否需要设备配对
    #[serde(default = "default_false")]
    pub require_pairing: bool,
    
    /// 允许的域名列表
    #[serde(default)]
    pub allowed_domains: Vec<String>,
    
    /// 阻止的域名列表
    #[serde(default)]
    pub blocked_domains: Vec<String>,
}

fn default_autonomy_level() -> AutonomyLevel {
    AutonomyLevel::Supervised
}

fn default_max_actions_per_hour() -> u32 {
    100
}

fn default_max_cost_per_day_cents() -> u32 {
    1000  // $10/day
}
```

### 3.4 渠道配置

```rust
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct ChannelsConfig {
    /// Telegram 配置
    #[serde(default)]
    pub telegram: TelegramConfig,
    
    /// Discord 配置
    #[serde(default)]
    pub discord: DiscordConfig,
    
    /// Slack 配置
    #[serde(default)]
    pub slack: SlackConfig,
    
    /// WhatsApp 配置
    #[serde(default)]
    pub whatsapp: WhatsAppConfig,
    
    /// Wati 配置
    #[serde(default)]
    pub wati: WatiConfig,
    
    /// Lark/飞书配置
    #[serde(default)]
    pub lark: LarkConfig,
    
    /// Feishu 配置
    #[serde(default)]
    pub feishu: FeishuConfig,
    
    /// Nextcloud Talk 配置
    #[serde(default)]
    pub nextcloud_talk: NextcloudTalkConfig,
    
    /// Matrix 配置
    #[serde(default)]
    pub matrix: MatrixConfig,
    
    /// Mattermost 配置
    #[serde(default)]
    pub mattermost: MattermostConfig,
    
    /// Signal 配置
    #[serde(default)]
    pub signal: SignalConfig,
    
    /// QQ 配置
    #[serde(default)]
    pub qq: QQConfig,
    
    /// Dingtalk 配置
    #[serde(default)]
    pub dingtalk: DingtalkConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct TelegramConfig {
    /// Bot Token
    pub bot_token: Option<String>,
    
    /// 启用的聊天 IDs
    #[serde(default)]
    pub allowed_chat_ids: Vec<i64>,
    
    /// Webhook 启用
    #[serde(default)]
    pub webhook_enabled: bool,
    
    /// Webhook 端口
    pub webhook_port: Option<u16>,
    
    /// Webhook URL
    pub webhook_url: Option<String>,
    
    /// 期望的来源 IP (可选)
    pub webhook_expected_ip: Option<String>,
    
    /// 管理员用户 IDs
    #[serde(default)]
    pub admin_user_ids: Vec<i64>,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct DiscordConfig {
    /// Bot Token
    pub bot_token: Option<String>,
    
    /// 允许的 Guild IDs
    #[serde(default)]
    pub allowed_guilds: Vec<String>,
    
    /// 监听所有频道 (默认 false)
    #[serde(default = "default_false")]
    pub listen_all_channels: bool,
    
    /// 允许的频道 IDs
    #[serde(default)]
    pub allowed_channels: Vec<String>,
    
    /// 管理员角色 IDs
    #[serde(default)]
    pub admin_role_ids: Vec<String>,
}
```

### 3.5 Memory 配置

```rust
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct MemoryConfig {
    /// 后端类型 (sqlite/markdown/lucid/postgres/qdrant)
    #[serde(default = "default_memory_backend")]
    pub backend: String,
    
    /// SQLite 数据库路径
    pub sqlite_path: Option<PathBuf>,
    
    /// Markdown 目录
    pub markdown_dir: Option<PathBuf>,
    
    /// 嵌入模型配置
    #[serde(default)]
    pub embeddings: EmbeddingsConfig,
    
    /// 向量搜索配置
    #[serde(default)]
    pub vector_search: VectorSearchConfig,
    
    /// 响应缓存配置
    #[serde(default)]
    pub response_cache: ResponseCacheConfig,
    
    /// 文本分块配置
    #[serde(default)]
    pub chunking: ChunkingConfig,
    
    /// 记忆卫生管理配置
    #[serde(default)]
    pub hygiene: MemoryHygieneConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct EmbeddingsConfig {
    /// 提供商 (openai/ollama/gemini/huggingface)
    #[serde(default = "default_embedding_provider")]
    pub provider: String,
    
    /// 模型名称
    pub model: Option<String>,
    
    /// API Key
    pub api_key: Option<String>,
    
    /// API URL
    pub api_url: Option<String>,
    
    /// 维度数
    pub dimensions: Option<u32>,
    
    /// 批量大小
    #[serde(default = "default_batch_size")]
    pub batch_size: u32,
}

fn default_embedding_provider() -> String {
    "openai".to_string()
}

fn default_batch_size() -> u32 {
    32
}
```

### 3.6 Gateway 配置

```rust
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct GatewayConfig {
    /// 监听地址
    #[serde(default = "default_gateway_host")]
    pub host: String,
    
    /// 监听端口
    #[serde(default = "default_gateway_port")]
    pub port: u16,
    
    /// 是否需要配对
    #[serde(default = "default_false")]
    pub require_pairing: bool,
    
    /// 配对请求速率限制 (次/分钟)
    #[serde(default = "default_pair_rate_limit")]
    pub pair_rate_limit: u32,
    
    /// Webhook 速率限制 (次/分钟)
    #[serde(default = "default_webhook_rate_limit")]
    pub webhook_rate_limit: u32,
    
    /// 限流器最大键数
    #[serde(default = "default_rate_limit_max_keys")]
    pub rate_limit_max_keys: usize,
    
    /// 幂等性 TTL (秒)
    #[serde(default = "default_idempotency_ttl_secs")]
    pub idempotency_ttl_secs: u64,
    
    /// 幂等性最大键数
    #[serde(default = "default_idempotency_max_keys")]
    pub idempotency_max_keys: usize,
    
    /// SSL 配置
    #[serde(default)]
    pub ssl: SslConfig,
}

fn default_gateway_host() -> String {
    "0.0.0.0".to_string()
}

fn default_gateway_port() -> u16 {
    8080
}

fn default_pair_rate_limit() -> u32 {
    5
}

fn default_webhook_rate_limit() -> u32 {
    100
}
```

### 3.7 代理配置

```rust
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct ProxyConfig {
    /// HTTP 代理 URL
    pub http_proxy: Option<String>,
    
    /// HTTPS 代理 URL
    pub https_proxy: Option<String>,
    
    /// SOCKS5 代理 URL
    pub socks5_proxy: Option<String>,
    
    /// 代理绕过列表
    #[serde(default)]
    pub no_proxy: Vec<String>,
    
    /// 服务级代理配置
    #[serde(default)]
    pub service_proxies: HashMap<String, ServiceProxyConfig>,
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct ServiceProxyConfig {
    /// 代理 URL
    pub url: String,
    
    /// 代理类型 (http/https/socks5)
    pub r#type: String,
    
    /// 用户名 (可选)
    pub username: Option<String>,
    
    /// 密码 (可选)
    pub password: Option<String>,
}
```

## 4. 配置加载流程

### 4.1 加载顺序

```rust
impl Config {
    /// 从默认位置加载配置
    pub fn load() -> Result<Self> {
        // 1. 尝试从 ZEROCLAW_WORKSPACE 环境变量
        if let Ok(workspace) = std::env::var("ZEROCLAW_WORKSPACE") {
            let config_path = PathBuf::from(&workspace).join("config.toml");
            if config_path.exists() {
                return Self::load_from_path(&config_path);
            }
        }

        // 2. 尝试从 active_workspace.toml 标记文件
        let home = UserDirs::new()
            .context("Failed to get home directory")?
            .home_dir()
            .to_path_buf();
        
        let marker_path = home.join(".zeroclaw/active_workspace.toml");
        if marker_path.exists() {
            let workspace_path = std::fs::read_to_string(&marker_path)?;
            let config_path = PathBuf::from(workspace_path.trim()).join("config.toml");
            if config_path.exists() {
                return Self::load_from_path(&config_path);
            }
        }

        // 3. 回退到 ~/.zeroclaw/config.toml
        let default_config_path = home.join(".zeroclaw/config.toml");
        if default_config_path.exists() {
            return Self::load_from_path(&default_config_path);
        }

        // 4. 返回默认配置
        Ok(Self::default())
    }

    /// 从指定路径加载
    fn load_from_path(path: &Path) -> Result<Self> {
        let content = std::fs::read_to_string(path)
            .with_context(|| format!("Failed to read config from {}", path.display()))?;

        let mut config: Config = toml::from_str(&content)
            .with_context(|| format!("Failed to parse config from {}", path.display()))?;

        // 应用环境变量覆盖
        config.apply_env_overrides()?;

        // 设置计算字段
        config.workspace_dir = path.parent()
            .unwrap_or_else(|| Path::new("."))
            .to_path_buf();
        
        config.config_path = path.to_path_buf();

        Ok(config)
    }

    /// 应用环境变量覆盖
    fn apply_env_overrides(&mut self) -> Result<()> {
        // API Key 覆盖
        if let Ok(key) = std::env::var("ZEROCLAW_API_KEY") {
            self.api_key = Some(key);
        } else if let Ok(key) = std::env::var("API_KEY") {
            self.api_key = Some(key);
        }

        // API URL 覆盖
        if let Ok(url) = std::env::var("ZEROCLAW_API_URL") {
            self.api_url = Some(url);
        }

        // Provider 覆盖
        if let Ok(provider) = std::env::var("ZEROCLAW_DEFAULT_PROVIDER") {
            self.default_provider = Some(provider);
        }

        // Model 覆盖
        if let Ok(model) = std::env::var("ZEROCLAW_DEFAULT_MODEL") {
            self.default_model = Some(model);
        }

        Ok(())
    }
}
```

### 4.2 配置验证

```rust
impl Config {
    /// 验证配置的有效性
    pub fn validate(&self) -> Result<()> {
        // 验证必需字段
        if self.api_key.is_none() && self.model_providers.is_empty() {
            bail!("Either api_key or model_providers must be specified");
        }

        // 验证渠道配置
        self.channels_config.validate()?;

        // 验证内存配置
        self.memory.validate()?;

        // 验证安全配置
        self.security.validate()?;

        // 验证网关配置
        self.gateway.validate()?;

        Ok(())
    }
}

impl ChannelsConfig {
    pub fn validate(&self) -> Result<()> {
        // 至少启用一个渠道
        let enabled_count = [
            self.telegram.bot_token.is_some(),
            self.discord.bot_token.is_some(),
            self.slack.bot_token.is_some(),
            self.whatsapp.webhook_verify_token.is_some(),
            // ... 其他渠道
        ].iter().filter(|&&x| x).count();

        if enabled_count == 0 {
            warn!("No channels enabled");
        }

        // 验证特定渠道配置
        if self.telegram.webhook_enabled && self.telegram.webhook_port.is_none() {
            bail!("Telegram webhook_port is required when webhook_enabled is true");
        }

        Ok(())
    }
}
```

### 4.3 配置合并

```rust
impl Config {
    /// 合并多个配置 (后者覆盖前者)
    pub fn merge(mut self, other: Config) -> Self {
        // 合并基础配置
        if other.api_key.is_some() {
            self.api_key = other.api_key;
        }
        if other.api_url.is_some() {
            self.api_url = other.api_url;
        }
        if other.default_provider.is_some() {
            self.default_provider = other.default_provider;
        }
        if other.default_model.is_some() {
            self.default_model = other.default_model;
        }

        // 合并嵌套配置
        self.model_providers.extend(other.model_providers);
        
        // 合并渠道配置
        self.channels_config.merge(other.channels_config);

        // ... 合并其他配置

        self
    }
}
```

## 5. 运行时配置更新

### 5.1 代理配置热更新

```rust
static RUNTIME_PROXY_CONFIG: OnceLock<RwLock<ProxyConfig>> = OnceLock::new();
static RUNTIME_PROXY_CLIENT_CACHE: OnceLock<RwLock<HashMap<String, reqwest::Client>>> =
    OnceLock::new();

impl ProxyConfig {
    /// 更新运行时代理配置
    pub fn update_runtime_config(&self, service_key: &str) -> Result<()> {
        if let Some(runtime_config) = RUNTIME_PROXY_CONFIG.get() {
            let mut config = runtime_config.write().unwrap();
            *config = self.clone();
            
            // 清除客户端缓存
            if let Some(cache) = RUNTIME_PROXY_CLIENT_CACHE.get() {
                cache.write().unwrap().clear();
            }
        }
        
        Ok(())
    }

    /// 获取或创建代理客户端
    pub fn get_client(&self, service_key: &str) -> Result<reqwest::Client> {
        // 检查缓存
        if let Some(cache) = RUNTIME_PROXY_CLIENT_CACHE.get() {
            if let Some(client) = cache.read().unwrap().get(service_key) {
                return Ok(client.clone());
            }
        }

        // 创建新客户端
        let client = self.build_client(service_key)?;

        // 缓存客户端
        if let Some(cache) = RUNTIME_PROXY_CLIENT_CACHE.get() {
            cache.write().unwrap().insert(service_key.to_string(), client.clone());
        }

        Ok(client)
    }

    fn build_client(&self, service_key: &str) -> Result<reqwest::Client> {
        let mut builder = reqwest::ClientBuilder::new()
            .timeout(Duration::from_secs(30));

        // 应用服务级代理
        if let Some(proxy_config) = self.service_proxies.get(service_key) {
            let proxy = match proxy_config.r#type.as_str() {
                "http" => reqwest::Proxy::http(&proxy_config.url)?,
                "https" => reqwest::Proxy::https(&proxy_config.url)?,
                "socks5" => reqwest::Proxy::all(&proxy_config.url)?,
                _ => bail!("Unsupported proxy type: {}", proxy_config.r#type),
            };

            builder = builder.proxy(proxy);
        }

        Ok(builder.build()?)
    }
}
```

## 6. Schema 生成

### 6.1 JSON Schema 生成

```rust
use schemars::{schema_for, JsonSchema};

// Config 实现了 JsonSchema trait
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct Config {
    // ...
}

// 生成完整的 JSON Schema
pub fn generate_config_schema() -> serde_json::Value {
    let schema = schema_for!(Config);
    serde_json::to_value(schema).unwrap()
}

// 生成简化版 Schema (用于文档)
pub fn generate_simple_schema() -> serde_json::Value {
    let mut schema = generate_config_schema();
    
    // 移除复杂的 definitions
    schema.as_object_mut().unwrap().remove("definitions");
    
    schema
}
```

### 6.2 Schema 示例输出

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Config",
  "description": "Top-level ZeroClaw configuration, loaded from `config.toml`.",
  "type": "object",
  "properties": {
    "api_key": {
      "description": "API key for the selected provider. Overridden by `ZEROCLAW_API_KEY` or `API_KEY` env vars.",
      "type": ["string", "null"]
    },
    "default_provider": {
      "description": "Default provider ID or alias (e.g. `\"openrouter\"`, `\"ollama\"`, `\"anthropic\"`). Default: `\"openrouter\"`.",
      "type": ["string", "null"]
    },
    "default_model": {
      "description": "Default model routed through the selected provider (e.g. `\"anthropic/claude-sonnet-4-6\"`).",
      "type": ["string", "null"]
    },
    "security": {
      "$ref": "#/definitions/SecurityConfig"
    },
    "gateway": {
      "$ref": "#/definitions/GatewayConfig"
    }
  }
}
```

## 7. 配置持久化

### 7.1 保存到文件

```rust
impl Config {
    /// 保存配置到文件
    pub fn save(&self, path: &Path) -> Result<()> {
        let dir = path.parent()
            .ok_or_else(|| anyhow!("Invalid config path"))?;
        
        // 创建目录
        std::fs::create_dir_all(dir)?;

        // 序列化配置
        let content = toml::to_string_pretty(self)
            .context("Failed to serialize config")?;

        // 写入文件 (原子操作)
        let tmp_path = path.with_extension("toml.tmp");
        std::fs::write(&tmp_path, &content)?;
        std::fs::rename(&tmp_path, path)?;

        info!("Config saved to {}", path.display());

        Ok(())
    }

    /// 保存到默认位置
    pub fn save_default(&self) -> Result<()> {
        let home = UserDirs::new()
            .context("Failed to get home directory")?
            .home_dir()
            .to_path_buf();
        
        let config_path = home.join(".zeroclaw/config.toml");
        self.save(&config_path)
    }
}
```

## 8. 错误处理

### 8.1 配置错误类型

```rust
pub enum ConfigError {
    FileNotFound(PathBuf),
    ParseError(String),
    ValidationError(String),
    EnvVarError(String),
    MergeError(String),
}

impl From<toml::de::Error> for ConfigError {
    fn from(err: toml::de::Error) -> Self {
        Self::ParseError(err.to_string())
    }
}

impl From<toml::ser::Error> for ConfigError {
    fn from(err: toml::ser::Error) -> Self {
        Self::MergeError(err.to_string())
    }
}
```

## 9. 测试策略

### 9.1 单元测试

```rust
#[test]
fn test_config_load_and_validate() {
    let temp_dir = tempfile::tempdir().unwrap();
    let config_path = temp_dir.path().join("config.toml");

    // 写入测试配置
    let config_content = r#"
        api_key = "test-key"
        default_provider = "openai"
        default_model = "gpt-4"
        
        [security]
        autonomy = "supervised"
        workspace_only = true
        
        [gateway]
        port = 9090
    "#;

    std::fs::write(&config_path, config_content).unwrap();

    // 加载配置
    let config = Config::load_from_path(&config_path).unwrap();

    // 验证配置
    assert_eq!(config.api_key, Some("test-key".to_string()));
    assert_eq!(config.default_provider, Some("openai".to_string()));
    assert_eq!(config.gateway.port, 9090);

    // 验证应该通过
    config.validate().unwrap();
}

#[test]
fn test_config_env_override() {
    let mut config = Config::default();
    config.api_key = Some("original-key".to_string());

    // 设置环境变量
    std::env::set_var("ZEROCLAW_API_KEY", "override-key");

    // 应用覆盖
    config.apply_env_overrides().unwrap();

    // 验证覆盖生效
    assert_eq!(config.api_key, Some("override-key".to_string()));

    // 清理环境变量
    std::env::remove_var("ZEROCLAW_API_KEY");
}
```

### 9.2 集成测试

测试完整配置加载流程和组件创建。

## 10. 配置最佳实践

### 10.1 配置组织建议

1. **使用环境变量管理敏感信息**
   ```bash
   export ZEROCLAW_API_KEY="sk-..."
   export ZEROCLAW_TELEGRAM_BOT_TOKEN="..."
   ```

2. **使用多个配置文件分离环境**
   ```
   ~/.zeroclaw/
   ├── config.dev.toml     # 开发环境
   ├── config.prod.toml    # 生产环境
   └── active_workspace.toml  # 当前激活的配置
   ```

3. **使用 Provider Profiles 管理多模型**
   ```toml
   [model_providers.openai]
   type = "openai"
   api_key = "sk-..."
   model = "gpt-4"

   [model_providers.anthropic]
   type = "anthropic"
   api_key = "sk-ant-..."
   model = "claude-3-opus"
   ```

4. **使用查询分类自动路由**
   ```toml
   [[query_classification.rules]]
   keywords = ["code", "programming", "debug"]
   hint = "coding-assistant"

   [model_routes.coding-assistant]
   provider = "anthropic"
   model = "claude-sonnet-4-6"
   ```

### 10.2 配置检查清单

- [ ] 设置了强 API Key 或使用环境变量
- [ ] 配置了合适的自主性级别 (推荐 supervised)
- [ ] 定义了允许的命令白名单
- [ ] 配置了速率限制防止滥用
- [ ] 启用了审计日志
- [ ] 配置了至少一个通信渠道
- [ ] 设置了合理的成本上限
- [ ] 配置了备份策略 (如果使用重要数据)
