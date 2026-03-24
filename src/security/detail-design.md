# Security 模块详细设计

## 1. 模块概述

Security 模块是 ZeroClaw 的安全基础设施核心，负责策略执行、沙箱隔离和密钥管理。提供多层次的安全防护机制，包括自主性级别控制、设备配对认证、审计日志和多种沙箱后端。

## 2. 架构设计

### 2.1 模块结构

src/security/
- mod.rs           - 模块导出 (3.4KB)
- traits.rs        - Sandbox Trait 定义 (3.9KB)
- policy.rs        - 安全策略 (78.0KB - 最核心)
- pairing.rs       - 设备配对守卫 (25.0KB)
- secrets.rs       - 密钥存储 (30.7KB)
- audit.rs         - 审计日志 (12.0KB)
- estop.rs         - 紧急停止 (13.0KB)
- otp.rs           - OTP 验证 (9.4KB)
- prompt_guard.rs  - Prompt 注入防御 (12.7KB)
- leak_detector.rs - 敏感信息泄露检测 (12.7KB)
- domain_matcher.rs - 域名匹配器 (7.2KB)
- detect.rs        - 沙箱检测 (5.2KB)
- docker.rs        - Docker 沙箱 (5.6KB)
- firejail.rs      - Firejail 沙箱 (5.5KB)
- bubblewrap.rs    - Bubblewrap 沙箱 (4.8KB)
- landlock.rs      - Landlock LSM (8.8KB)

### 2.2 设计模式

- **策略模式**: 多种沙箱后端可插拔
- **责任链**: 多层安全检查串联
- **装饰器模式**: 在基础功能上增加安全层
- **单例模式**: SecretStore 全局唯一实例

## 3. 核心组件详解

### 3.1 安全策略 (policy.rs - 78.0KB)

#### 3.1.1 自主性级别

```rust
pub enum AutonomyLevel {
    Manual,         // 完全手动，所有操作需确认
    Supervised,     // 监督模式，高风险操作需确认
    Autonomous,     // 自主模式，在限制范围内自动执行
}

impl Default for AutonomyLevel {
    fn default() -> Self {
        Self::Supervised  // 默认监督模式 (安全优先)
    }
}
```

#### 3.1.2 SecurityPolicy 结构体

```rust
pub struct SecurityPolicy {
    pub autonomy: AutonomyLevel,
    pub workspace_only: bool,           // 限制在工作目录内
    pub allowed_roots: Vec<PathBuf>,    // 允许的根目录列表
    pub allowed_commands: Vec<String>,  // 允许的命令白名单
    pub max_actions_per_hour: u32,      // 每小时最大操作数
    pub max_cost_per_day_cents: u32,    // 每日最大成本 (美分)
    pub require_pairing: bool,          // 是否需要设备配对
    pub allowed_domains: Vec<String>,   // 允许的网络域名
    pub blocked_domains: Vec<String>,   // 阻止的域名
}
```

#### 3.1.3 策略检查流程

```rust
impl SecurityPolicy {
    /// 检查 Shell 命令是否被允许
    pub fn is_command_allowed(&self, command: &str) -> bool {
        match self.autonomy {
            AutonomyLevel::Manual => false,  // 完全手动模式下不允许自动执行
            AutonomyLevel::Supervised | AutonomyLevel::Autonomous => {
                // 检查白名单
                self.allowed_commands.iter().any(|allowed| {
                    command.starts_with(allowed) || 
                    self.matches_glob_pattern(allowed, command)
                })
            }
        }
    }

    /// 检查路径是否在允许范围内
    pub fn is_path_allowed(&self, path: &Path) -> bool {
        if self.workspace_only {
            return path.starts_with(&self.workspace_dir);
        }

        self.allowed_roots.iter().any(|root| {
            path.starts_with(root)
        })
    }

    /// 检查域名是否被允许
    pub fn is_domain_allowed(&self, domain: &str) -> bool {
        // 先检查黑名单
        if self.blocked_domains.iter().any(|blocked| {
            self.domain_matches(blocked, domain)
        }) {
            return false;
        }

        // 再检查白名单
        if !self.allowed_domains.is_empty() {
            return self.allowed_domains.iter().any(|allowed| {
                self.domain_matches(allowed, domain)
            });
        }

        true  // 无白名单时默认允许
    }
}
```

