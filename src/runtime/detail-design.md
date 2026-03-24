# Runtime 模块详细设计

## 1. 模块概述

Runtime 模块提供运行时适配器抽象，屏蔽不同执行环境 (本地、Docker、WASM、云函数) 的差异。通过统一的 Trait 接口声明平台能力 (Shell 访问、文件系统、长运行进程等),使 Agent 能够根据运行环境自适应调整行为。

## 2. 架构设计

### 2.1 模块结构

src/runtime/
- mod.rs          - 模块导出和运行时创建 (2.6KB)
- traits.rs       - RuntimeAdapter Trait 定义 (4.7KB - 核心)
- native.rs       - 本地运行时实现 (2.1KB)
- docker.rs       - Docker 运行时实现 (8.0KB)
- wasm.rs         - WASM 运行时实现 (23.4KB)

### 2.2 设计模式

- **适配器模式**: 统一不同运行时平台的差异
- **策略模式**: 根据运行时能力选择执行策略
- **工厂模式**: 创建合适的运行时实例
- **能力查询模式**: 运行时前检查平台支持的能力

### 2.3 运行时层次

```
RuntimeAdapter (Trait)
├── NativeRuntime (本地执行)
│   ├── 完整的 Shell 访问
│   ├── 文件系统读写
│   └── 长运行进程支持
│
├── DockerRuntime (容器化执行)
│   ├── 容器内 Shell 访问
│   ├── 容器文件系统
│   ├── 资源限制
│   └── 网络隔离
│
└── WasmRuntime (WebAssembly 执行)
    ├── 无 Shell 访问
    ├── 虚拟文件系统
    ├── 内存限制
    └── 事件驱动模型
```

## 3. 核心 Trait 详解 (traits.rs)

### 3.1 RuntimeAdapter Trait

```rust
/// 运行时适配器 Trait
/// 
/// 实现此 Trait 以将 Agent 移植到新的执行环境。
/// 适配器声明平台能力并提供平台特定的操作实现。
/// 
/// 实现必须是 `Send + Sync` 因为适配器在异步任务间共享。
pub trait RuntimeAdapter: Send + Sync {
    /// 返回运行时环境的可读名称
    /// 
    /// 用于日志和诊断 (如 "native", "docker", "cloudflare-workers")
    fn name(&self) -> &str;

    /// 报告是否支持 Shell 命令执行
    /// 
    /// 当为 false 时，Agent 禁用基于 Shell 的工具。
    /// Serverless 和边缘运行时通常返回 false。
    fn has_shell_access(&self) -> bool;

    /// 报告是否支持文件系统读写
    /// 
    /// 当为 false 时，Agent 禁用基于文件的工具并回退到内存存储。
    fn has_filesystem_access(&self) -> bool;

    /// 返回持久化存储的基路径
    /// 
    /// 内存后端、日志和其他工件存储在此路径下。
    /// 实现应返回平台适当的可写目录。
    fn storage_path(&self) -> PathBuf;

    /// 报告是否支持长运行后台进程
    /// 
    /// 当为 true 时，Agent 可启动网关服务器、心跳循环等持久任务。
    /// Serverless 运行时如有短执行限制应返回 false。
    fn supports_long_running(&self) -> bool;

    /// 返回最大内存预算 (字节)
    /// 
    /// 值为 0 (默认) 表示无限制。
    /// 受限环境 (嵌入式、serverless) 应返回实际内存上限，
    /// 以便 Agent 调整缓冲区大小和缓存策略。
    fn memory_budget(&self) -> u64 {
        0  // 默认无限制
    }

    /// 构建适合此运行时的 Shell 命令进程
    /// 
    /// 构造一个 [`tokio::process::Command`] 来执行 `command`,
    /// 使用 `workspace_dir` 作为工作目录。
    /// 
    /// 实现可根据需要添加沙箱包装器、设置环境变量或重定向 I/O。
    /// 
    /// # Errors
    /// 
    /// 如果运行时不支持 Shell 访问或无法构造命令则返回错误。
    fn build_shell_command(
        &self,
        command: &str,
        workspace_dir: &Path,
    ) -> anyhow::Result<tokio::process::Command>;

    // === 可选的高级能力方法 ===

    /// 是否支持进程间通信 (IPC)
    fn supports_ipc(&self) -> bool {
        false  // 默认不支持
    }

    /// 是否支持网络套接字
    fn supports_networking(&self) -> bool {
        true  // 默认支持
    }

    /// 是否支持多线程
    fn supports_threads(&self) -> bool {
        true  // 默认支持
    }

    /// 获取 CPU 核心数
    fn num_cpus(&self) -> usize {
        std::thread::available_parallelism()
            .map(|p| p.get())
            .unwrap_or(1)
    }

    /// 是否支持信号处理 (SIGINT, SIGTERM 等)
    fn supports_signals(&self) -> bool {
        cfg!(unix)  // Unix 系统默认支持
    }
}
```

