# Tools 模块详细设计

## 1. 模块概述

Tools 模块实现了 ZeroClaw 的执行能力层，提供了 40+ 个内置工具，涵盖文件操作、Shell 执行、网络请求、浏览器自动化、记忆管理、SOP 工作流等功能。所有工具都遵循统一的 Trait 接口。

## 2. 架构设计

### 2.1 模块结构

src/tools/ 目录包含:
- mod.rs - 模块导出和工厂 (21.1KB)
- traits.rs - Tool Trait 定义 (3.2KB)
- schema.rs - JSON Schema 生成 (25.7KB)
- shell.rs - Shell 命令执行 (22.1KB)
- file_read.rs - 文件读取 (37.0KB)
- file_write.rs - 文件写入 (15.5KB)
- file_edit.rs - 文件编辑 (22.8KB)
- browser.rs - 浏览器自动化 (85.1KB)
- browser_open.rs - 打开网页 (15.8KB)
- web_search_tool.rs - 网络搜索 (11.0KB)
- web_fetch.rs - 网页抓取 (26.0KB)
- http_request.rs - HTTP 请求 (29.9KB)
- memory_store.rs - 记忆存储 (7.2KB)
- memory_recall.rs - 记忆召回 (5.0KB)
- memory_forget.rs - 记忆删除 (5.5KB)
- git_operations.rs - Git 操作 (28.2KB)
- cron_add.rs - 添加定时任务 (17.0KB)
- cron_list.rs - 列出定时任务 (2.7KB)
- cron_remove.rs - 删除定时任务 (6.3KB)
- schedule.rs - 调度工具 (25.6KB)
- delegate.rs - 任务委托 (35.2KB)
- composio.rs - Compos.io 集成 (66.4KB)
- content_search.rs - 内容搜索 (31.7KB)
- glob_search.rs - Glob 模式搜索 (13.7KB)
- image_info.rs - 图像信息 (16.5KB)
- pdf_read.rs - PDF 读取 (19.2KB)
- screenshot.rs - 截图 (11.3KB)
- pushover.rs - Pushover 通知 (13.5KB)
- proxy_config.rs - 代理配置 (18.2KB)
- model_routing_config.rs - 模型路由配置 (37.8KB)
- hardware_* - 硬件相关工具 (4 个文件)
- sop_* - SOP 工作流工具 (5 个文件)
- cli_discovery.rs - CLI 发现 (6.6KB)

总计 42 个工具文件。

### 2.2 设计模式

1. **命令模式**: 每个工具是一个独立的命令对象
2. **组合模式**: 复杂工具由多个简单工具组合
3. **责任链**: 工具调用链可以串联

## 3. 核心 Trait 定义

### 3.1 Tool Trait

```rust
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value;
    async fn execute(&self, args: serde_json::Value) -> Result<ToolResult>;
    
    fn spec(&self) -> ToolSpec {
        ToolSpec {
            name: self.name().to_string(),
            description: self.description().to_string(),
            parameters: self.parameters_schema(),
        }
    }
}
```

### 3.2 数据结构

**ToolResult**:
- success: bool
- output: String
- error: Option<String>

**ToolSpec**:
- name: String
- description: String
- parameters: serde_json::Value (JSON Schema)

## 4. 主要工具实现

### 4.1 Shell 工具

功能：执行 Shell 命令

安全特性:
- 白名单机制
- 工作目录限制
- 速率限制

### 4.2 文件操作工具

包括:
- FileReadTool: 读取文件
- FileWriteTool: 写入文件
- FileEditTool: 编辑文件 (diff/patch)

安全特性:
- 路径验证
- 文件大小限制
- 扩展名白名单

### 4.3 浏览器工具

最复杂的工具 (85.1KB)

支持操作:
- 导航到 URL
- 点击元素
- 填充表单
- 截图
- 提取内容
- 等待条件

基于 Playwright/ferrum 实现。

### 4.4 网络工具

包括:
- WebSearchTool: 搜索引擎集成 (Google/Bing)
- WebFetchTool: 网页抓取
- HttpRequestTool: 自定义 HTTP 请求

### 4.5 记忆工具

包括:
- MemoryStoreTool: 存储记忆
- MemoryRecallTool: 召回记忆
- MemoryForgetTool: 删除记忆

### 4.6 Git 工具

支持常见 Git 操作:
- status, add, commit
- push, pull
- log, diff

### 4.7 Cron 工具

定时任务管理:
- CronAddTool: 添加任务
- CronListTool: 列出任务
- CronRemoveTool: 删除任务

### 4.8 Composio 工具

集成 Compos.io 平台，可访问上千个 SaaS 应用。

## 5. 安全机制

### 5.1 输入验证

- 路径遍历防护
- 命令注入防护
- 参数类型检查

### 5.2 访问控制

- 白名单机制
- 工作目录限制
- 权限检查

### 5.3 速率限制

使用 governor crate 实现令牌桶算法。

### 5.4 审计日志

记录所有工具调用:
```rust
tracing::info!(
    tool = %self.name(),
    args = %serde_json::to_string(&args)?,
    success = %result.success,
    "Tool executed"
);
```

## 6. 错误处理

### 6.1 错误类型

- InvalidArguments: 无效参数
- PermissionDenied: 权限拒绝
- ResourceNotFound: 资源未找到
- ExecutionFailed: 执行失败
- Timeout: 超时

### 6.2 错误返回格式

```rust
Ok(ToolResult {
    success: false,
    output: String::new(),
    error: Some("Error message".to_string()),
})
```

## 7. 性能优化

### 7.1 连接池

HTTP 工具使用 reqwest 连接池。

### 7.2 缓存

使用 moka crate 实现本地缓存。

### 7.3 并发

异步执行，支持批量操作。

## 8. 测试策略

### 8.1 单元测试

测试单个工具的逻辑正确性。

### 8.2 集成测试

在沙箱环境中测试真实场景。

### 8.3 Mock 工具

用于测试的模拟实现。

## 9. 扩展指南

### 9.1 添加新工具步骤

1. 创建 src/tools/my_tool.rs
2. 实现 Tool trait
3. 在 mod.rs 中注册
4. 编写 JSON Schema
5. 添加测试

### 9.2 最小实现示例

```rust
pub struct MyTool;

#[async_trait]
impl Tool for MyTool {
    fn name(&self) -> &str { "my_tool" }
    fn description(&self) -> &str { "Does something" }
    
    fn parameters_schema(&self) -> serde_json::Value {
        json!({
            "type": "object",
            "properties": {
                "param1": {"type": "string"}
            },
            "required": ["param1"]
        })
    }
    
    async fn execute(&self, args: serde_json::Value) -> Result<ToolResult> {
        // 实现逻辑
        Ok(ToolResult { success: true, output: "Done".into(), error: None })
    }
}
```

## 10. 监控指标

### 10.1 关键指标

- 调用频率
- 平均执行时间
- 成功率
- 错误率

### 10.2 日志级别

- DEBUG: 工具被调用
- INFO: 工具完成
- ERROR: 工具失败

## 11. 与其他模块的接口

### 11.1 与 Agent 模块

Agent 通过 ToolCall 调用工具。

### 11.2 与 Security 模块

Security 提供审计和策略检查。

### 11.3 与 Memory 模块

记忆工具直接调用 Memory trait。