### 3.2 设备配对 (pairing.rs - 25.0KB)

#### 3.2.1 PairingGuard 结构体

```rust
pub struct PairingGuard {
    require_pairing: bool,
    trusted_devices: HashSet<DeviceId>,
    pending_codes: DashMap<Code, ExpiryTime>,
}

struct DeviceId {
    id: String,
    name: String,
    public_key: PublicKey,
}
```

#### 3.2.2 配对流程

```rust
impl PairingGuard {
    /// 生成配对码
    pub fn generate_pairing_code(&self) -> Result<Code> {
        let code = format!("{:06}", rand::random::<u32>() % 1_000_000);
        let expiry = Utc::now() + Duration::minutes(10);
        
        self.pending_codes.insert(code.clone(), expiry);
        
        Ok(code)
    }

    /// 验证配对码
    pub fn verify_pairing_code(&self, code: &str) -> Result<DeviceId> {
        // 检查是否存在且未过期
        if let Some(expiry) = self.pending_codes.get(code) {
            if Utc::now() > *expiry {
                bail!("Pairing code expired");
            }
            
            // 生成设备 ID
            let device_id = self.create_device_id()?;
            self.trusted_devices.insert(device_id.clone());
            
            Ok(device_id)
        } else {
            bail!("Invalid pairing code");
        }
    }

    /// 检查设备是否可信
    pub fn is_trusted_device(&self, device_id: &DeviceId) -> bool {
        self.trusted_devices.contains(device_id)
    }
}
```

### 3.3 密钥存储 (secrets.rs - 30.7KB)

#### 3.3.1 SecretStore 结构体

```rust
pub struct SecretStore {
    keyring_entry: Option<keyring::Entry>,
    encryption_key: Option<[u8; 32]>,  // AES-256-GCM key
    storage_path: PathBuf,
}
```

#### 3.3.2 加密实现

```rust
impl SecretStore {
    pub fn new(config_dir: &Path, use_encryption: bool) -> Self {
        let storage_path = config_dir.join("secret_store.enc");
        
        let encryption_key = if use_encryption {
            // 尝试从系统密钥环获取
            if let Ok(entry) = keyring::Entry::new("zeroclaw", "master_key") {
                if let Ok(key_data) = entry.get_password() {
                    let mut key = [0u8; 32];
                    key.copy_from_slice(&key_data.as_bytes()[..32]);
                    Some(key)
                } else {
                    None
                }
            } else {
                None
            }
        } else {
            None
        };
        
        Self {
            keyring_entry: None,
            encryption_key,
            storage_path,
        }
    }

    /// 加密存储
    pub fn encrypt(&self, plaintext: &str) -> Result<Vec<u8>> {
        if let Some(key) = self.encryption_key {
            // 使用 AES-256-GCM
            let cipher = Aes256Gcm::new_from_slice(&key)?;
            let nonce = generate_nonce();
            
            let mut ciphertext = cipher.encrypt(&nonce, plaintext.as_bytes())?;
            
            // 前缀添加 nonce
            let mut result = nonce.to_vec();
            result.append(&mut ciphertext);
            
            Ok(result)
        } else {
            // 无加密模式 (不推荐)
            Ok(plaintext.as_bytes().to_vec())
        }
    }

    /// 解密读取
    pub fn decrypt(&self, ciphertext: &[u8]) -> Result<String> {
        if let Some(key) = self.encryption_key {
            if ciphertext.len() < 12 {
                bail!("Ciphertext too short");
            }
            
            let nonce = &ciphertext[..12];
            let data = &ciphertext[12..];
            
            let cipher = Aes256Gcm::new_from_slice(key)?;
            let plaintext = cipher.decrypt(nonce.into(), data)?;
            
            Ok(String::from_utf8(plaintext)?)
        } else {
            Ok(String::from_utf8(ciphertext.to_vec())?)
        }
    }
}
```