### 3.2 能力查询辅助结构

```rust
/// 运行时能力标志集合
#[derive(Debug, Clone, Copy)]
pub struct RuntimeCapabilities {
    pub has_shell: bool,
    pub has_filesystem: bool,
    pub has_long_running: bool,
    pub has_networking: bool,
    pub has_ipc: bool,
    pub has_threads: bool,
    pub has_signals: bool,
    pub memory_budget: u64,
    pub num_cpus: usize,
}

impl RuntimeCapabilities {
    /// 从 RuntimeAdapter 提取能力
    pub fn from_adapter(adapter: &dyn RuntimeAdapter) -> Self {
        Self {
            has_shell: adapter.has_shell_access(),
            has_filesystem: adapter.has_filesystem_access(),
            has_long_running: adapter.supports_long_running(),
            has_networking: adapter.supports_networking(),
            has_ipc: adapter.supports_ipc(),
            has_threads: adapter.supports_threads(),
            has_signals: adapter.supports_signals(),
            memory_budget: adapter.memory_budget(),
            num_cpus: adapter.num_cpus(),
        }
    }

    /// 是否为功能完整的运行时
    pub fn is_full_featured(&self) -> bool {
        self.has_shell && self.has_filesystem && self.has_long_running
    }

    /// 是否为受限运行时
    pub fn is_constrained(&self) -> bool {
        !self.has_shell || !self.has_filesystem || self.memory_budget > 0
    }

    /// 是否适合运行 Agent
    pub fn can_run_agent(&self) -> bool {
        // 至少需要文件系统和某种形式的长期运行能力
        self.has_filesystem && (self.has_long_running || self.memory_budget > 0)
    }
}
```

## 4. 运行时实现详解

### 4.1 本地运行时 (native.rs)

```rust
pub struct NativeRuntime {
    storage_path: PathBuf,
    memory_budget: u64,
}

impl NativeRuntime {
    pub fn new(workspace_dir: &Path) -> Self {
        let storage_path = workspace_dir.join(".zeroclaw/storage");
        
        // 确保目录存在
        if let Err(e) = std::fs::create_dir_all(&storage_path) {
            warn!("Failed to create storage directory: {}", e);
        }

        Self {
            storage_path,
            memory_budget: 0,  // 本地运行无内存限制
        }
    }

    pub fn with_memory_budget(mut self, budget: u64) -> Self {
        self.memory_budget = budget;
        self
    }
}

impl RuntimeAdapter for NativeRuntime {
    fn name(&self) -> &str {
        "native"
    }

    fn has_shell_access(&self) -> bool {
        true  // 本地环境总是支持 Shell
    }

    fn has_filesystem_access(&self) -> bool {
        true  // 本地环境总是支持文件系统
    }

    fn storage_path(&self) -> PathBuf {
        self.storage_path.clone()
    }

    fn supports_long_running(&self) -> bool {
        true  // 本地环境支持长运行进程
    }

    fn memory_budget(&self) -> u64 {
        self.memory_budget
    }

    fn build_shell_command(
        &self,
        command: &str,
        workspace_dir: &Path,
    ) -> anyhow::Result<tokio::process::Command> {
        // 确定 Shell
        let shell = if cfg!(windows) {
            ("cmd", vec!["/C".to_string(), command.to_string()])
        } else {
            ("sh", vec!["-c".to_string(), command.to_string()])
        };

        let mut cmd = tokio::process::Command::new(shell.0);
        cmd.args(&shell.1);
        cmd.current_dir(workspace_dir);

        // 继承标准输出和错误输出
        cmd.stdout(std::process::Stdio::piped());
        cmd.stderr(std::process::Stdio::piped());

        Ok(cmd)
    }

    // 重写默认实现以提供更准确的信息
    fn num_cpus(&self) -> usize {
        std::thread::available_parallelism()
            .map(|p| p.get())
            .unwrap_or(1)
    }

    fn supports_signals(&self) -> bool {
        cfg!(unix)  // Unix 支持信号
    }
}
```