### 3.4 审计日志 (audit.rs - 12.0KB)

#### 3.4.1 AuditEvent 类型

```rust
pub enum AuditEventType {
    ToolExecution { tool: String, success: bool },
    FileAccess { path: String, operation: String },
    NetworkRequest { url: String, method: String },
    Authentication { user: String, method: String },
    PolicyViolation { rule: String, details: String },
    SecretAccess { key_name: String },
}

pub struct AuditEvent {
    pub id: String,
    pub event_type: AuditEventType,
    pub timestamp: DateTime<Utc>,
    pub severity: Severity,
    pub user: Option<String>,
    pub session_id: Option<String>,
    pub details: serde_json::Value,
}
```

#### 3.4.2 审计日志记录

```rust
pub struct AuditLogger {
    log_path: PathBuf,
    max_size: u64,  // 最大文件大小 (字节)
    rotation: RotationPolicy,
    writer: Mutex<Box<dyn Write + Send>>,
}

impl AuditLogger {
    pub fn log(&self, event: AuditEvent) -> Result<()> {
        let json = serde_json::to_string(&event)?;
        
        // 写入日志文件
        writeln!(self.writer.lock().unwrap(), "{}", json)?;
        
        // 检查是否需要轮转
        self.check_rotation()?;
        
        Ok(())
    }

    /// 查询审计日志
    pub fn query(
        &self,
        start_time: DateTime<Utc>,
        end_time: DateTime<Utc>,
        event_type: Option<&str>,
    ) -> Result<Vec<AuditEvent>> {
        let file = File::open(&self.log_path)?;
        let reader = BufReader::new(file);
        
        let mut events = Vec::new();
        for line in reader.lines() {
            let event: AuditEvent = serde_json::from_str(&line?)?;
            
            if event.timestamp >= start_time && event.timestamp <= end_time {
                if let Some(ty) = event_type {
                    if self.event_type_matches(&event.event_type, ty) {
                        events.push(event);
                    }
                } else {
                    events.push(event);
                }
            }
        }
        
        Ok(events)
    }
}
```

### 3.5 紧急停止 (estop.rs - 13.0KB)

#### 3.5.1 EstopLevel 枚举

```rust
pub enum EstopLevel {
    KillAll,        // 终止所有操作
    NetworkKill,    // 切断网络访问
    DomainBlock,    // 阻止特定域名
    ToolFreeze,     // 冻结特定工具
}
```

#### 3.5.2 EstopManager 实现

```rust
pub struct EstopManager {
    state: RwLock<EstopState>,
    config: EstopConfig,
}

struct EstopState {
    engaged: bool,
    level: EstopLevel,
    engaged_at: DateTime<Utc>,
    reason: Option<String>,
    blocked_domains: HashSet<String>,
    frozen_tools: HashSet<String>,
}

impl EstopManager {
    /// 接合紧急停止
    pub fn engage(&self, level: EstopLevel, reason: Option<String>) -> Result<()> {
        let mut state = self.state.write().unwrap();
        
        state.engaged = true;
        state.level = level;
        state.engaged_at = Utc::now();
        state.reason = reason;
        
        // 应用限制
        match level {
            EstopLevel::NetworkKill => {
                self.block_all_network()?;
            }
            EstopLevel::DomainBlock => {
                // 阻止配置的域名列表
            }
            EstopLevel::ToolFreeze => {
                // 冻结配置的工具列表
            }
            _ => {}
        }
        
        self.audit_engagement(&state)?;
        
        Ok(())
    }

    /// 恢复操作
    pub fn resume(&self, selector: ResumeSelector, otp: Option<&str>) -> Result<()> {
        let mut state = self.state.write().unwrap();
        
        // 验证 OTP (如果配置了)
        if self.config.require_otp {
            if !self.verify_otp(otp)? {
                bail!("Invalid OTP");
            }
        }
        
        // 根据选择器恢复特定部分
        match selector {
            ResumeSelector::Network => {
                self.restore_network()?;
            }
            ResumeSelector::Domain(domain) => {
                state.blocked_domains.remove(&domain);
            }
            ResumeSelector::Tool(tool) => {
                state.frozen_tools.remove(&tool);
            }
            ResumeSelector::All => {
                state.engaged = false;
                state.blocked_domains.clear();
                state.frozen_tools.clear();
            }
        }
        
        self.audit_resume(&state, &selector)?;
        
        Ok(())
    }
}
```