### 4.2 Docker 运行时 (docker.rs)

```rust
pub struct DockerRuntime {
    client: bollard::Docker,
    container_id: String,
    config: DockerConfig,
    capabilities: RuntimeCapabilities,
}

#[derive(Debug, Clone)]
pub struct DockerConfig {
    pub container_id: String,
    pub memory_limit: Option<u64>,      // 内存限制 (字节)
    pub cpu_limit: Option<f64>,         // CPU 限制 (核心数)
    pub network_enabled: bool,          // 是否启用网络
    pub allowed_hosts: Vec<String>,     // 允许的主机列表
    pub volume_mounts: Vec<VolumeMount>, // 卷挂载
}

#[derive(Debug, Clone)]
pub struct VolumeMount {
    pub host_path: PathBuf,
    pub container_path: PathBuf,
    pub read_only: bool,
}

impl DockerRuntime {
    pub async fn new(config: DockerConfig) -> Result<Self> {
        let client = bollard::Docker::connect_with_local_defaults()?;
        
        // 验证容器是否存在
        let _ = client.inspect_container(&config.container_id, None).await?;

        // 检查容器配置
        let container_info = client.inspect_container(&config.container_id, None).await?;
        
        // 提取实际限制
        let memory_limit = container_info.host_config
            .as_ref()
            .and_then(|hc| hc.memory)
            .map(|m| m as u64);

        let cpu_limit = container_info.host_config
            .as_ref()
            .and_then(|hc| hc.nano_cpus)
            .map(|cpus| cpus as f64 / 1_000_000_000.0);

        let capabilities = RuntimeCapabilities {
            has_shell: true,
            has_filesystem: true,
            has_long_running: true,
            has_networking: config.network_enabled,
            has_ipc: false,  // Docker 容器默认不支持 IPC
            has_threads: true,
            has_signals: true,
            memory_budget: memory_limit.unwrap_or(0),
            num_cpus: cpu_limit.map(|c| c.ceil() as usize).unwrap_or(1),
        };

        Ok(Self {
            client,
            config,
            capabilities,
        })
    }

    /// 在容器中执行命令
    pub async fn exec_in_container(
        &self,
        command: &str,
        working_dir: &Path,
    ) -> Result<std::process::Output> {
        use bollard::container::{ExecContainerOptions, StartExecResults};
        use futures_util::stream::TryStreamExt;

        // 创建 exec 实例
        let exec = self.client
            .create_exec::<String>(
                &self.config.container_id,
                ExecContainerOptions {
                    cmd: Some(vec!["sh", "-c", command]),
                    working_dir: Some(working_dir.to_str().unwrap()),
                    attach_stdout: Some(true),
                    attach_stderr: Some(true),
                    ..Default::default()
                },
            )
            .await?;

        // 启动执行
        match self.client.start_exec(&exec.id, None).await? {
            StartExecResults::Attached { output, .. } => {
                // 收集输出
                let mut stdout = Vec::new();
                let mut stderr = Vec::new();

                while let Some(chunk) = output.try_next().await? {
                    // 解析 Docker exec 输出格式
                    match chunk[0] {
                        1 => stdout.extend_from_slice(&chunk[8..]),
                        2 => stderr.extend_from_slice(&chunk[8..]),
                        _ => {}
                    }
                }

                // 检查退出码
                let inspect = self.client.inspect_exec(&exec.id).await?;
                let exit_code = inspect.exit_code.unwrap_or(0);

                Ok(std::process::Output {
                    status: std::process::ExitStatus::from_raw(exit_code),
                    stdout,
                    stderr,
                })
            }
            StartExecResults::Detached => {
                bail!("Exec detached unexpectedly");
            }
        }
    }

    /// 应用资源限制
    pub async fn apply_limits(&self) -> Result<()> {
        // 更新容器配置
        let update_config = bollard::models::HostConfig {
            memory: self.capabilities.memory_budget.try_into().ok(),
            nano_cpus: (self.capabilities.num_cpus as i64 * 1_000_000_000).into(),
            ..Default::default()
        };

        self.client
            .update_container(&self.config.container_id, update_config)
            .await?;

        Ok(())
    }
}

impl RuntimeAdapter for DockerRuntime {
    fn name(&self) -> &str {
        "docker"
    }

    fn has_shell_access(&self) -> bool {
        self.capabilities.has_shell
    }

    fn has_filesystem_access(&self) -> bool {
        self.capabilities.has_filesystem
    }

    fn storage_path(&self) -> PathBuf {
        // 使用容器内的存储路径
        PathBuf::from("/var/lib/zeroclaw/storage")
    }

    fn supports_long_running(&self) -> bool {
        self.capabilities.has_long_running
    }

    fn memory_budget(&self) -> u64 {
        self.capabilities.memory_budget
    }

    fn build_shell_command(
        &self,
        command: &str,
        workspace_dir: &Path,
    ) -> anyhow::Result<tokio::process::Command> {
        // Docker 环境中使用 docker exec
        let mut cmd = tokio::process::Command::new("docker");
        cmd.arg("exec")
           .arg(&self.config.container_id)
           .arg("sh")
           .arg("-c")
           .arg(command)
           .current_dir(workspace_dir);

        Ok(cmd)
    }

    fn supports_networking(&self) -> bool {
        self.capabilities.has_networking
    }

    fn num_cpus(&self) -> usize {
        self.capabilities.num_cpus
    }
}
```