### 3.6 OTP 验证 (otp.rs - 9.4KB)

```rust
pub struct OtpValidator {
    secret: Secret<String>,
    issuer: String,
    account_name: String,
}

impl OtpValidator {
    pub fn from_config(
        config: &OtpConfig,
        config_dir: &Path,
        store: &SecretStore,
    ) -> Result<(Self, Option<String>)> {
        let secret = if let Some(existing) = load_secret(store)? {
            existing
        } else {
            // 生成新密钥
            let secret = generate_totp_secret();
            save_secret(store, &secret)?;
            
            // 返回注册 URI
            let uri = generate_totp_uri(&secret, &config.issuer, &config.account_name);
            return Ok((Self::new(secret, config), Some(uri)));
        };
        
        Ok((Self::new(secret, config), None))
    }

    /// 验证 TOTP
    pub fn validate(&self, code: &str) -> bool {
        let totp = Totp::new(
            Algorithm::SHA1,
            6,
            30,
            0,
            None,
            self.secret.expose().as_bytes(),
        ).unwrap();
        
        totp.check_current(code).is_ok()
    }
}
```

### 3.7 Prompt 注入防御 (prompt_guard.rs)

```rust
pub struct PromptGuard {
    patterns: Vec<Regex>,
    threshold: f32,
}

impl PromptGuard {
    /// 检测 Prompt 注入攻击
    pub fn detect_injection(&self, input: &str) -> GuardResult {
        let mut score = 0.0;
        
        // 检查常见注入模式
        for pattern in &self.patterns {
            if pattern.is_match(input) {
                score += 0.2;
            }
        }
        
        // 检查指令覆盖尝试
        if self.contains_override_attempt(input) {
            score += 0.3;
        }
        
        // 检查上下文逃逸
        if self.contains_context_escape(input) {
            score += 0.3;
        }
        
        if score >= self.threshold {
            GuardResult::Blocked(GuardAction::Reject)
        } else if score >= self.threshold * 0.5 {
            GuardResult::Warn(GuardAction::Log)
        } else {
            GuardResult::Allow
        }
    }
}
```

### 3.8 沙箱抽象 (traits.rs)

```rust
#[async_trait]
pub trait Sandbox: Send + Sync {
    /// 执行命令 (在沙箱中)
    async fn exec(&self, cmd: &str, args: &[&str]) -> Result<std::process::Output>;
    
    /// 限制文件系统访问
    fn restrict_fs(&self, paths: &[&Path]) -> Result<()>;
    
    /// 限制网络访问
    fn restrict_network(&self, allowed_hosts: &[&str]) -> Result<()>;
    
    /// 限制资源使用
    fn restrict_resources(&self, limits: ResourceLimits) -> Result<()>;
}

/// 资源限制
pub struct ResourceLimits {
    pub max_memory: Option<u64>,      // 最大内存 (字节)
    pub max_cpu_percent: Option<u32>, // 最大 CPU 百分比
    pub max_processes: Option<u32>,   // 最大进程数
    pub max_open_files: Option<u32>,  // 最大打开文件数
}
```