### 4.3 WASM 运行时 (wasm.rs - 简化版)

```rust
pub struct WasmRuntime {
    storage_path: PathBuf,
    memory_limit: u64,
    allowed_apis: Vec<String>,
}

impl WasmRuntime {
    pub fn new(storage_path: PathBuf, memory_limit_mb: u64) -> Self {
        Self {
            storage_path,
            memory_limit: memory_limit_mb * 1024 * 1024,
            allowed_apis: vec![
                "https://api.openai.com".to_string(),
                "https://api.anthropic.com".to_string(),
                // ... 其他允许的 API
            ],
        }
    }
}

impl RuntimeAdapter for WasmRuntime {
    fn name(&self) -> &str {
        "wasm"
    }

    fn has_shell_access(&self) -> bool {
        false  // WASM 不支持 Shell
    }

    fn has_filesystem_access(&self) -> bool {
        true  // 支持虚拟文件系统
    }

    fn storage_path(&self) -> PathBuf {
        self.storage_path.clone()
    }

    fn supports_long_running(&self) -> bool {
        false  // WASM 通常是事件驱动，不支持长运行
    }

    fn memory_budget(&self) -> u64 {
        self.memory_limit
    }

    fn build_shell_command(
        &self,
        _command: &str,
        _workspace_dir: &Path,
    ) -> anyhow::Result<tokio::process::Command> {
        bail!("Shell commands not supported in WASM runtime")
    }

    fn supports_networking(&self) -> bool {
        true  // WASM 支持网络 (通过 JS 互操作)
    }

    fn supports_threads(&self) -> bool {
        false  // WASM 单线程
    }

    fn num_cpus(&self) -> usize {
        1
    }
}
```