### 3.9 Docker 沙箱实现 (docker.rs)

```rust
pub struct DockerSandbox {
    client: bollard::Docker,
    container_id: String,
    config: SandboxConfig,
}

#[async_trait]
impl Sandbox for DockerSandbox {
    async fn exec(&self, cmd: &str, args: &[&str]) -> Result<std::process::Output> {
        // 在容器中执行命令
        let exec_config = CreateExecOptions {
            cmd: Some(iter::once(cmd).chain(args.iter().copied()).collect()),
            attach_stdout: Some(true),
            attach_stderr: Some(true),
            ..Default::default()
        };
        
        let exec = self.client.create_exec(&self.container_id, exec_config).await?;
        let output = self.client.start_exec(&exec.id, None).await?;
        
        Ok(parse_output(output))
    }

    fn restrict_fs(&self, paths: &[&Path]) -> Result<()> {
        // Docker 容器天然隔离文件系统
        // 额外挂载点限制通过 volumes 配置实现
        Ok(())
    }

    fn restrict_network(&self, allowed_hosts: &[&str]) -> Result<()> {
        // 通过 iptables 规则限制
        // 或使用 Docker 网络策略
        Ok(())
    }
}
```

## 4. 安全检查流程

### 4.1 工具执行前检查

```rust
pub async fn check_before_tool_execution(
    policy: &SecurityPolicy,
    tool: &dyn Tool,
    args: &serde_json::Value,
    sandbox: Option<&dyn Sandbox>,
) -> Result<()> {
    // 1. 检查自主性级别
    if policy.autonomy == AutonomyLevel::Manual {
        bail!("Manual mode requires explicit approval");
    }

    // 2. 检查速率限制
    if !policy.check_rate_limit(tool.name()).await? {
        bail!("Rate limit exceeded");
    }

    // 3. 检查工具是否在白名单
    if !policy.allowed_tools.contains(tool.name()) {
        bail!("Tool '{}' not in allowed list", tool.name());
    }

    // 4. 检查参数安全性
    if let Some(paths) = extract_file_paths(args) {
        for path in paths {
            if !policy.is_path_allowed(path)? {
                bail!("Path '{}' outside allowed boundaries", path.display());
            }
        }
    }

    // 5. 如果有沙箱，在沙箱中执行
    if let Some(sb) = sandbox {
        // 应用额外限制
        sb.restrict_fs(&policy.allowed_roots)?;
        sb.restrict_network(&policy.allowed_domains)?;
    }

    Ok(())
}
```

### 4.2 敏感信息泄露检测

```rust
pub struct LeakDetector {
    patterns: Vec<Regex>,
    entropy_threshold: f32,
}

impl LeakDetector {
    /// 检测输出中是否包含敏感信息
    pub fn detect_leaks(&self, output: &str) -> LeakResult {
        // 检查 API Key 模式
        for pattern in &self.patterns {
            if let Some(m) = pattern.find(output) {
                return LeakResult::Leaked(m.as_str().to_string());
            }
        }

        // 检查高熵字符串 (可能是密钥)
        for token in output.split_whitespace() {
            if token.len() > 20 && self.calculate_entropy(token) > self.entropy_threshold {
                return LeakResult::Suspicious(token.to_string());
            }
        }

        LeakResult::Clean
    }

    fn calculate_entropy(&self, s: &str) -> f32 {
        // Shannon entropy calculation
        let mut freq = [0u32; 256];
        for byte in s.as_bytes() {
            freq[*byte as usize] += 1;
        }

        let len = s.len() as f32;
        let mut entropy = 0.0;

        for &count in &freq {
            if count > 0 {
                let p = count as f32 / len;
                entropy -= p * p.log2();
            }
        }

        entropy
    }
}
```

## 5. 配置选项