## 5. 运行时检测和选择

### 5.1 自动检测

```rust
pub fn detect_runtime(workspace_dir: &Path) -> Box<dyn RuntimeAdapter> {
    // 1. 检查是否在 Docker 容器中
    if std::path::Path::new("/.dockerenv").exists() {
        info!("Detected Docker environment");
        return Box::new(DockerRuntime::in_current_container());
    }

    // 2. 检查是否在 WASM 环境
    #[cfg(target_arch = "wasm32")]
    {
        info!("Running in WASM environment");
        return Box::new(WasmRuntime::default());
    }

    // 3. 默认为本地运行时
    info!("Using native runtime");
    Box::new(NativeRuntime::new(workspace_dir))
}
```

### 5.2 显式配置

```rust
pub fn create_runtime(
    config: &Config,
    workspace_dir: &Path,
) -> Result<Box<dyn RuntimeAdapter>> {
    match config.runtime.adapter.as_str() {
        "native" => Ok(Box::new(NativeRuntime::new(workspace_dir))),
        
        "docker" => {
            let docker_config = DockerConfig {
                container_id: config.runtime.container_id.clone()
                    .context("container_id required for Docker runtime")?,
                memory_limit: config.runtime.memory_limit,
                cpu_limit: config.runtime.cpu_limit,
                network_enabled: config.runtime.network_enabled,
                allowed_hosts: config.runtime.allowed_hosts.clone(),
                volume_mounts: config.runtime.volume_mounts.clone(),
            };
            
            let rt = tokio::runtime::Handle::current();
            let docker_rt = rt.block_on(DockerRuntime::new(docker_config))?;
            Ok(Box::new(docker_rt))
        }
        
        "wasm" => {
            #[cfg(target_arch = "wasm32")]
            {
                Ok(Box::new(WasmRuntime::new(
                    workspace_dir.join("storage"),
                    config.runtime.memory_limit.unwrap_or(256),
                )))
            }
            
            #[cfg(not(target_arch = "wasm32"))]
            {
                bail!("WASM runtime not supported on this platform")
            }
        }
        
        _ => bail!("Unknown runtime adapter: {}", config.runtime.adapter),
    }
}
```

## 6. 运行时适配层应用

### 6.1 Tool 执行适配

```rust
pub async fn execute_tool_with_runtime(
    tool: &dyn Tool,
    args: serde_json::Value,
    runtime: &dyn RuntimeAdapter,
    workspace_dir: &Path,
) -> Result<ToolResult> {
    // 检查运行时能力
    if tool.requires_shell() && !runtime.has_shell_access() {
        bail!(
            "Tool '{}' requires shell access but runtime does not support it",
            tool.name()
        );
    }

    if tool.requires_filesystem() && !runtime.has_filesystem_access() {
        bail!(
            "Tool '{}' requires filesystem access but runtime does not support it",
            tool.name()
        );
    }

    // 对于 Shell 工具，使用运行时构建命令
    if let Some(shell_tool) = tool.as_shell_tool() {
        let cmd = runtime.build_shell_command(
            &shell_tool.command,
            workspace_dir,
        )?;

        let output = cmd.output().await?;
        
        Ok(ToolResult {
            success: output.status.success(),
            output: String::from_utf8_lossy(&output.stdout).to_string(),
            error: String::from_utf8_lossy(&output.stderr).to_string(),
        })
    } else {
        // 纯 Rust 工具直接执行
        tool.execute(args).await
    }
}
```

### 6.2 Memory 后端适配

```rust
pub fn create_memory_backend(
    config: &MemoryConfig,
    runtime: &dyn RuntimeAdapter,
) -> Result<Box<dyn Memory>> {
    // 如果没有文件系统访问，强制使用内存后端
    if !runtime.has_filesystem_access() {
        info!("Runtime lacks filesystem access, using in-memory backend");
        return Ok(Box::new(InMemoryMemory::new()));
    }

    // 根据配置创建后端
    match config.backend.as_str() {
        "sqlite" => {
            let db_path = runtime.storage_path().join("memory.db");
            Ok(Box::new(SqliteMemory::new(&db_path)?))
        }
        
        "markdown" => {
            let dir = runtime.storage_path().join("memory");
            Ok(Box::new(MarkdownMemory::new(&dir)?))
        }
        
        _ => Ok(Box::new(InMemoryMemory::new())),
    }
}
```

## 7. 资源管理

### 7.1 内存预算管理

```rust
pub struct ResourceManager {
    runtime: Arc<dyn RuntimeAdapter>,
    current_usage: AtomicU64,
}

impl ResourceManager {
    pub fn new(runtime: Arc<dyn RuntimeAdapter>) -> Self {
        Self {
            runtime,
            current_usage: AtomicU64::new(0),
        }
    }

    /// 尝试分配内存
    pub fn try_allocate(&self, size: u64) -> bool {
        let budget = self.runtime.memory_budget();
        
        // 无限制
        if budget == 0 {
            return true;
        }

        // 检查是否超出预算
        let current = self.current_usage.load(Ordering::Relaxed);
        if current + size > budget {
            warn!(
                "Memory allocation would exceed budget: {} + {} > {}",
                current, size, budget
            );
            return false;
        }

        // 分配成功
        self.current_usage.fetch_add(size, Ordering::Relaxed);
        true
    }

    /// 释放内存
    pub fn deallocate(&self, size: u64) {
        self.current_usage.fetch_sub(size, Ordering::Relaxed);
    }

    /// 获取当前使用率
    pub fn usage_percent(&self) -> f64 {
        let budget = self.runtime.memory_budget();
        if budget == 0 {
            return 0.0;
        }

        let current = self.current_usage.load(Ordering::Relaxed);
        (current as f64 / budget as f64) * 100.0
    }
}
```

## 8. 错误处理

### 8.1 运行时错误类型

```rust
pub enum RuntimeError {
    ShellNotSupported(String),
    FilesystemNotSupported(String),
    ResourceLimitExceeded { resource: String, limit: u64, requested: u64 },
    ContainerNotFound(String),
    NetworkDisabled(String),
    UnsupportedOperation(String),
}

impl Display for RuntimeError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::ShellNotSupported(tool) => {
                write!(f, "Tool '{}' requires shell access which is not available", tool)
            }
            Self::ResourceLimitExceeded { resource, limit, requested } => {
                write!(
                    f,
                    "{} limit exceeded: requested {}, limit is {}",
                    resource, requested, limit
                )
            }
            _ => write!(f, "{:?}", self),
        }
    }
}
```

## 9. 测试策略

### 9.1 Mock 运行时

```rust
pub struct MockRuntime {
    pub name: String,
    pub has_shell: bool,
    pub has_filesystem: bool,
    pub storage_path: PathBuf,
    pub long_running: bool,
    pub memory_budget: u64,
    pub commands_executed: Mutex<Vec<String>>,
}

impl RuntimeAdapter for MockRuntime {
    fn name(&self) -> &str {
        &self.name
    }

    fn has_shell_access(&self) -> bool {
        self.has_shell
    }

    fn has_filesystem_access(&self) -> bool {
        self.has_filesystem
    }

    fn storage_path(&self) -> PathBuf {
        self.storage_path.clone()
    }

    fn supports_long_running(&self) -> bool {
        self.long_running
    }

    fn memory_budget(&self) -> u64 {
        self.memory_budget
    }

    fn build_shell_command(
        &self,
        command: &str,
        _workspace_dir: &Path,
    ) -> anyhow::Result<tokio::process::Command> {
        self.commands_executed.lock().unwrap().push(command.to_string());
        
        // 返回一个不执行的命令
        Ok(tokio::process::Command::new("echo"))
    }
}
```

## 10. 监控指标

### 10.1 关键指标

- 运行时类型分布
- 资源使用率 (内存/CPU)
- Shell 命令执行成功率
- 文件系统操作延迟
- 长运行进程存活时间