```toml
[security]
autonomy = "supervised"  # manual, supervised, autonomous
workspace_only = true
max_actions_per_hour = 100
max_cost_per_day_cents = 1000

[security.allowed_roots]
paths = ["~/projects", "/tmp/zeroclaw"]

[security.allowed_commands]
commands = ["git", "cargo", "ls", "cat", "grep"]

[security.network]
allowed_domains = ["api.github.com", "api.openai.com"]
blocked_domains = ["malware.com", "*.evil.com"]

[security.pairing]
require_pairing = true
timeout_minutes = 10

[security.otp]
enabled = true
issuer = "ZeroClaw"
account_name = "zeroclaw@localhost"

[security.sandbox]
enabled = true
backend = "auto"  # auto, docker, firejail, bubblewrap, landlock

[security.audit]
enabled = true
log_path = "~/.zeroclaw/audit.log"
max_size_mb = 100
rotation = "daily"

[security.estop]
enabled = true
require_otp = true
```

## 6. 错误处理

### 6.1 错误类型

```rust
pub enum SecurityError {
    PolicyViolation { rule: String, details: String },
    AuthenticationFailed(String),
    AuthorizationDenied(String),
    SandboxError(anyhow::Error),
    EncryptionError(anyhow::Error),
    RateLimitExceeded,
    EStopEngaged(EstopLevel),
}
```

### 6.2 错误响应

所有一级安全错误都会:
1. 记录到审计日志
2. 返回明确的错误消息 (但不泄露敏感信息)
3. 可能触发紧急停止

## 7. 性能优化

### 7.1 缓存检查结果

```rust
use moka::future::Cache;

pub struct CachedPolicyChecker {
    cache: Cache<String, bool>,
    policy: Arc<SecurityPolicy>,
}

impl CachedPolicyChecker {
    pub async fn is_allowed(&self, key: &str) -> bool {
        if let Some(&result) = self.cache.get(key) {
            return result;
        }

        let result = self.policy.check(key).await;
        self.cache.insert(key.to_string(), result).await;
        result
    }
}
```

### 7.2 异步审计日志

使用通道异步写入审计日志，避免阻塞主流程。

## 8. 测试策略

### 8.1 单元测试

```rust
#[test]
fn test_policy_is_path_allowed() {
    let policy = SecurityPolicy {
        workspace_only: true,
        workspace_dir: PathBuf::from("/workspace"),
        ..Default::default()
    };

    assert!(policy.is_path_allowed(Path::new("/workspace/test.txt")));
    assert!(!policy.is_path_allowed(Path::new("/etc/passwd")));
}
```

### 8.2 集成测试

在沙箱环境中测试真实场景的安全策略。

## 9. 监控指标

### 9.1 关键指标

- 策略违规次数
- 认证失败率
- 沙箱启动时间
- 审计日志大小增长率
- 紧急停止触发次数

### 9.2 告警规则

```rust
// 示例：策略违规频率过高
if policy_violations_last_hour > 10 {
    trigger_alert("High policy violation rate detected");
}

// 紧急停止被触发
if estop_engaged {
    trigger_alert("E-Stop engaged: {:?}", reason);
}
```

## 10. 扩展指南

### 10.1 添加新的沙箱后端

1. 创建 `src/security/my_sandbox.rs`
2. 实现 `Sandbox` trait
3. 在 `detect.rs` 中注册
4. 更新配置 schema

### 10.2 最小沙箱实现

```rust
use crate::security::traits::Sandbox;

pub struct MySandbox;

#[async_trait]
impl Sandbox for MySandbox {
    async fn exec(&self, cmd: &str, args: &[&str]) -> Result<std::process::Output> {
        // 实现隔离执行逻辑
        Ok(Command::new(cmd).args(args).output()?)
    }

    fn restrict_fs(&self, paths: &[&Path]) -> Result<()> {
        // 实现文件系统限制
        Ok(())
    }

    fn restrict_network(&self, allowed_hosts: &[&str]) -> Result<()> {
        // 实现网络限制
        Ok(())
    }
}
```
