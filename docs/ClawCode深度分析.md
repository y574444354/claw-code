# Claw Code：架构全景解析

> 报告日期：2026-04-05
> 代码库：`claw-code`（Rust ~31K 行 + Python ~1.7K 行）
> 版本：0.1.0（早期实现，2026-03-31 创建）

---

## 目录

1. [核心技术栈](#11-核心技术栈)
2. [核心架构设计](#12-核心架构设计)
   - [1.2.1 Agent Loop](#121-agent-loop-泛型状态机)
   - [1.2.2 工具体系](#122-工具体系内置工具的注册逻辑)
   - [1.2.3 MCP 集成](#123-mcp-集成多传输协议支持)
   - [1.2.4 Skill 系统](#124-skill-系统斜杠命令固化工作流)
   - [1.2.5 上下文压缩](#125-上下文压缩规则驱动的摘要系统)
   - [1.2.6 Hook 系统](#126-hook-系统工具执行拦截器)
   - [1.2.7 流式执行引擎](#127-流式执行引擎sse-解析与事件分发)
3. [高级特性](#13-高级特性)
   - [1.3.1 权限系统](#131-权限系统五级有序模型)
   - [1.3.2 OAuth 2.0 PKCE 认证](#132-oauth-20-pkce-认证)
   - [1.3.3 配置系统](#133-配置系统三层来源合并)
   - [1.3.4 沙箱与隔离](#134-沙箱与隔离)
4. [其他功能模块](#14-其他功能模块)
   - [1.4.1 Token 计量与成本估算](#141-token-计量与成本估算)
   - [1.4.2 系统提示构建](#142-系统提示构建)
   - [1.4.3 斜杠命令系统](#143-斜杠命令系统)
   - [1.4.4 Python 移植工作空间](#144-python-移植工作空间)
   - [1.4.5 兼容性层](#145-兼容性层compat-harness)
5. [架构亮点与局限](#15-架构亮点与局限)

---

读完 Claw Code 的源码，最深的感受是：这是一套**为正确性和可测试性而设计的架构**，而不是功能堆砌。泛型运行时、trait 驱动的工具执行、有序枚举的权限模型——每个设计决策都在回答同一个问题：**怎么让 AI 代理在 Rust 的编译期约束下安全、可验证地执行任务**。要理解它的强处和局限，必须从整体结构开始看。

---

### 1.1 核心技术栈

| 层级 | 技术选型 | 说明 |
|------|---------|------|
| **运行时** | 原生 Rust 二进制 | 无 GC，无运行时依赖，`cargo build --release` 产出单一静态二进制 |
| **语言** | Rust 2021 Edition | `unsafe_code = "forbid"`，全局禁止 unsafe，Clippy pedantic 级别 |
| **构建产物** | 单一静态二进制 | 零运行时依赖（无 Node.js/Bun），可直接分发 |
| **异步运行时** | Tokio（多线程） | HTTP 客户端、SSE 流解析使用 async/await |
| **同步核心** | 标准库阻塞 I/O | `ConversationRuntime` 主循环同步阻塞，不依赖 async |
| **HTTP 客户端** | `reqwest`（rustls-tls） | 不依赖系统 OpenSSL |
| **序列化** | `serde` + `serde_json` | 全类型安全，编译期 schema 验证 |
| **Schema 验证** | 无运行时验证器 | Rust 类型系统即是 schema，无需 Zod 等 |
| **LLM 接入** | 自研 Provider 枚举 | Anthropic API + xAI + OpenAI 兼容，自研 SSE 解析器 |
| **扩展协议** | MCP（JSON-RPC 2.0） | 支持 stdio/SSE/HTTP/WebSocket/SDK/ManagedProxy |
| **终端 UI** | 自研 REPL | 无 React/Ink，纯 ANSI 转义码 + Markdown 流式渲染 |
| **工作空间管理** | Cargo workspace | 9 个 crate，crate 边界即 API 边界 |
| **Python 参考实现** | 纯标准库 | 无外部依赖，用于移植参考和 parity 审计 |

```bash
# 构建
cd rust && cargo build --release
# → rust/target/release/claw（单一静态二进制）

# 格式化 + 静态检查
cargo fmt && cargo clippy --all-targets -- -D warnings

# 单元测试
cargo test --workspace
```

---

### 1.2 核心架构设计

#### 1.2.1 Agent Loop：泛型状态机

Agent Loop 是 Claw Code 所有能力的执行引擎，位于 `rust/crates/runtime/src/conversation.rs`（801 行）。理解它需要从两个维度切入：泛型设计哲学、以及实际的 ReAct 执行路径。

---

##### 泛型解耦：`ConversationRuntime<C, T>`

这是 Claw Code 架构里最重要的设计决策。运行时不直接依赖任何具体的 API 客户端或工具执行器，而是通过两个泛型参数完全解耦：

```rust
// conversation.rs:91-100
pub struct ConversationRuntime<C, T> {
    session: Session,
    api_client: C,           // 实现 ApiClient trait
    tool_executor: T,        // 实现 ToolExecutor trait
    permission_policy: PermissionPolicy,
    system_prompt: Vec<String>,
    max_iterations: usize,
    usage_tracker: UsageTracker,
    hook_runner: HookRunner,
}

// trait 定义（conversation.rs:31-37）
pub trait ApiClient {
    fn stream(&mut self, request: ApiRequest) -> Result<Vec<AssistantEvent>, RuntimeError>;
}

pub trait ToolExecutor {
    fn execute(&mut self, tool_name: &str, input: &str) -> Result<String, ToolError>;
}
```

这个设计的核心价值在于：**测试时可以用 mock 实现替换真实 API 和工具执行器，而不改动任何业务逻辑**。Rust 的单态化（monomorphization）让这套泛型在运行时零开销——编译器为每组 `<C, T>` 生成独立的实现，没有虚表开销。

---

##### ReAct 模式的具体实现

ReAct（Reason → Act → Observe）不是文档里的概念，是 `run_turn()` 方法里 `loop { }` 每一轮的实际执行路径（`conversation.rs:153-263`）：

**Reason**：调用 LLM，收集事件流

```rust
// 每次循环向 API 发送完整会话
let request = ApiRequest {
    system_prompt: self.system_prompt.clone(),
    messages: self.session.messages.clone(),
};
let events = self.api_client.stream(request)?;

// 将事件流折叠成结构化消息
let (assistant_message, usage) = build_assistant_message(events)?;
```

`build_assistant_message()`（`conversation.rs:291-340`）遍历事件流，把 `TextDelta` 拼接成文本块，把 `ToolUse` 事件直接插入 block 列表，等待 `MessageStop` 事件确认流结束。**整个过程等待完整响应**，不在流式接收过程中触发工具执行——这与 Claude Code 的 `StreamingToolExecutor` 设计截然不同。

**Act**：串行执行工具

```rust
// 提取所有待执行的工具调用
let pending_tool_uses = assistant_message.blocks.iter()
    .filter_map(|block| match block {
        ContentBlock::ToolUse { id, name, input } => Some(...),
        _ => None,
    })
    .collect::<Vec<_>>();

// 顺序执行，无并发
for (tool_use_id, tool_name, input) in pending_tool_uses {
    let permission_outcome = self.permission_policy.authorize(...);
    let pre_hook_result = self.hook_runner.run_pre_tool_use(&tool_name, &input);
    let (output, is_error) = match self.tool_executor.execute(&tool_name, &input) {
        Ok(output) => (output, false),
        Err(error)  => (error.to_string(), true),
    };
    let post_hook_result = self.hook_runner.run_post_tool_use(...);
    self.session.messages.push(result_message);
}
```

每个工具调用经过完整的权限检查 → pre-hook → 执行 → post-hook 链路。**工具串行执行**：一个工具未完成前，下一个不会启动。

**Observe**：工具结果注入下一轮

```rust
// 工具结果以 tool_result ContentBlock 追加到会话
// 下一次循环的 session.messages 已包含这些结果
self.session.messages.push(result_message.clone());
```

一个完整的两轮对话消息结构：

```
Turn 1 发给 LLM：
  [user: "帮我查 src/auth.rs 有没有问题"]

Turn 1 LLM 输出：
  [assistant: "我来读一下... [ToolUse id=abc name=read_file]"]

Turn 2 发给 LLM（Observe 注入后）：
  [user: "帮我查..."]
  [assistant: "我来读一下... [ToolUse id=abc]"]
  [tool: { tool_use_id: "abc", output: "文件内容..." }]
                                    ↑ Observe：结果在这里注入

Turn 2 LLM 输出：
  [assistant: "第 47 行有个未过滤的用户输入..."]
  → pending_tool_uses 为空 → break → 返回 TurnSummary
```

---

##### 循环终止逻辑

```rust
// conversation.rs:166-172
loop {
    iterations += 1;
    if iterations > self.max_iterations {
        return Err(RuntimeError::new(
            "conversation loop exceeded the maximum number of iterations",
        ));
    }
    // ...
    if pending_tool_uses.is_empty() {
        break;  // LLM 不再调用工具 → 本轮结束
    }
}
```

终止条件只有两种：超过迭代上限（错误）或 LLM 不再发出工具调用（正常完成）。**没有 PTL 恢复、没有 max_tokens 升级、没有死循环检测**——这些是当前版本最显著的功能缺口。

---

##### `TurnSummary`：轮次结果

```rust
// conversation.rs:83-89
pub struct TurnSummary {
    pub assistant_messages: Vec<ConversationMessage>,
    pub tool_results: Vec<ConversationMessage>,
    pub iterations: usize,
    pub usage: TokenUsage,  // 累积 token 使用量
}
```

调用方通过 `TurnSummary` 获取本次轮转的所有信息：助手消息、工具结果、迭代次数、token 消耗。`usage` 字段来自 `UsageTracker`，跨轮次累积，不会因为压缩而丢失。

---

#### 1.2.2 工具体系：内置工具的注册逻辑

##### ToolSpec：工具的元数据结构

每个工具通过 `ToolSpec` 结构描述（`tools/src/lib.rs:51-57`）：

```rust
pub struct ToolSpec {
    pub name: &'static str,          // 工具名（编译期常量）
    pub description: &'static str,   // 发给 LLM 的描述
    pub input_schema: Value,         // JSON Schema（serde_json::Value）
    pub required_permission: PermissionMode,  // 执行所需最低权限
}
```

与 Claude Code 的 `Tool<Input, Output>` 泛型接口相比，Claw Code 的 `ToolSpec` 更简洁——**没有并发安全标记、没有只读标记、没有结果持久化阈值**。这是功能覆盖率的差距，也是设计简洁性的体现。

##### GlobalToolRegistry：运行时工具池

```rust
// tools/src/lib.rs:59-92
pub struct GlobalToolRegistry {
    plugin_tools: Vec<PluginTool>,  // 插件提供的工具
}

impl GlobalToolRegistry {
    pub fn builtin() -> Self { ... }  // 仅内置工具

    pub fn with_plugin_tools(plugin_tools: Vec<PluginTool>) -> Result<Self, String> {
        // 检查插件工具名与内置工具不冲突
        // 检查插件工具名无重复
        // 构建完整工具池
    }
}
```

工具池由内置工具（`mvp_tool_specs()`）和插件工具合并而成，名称冲突在构建时即报错——**Rust 的编译期安全延伸到了运行时配置验证**。

##### 内置工具清单（MVP）

`mvp_tool_specs()` 注册了以下内置工具：

| 工具名 | 功能 | 所需权限 |
|--------|------|---------|
| `read_file` | 读文件（支持行偏移/限制） | ReadOnly |
| `write_file` | 写文件（全覆盖） | WorkspaceWrite |
| `edit_file` | 字符串替换编辑（old→new） | WorkspaceWrite |
| `glob_search` | 文件路径 glob 匹配 | ReadOnly |
| `grep_search` | 内容正则搜索 | ReadOnly |
| `bash` | Shell 命令执行 | DangerFullAccess |
| `web_fetch` | HTTP GET 网页内容 | WorkspaceWrite |
| `web_search` | 搜索引擎查询 | WorkspaceWrite |
| `todo_write` | 任务列表管理 | ReadOnly |
| `config_read` / `config_write` | 配置读写 | ReadOnly / WorkspaceWrite |
| `replica` | 子代理克隆 | DangerFullAccess |
| `skill` | Skill 调用（斜杠命令） | ReadOnly |
| `agent` | Agent 分叉执行 | DangerFullAccess |

工具名别名（`tools/src/lib.rs:110-117`）：`read` → `read_file`，`write` → `write_file`，`edit` → `edit_file`，`glob` → `glob_search`，`grep` → `grep_search`。CLI 的 `--allowed-tools` 参数接受别名形式。

##### 工具执行链路

工具执行路径从 `ConversationRuntime` 的 `run_turn()` 进入（`conversation.rs:200-254`），经过完整的检查链：

```
[LLM 产出 ToolUse block]
  ↓
1. permission_policy.authorize(tool_name, input, prompter)
   → PermissionOutcome::Allow / Deny
  ↓
2. hook_runner.run_pre_tool_use(tool_name, input)
   → HookRunResult { denied, messages }
   → 如果 denied → 返回错误消息，跳过执行
  ↓
3. tool_executor.execute(tool_name, input)
   → Ok(output) / Err(ToolError)
  ↓
4. hook_runner.run_post_tool_use(tool_name, input, output, is_error)
   → 后置 hook 可追加消息、标记错误
  ↓
5. ConversationMessage::tool_result(tool_use_id, tool_name, output, is_error)
   → 追加到 session.messages
```

**没有结果持久化**（超大结果直接回灌上下文），**没有并发调度**（逐个顺序执行）。这两点是当前版本与 Claude Code 最大的工具执行差距。

---

#### 1.2.3 MCP 集成：多传输协议支持

MCP 对 Claw Code 的意义和对 Claude Code 一样：把外部系统转换成 Agent Loop 能统一调度的能力单元。配置解析（`runtime/src/config.rs`）、客户端引导（`runtime/src/mcp_client.rs`）、协议通信（`runtime/src/mcp.rs`、`mcp_stdio.rs`）三层分离。

---

##### 配置：六种传输类型

`McpServerConfig` 枚举（`config.rs:86-93`）支持六种传输：

```rust
pub enum McpServerConfig {
    Stdio(McpStdioServerConfig),         // 本地子进程，JSON-RPC via stdin/stdout
    Sse(McpRemoteServerConfig),          // Server-Sent Events（长连接）
    Http(McpRemoteServerConfig),         // Streamable HTTP
    Ws(McpWebSocketServerConfig),        // WebSocket 双向通信
    Sdk(McpSdkServerConfig),             // 进程内嵌入（零网络开销）
    ManagedProxy(McpManagedProxyServerConfig),  // 托管代理（URL + ID 方式）
}
```

配置按三种来源分级（`ConfigSource`：`User` < `Project` < `Local`），同名服务器高优先级覆盖低优先级。

##### 工具发现：从 MCP server 到工具池

`McpClientBootstrap`（`mcp_client.rs:48-68`）在启动时为每个 MCP server 生成引导信息：

```rust
pub struct McpClientBootstrap {
    pub server_name: String,
    pub normalized_name: String,   // 规范化名称（用于工具前缀）
    pub tool_prefix: String,       // "mcp__<server>__"
    pub signature: Option<String>, // server 配置签名（用于缓存失效）
    pub transport: McpClientTransport,
}
```

工具名规范化为 `mcp__<server>__<tool>`，与内置工具一起注入工具池（`assembleToolDefinitions()`），LLM 看到的是统一的工具列表。

##### stdio 传输的协议流程

`mcp_stdio.rs` 实现完整的 JSON-RPC 2.0 over stdio 协议：

```
1. 生成子进程（如 `npx @modelcontextprotocol/server-filesystem`）
2. initialize：
   { "method": "initialize", "params": { "protocolVersion": "2024-11-05", ... } }
   ← { "result": { "capabilities": {...}, "serverInfo": {...} } }
3. initialized 通知（无响应）
4. tools/list：
   { "method": "tools/list" }
   ← { "result": { "tools": [{ "name": "...", "description": "...", "inputSchema": {...} }] } }
5. tools/call（工具调用）：
   { "method": "tools/call", "params": { "name": "...", "arguments": {...} } }
   ← { "result": { "content": [...], "isError": false } }
```

子进程 stderr 重定向为管道，避免污染 TUI 输出。

---

#### 1.2.4 Skill 系统：斜杠命令固化工作流

Skill 是 Claw Code 把重复工作流固化为可调用资产的机制。用户或 AI 调用 `/skill-name` 时，`skill` 工具从文件系统加载对应 Markdown，注入当前会话上下文。

##### 两种发现路径

**本地文件型**：从 `.claw/skills/*.md` 加载。Skill 内容注入当前上下文，影响后续 AI 行为：

```rust
// tools/src/lib.rs：skill 工具执行逻辑
// 1. 在 .claw/skills/ 目录查找匹配的 .md 文件
// 2. 读取内容，替换变量（$ARGUMENTS → 用户传入参数）
// 3. 通过 tool_result 把 skill 内容返回给 LLM
// 4. LLM 在后续对话中依据 skill 约束执行
```

**MCP Prompt 型**：来自 MCP server 暴露的 prompts，与本地 Skill 合并到同一个命令列表，对 AI 完全透明。

##### Skill 与 MCP 工具的边界

| | Skill | MCP 工具 |
|--|--|--|
| 执行载体 | 提示词注入（返回 Markdown 文本） | MCP 协议 API 调用 |
| 结果形式 | 影响后续 AI 行为 | 返回结构化数据 |
| 权限需求 | ReadOnly（仅读文件） | 由工具决定 |
| 适合场景 | 流程约束、工作方法、检查清单 | 外部系统读写、数据查询 |

---

#### 1.2.5 上下文压缩：规则驱动的摘要系统

上下文压缩位于 `runtime/src/compact.rs`（703 行含测试），是代码库中测试最完善的模块之一（7 个单元测试覆盖核心路径）。

---

##### CompactionConfig：两个关键阈值

```rust
// compact.rs:8-21
pub struct CompactionConfig {
    pub preserve_recent_messages: usize,  // 保留最新 N 条消息（默认 4）
    pub max_estimated_tokens: usize,      // 估算 token 超过此值触发压缩（默认 10,000）
}
```

两个参数完全控制压缩行为：保留多少近期消息、以及触发阈值。

##### Token 估算：len/4 启发式

```rust
// compact.rs:391-403
fn estimate_message_tokens(message: &ConversationMessage) -> usize {
    message.blocks.iter().map(|block| match block {
        ContentBlock::Text { text }           => text.len() / 4 + 1,
        ContentBlock::ToolUse { name, input } => (name.len() + input.len()) / 4 + 1,
        ContentBlock::ToolResult { tool_name, output, .. } => (tool_name.len() + output.len()) / 4 + 1,
    }).sum()
}
```

用字符数除以 4 估算 token 数，是业界常用的粗略启发式（英文文本约 1 token/4 字符）。**没有接入真实的 tokenizer**——中文、代码等密集内容估算误差可能超过 50%。

##### 单次压缩执行路径

`compact_session()`（`compact.rs:89-131`）：

```rust
pub fn compact_session(session: &Session, config: CompactionConfig) -> CompactionResult {
    // 1. should_compact() → 计算可压缩段的估算 token 数
    if !should_compact(session, config) {
        return CompactionResult { removed_message_count: 0, ... }
    }

    // 2. 提取已有压缩摘要（支持递进合并）
    let existing_summary = session.messages.first()
        .and_then(extract_existing_compacted_summary);

    // 3. 确定保留范围：最新 preserve_recent_messages 条消息原样保留
    let keep_from = session.messages.len().saturating_sub(config.preserve_recent_messages);
    let removed = &session.messages[compacted_prefix_len..keep_from];
    let preserved = session.messages[keep_from..].to_vec();

    // 4. 规则生成摘要（无 LLM 调用）
    let summary = merge_compact_summaries(existing_summary, &summarize_messages(removed));

    // 5. 用 System 消息替换旧消息，保留最近 N 条原始消息
    let mut compacted_messages = vec![ConversationMessage {
        role: MessageRole::System,
        blocks: vec![ContentBlock::Text { text: continuation }],
        usage: None,
    }];
    compacted_messages.extend(preserved);
}
```

**关键特性**：压缩本身**不调用 LLM**，完全基于规则。这是与 CoStrict Session Compaction（需要一次 LLM 调用）的根本差异。

##### 摘要内容：`summarize_messages()` 生成的结构

`summarize_messages()` 扫描被压缩的消息，生成结构化摘要（`compact.rs:143-228`）：

```xml
<summary>
Conversation summary:
- Scope: {N} 条消息 (user={}, assistant={}, tool={})
- Tools mentioned: {工具名列表，去重排序}
- Recent user requests:
  - {最近3条用户消息，截断到160字符}
- Pending work:
  - {含 todo/next/pending/follow up/remaining 关键词的文本}
- Key files referenced: {提到的文件路径，含特定扩展名，最多8个}
- Current work: {最新非空助手消息，截断到200字符}
- Key timeline:
  - user: {消息内容摘要}
  - assistant: {消息内容摘要}
  - tool: tool_result {工具名}: {输出摘要}
</summary>
```

##### 递进压缩：merge_compact_summaries()

多次压缩时，新摘要不会覆盖旧摘要，而是合并（`compact.rs:230-263`）：

```rust
// 二次压缩时，输出结构：
// <summary>
// Conversation summary:
// - Previously compacted context:
//   {从旧摘要提取的非时间线高亮}
// - Newly compacted context:
//   {新一轮压缩的高亮}
// - Key timeline:
//   {新一轮压缩的时间线}
// </summary>
```

这确保了历史压缩的重要信息（如"之前决定使用 Drizzle ORM"）不会在二次压缩时丢失。

---

#### 1.2.6 Hook 系统：工具执行拦截器

Hook 解决的是一个工程实际问题：**AI 修改文件之后，格式化、类型检查、lint 怎么自动跟上**。Claw Code 的 Hook 系统位于 `runtime/src/hooks.rs`（358 行含测试），是为数不多的**真正执行外部命令**的模块。

---

##### 两种 Hook 事件

```rust
// hooks.rs:9-11
pub enum HookEvent {
    PreToolUse,   // 工具调用前：可拦截、修改行为
    PostToolUse,  // 工具调用后：格式化、检查、通知
}
```

配置示例（`.claw.json`）：

```json
{
    "hooks": {
        "preToolUse": ["scripts/audit-tool.sh"],
        "postToolUse": ["prettier --write {FILE}", "cargo fmt"]
    }
}
```

##### 三态退出码协议

Hook 命令通过 exit code 返回决策（`hooks.rs:183-212`）：

```rust
match output.status.code() {
    Some(0) => HookCommandOutcome::Allow { message },  // 允许，stdout 作为附加信息
    Some(2) => HookCommandOutcome::Deny  { message },  // 拒绝，终止工具执行
    Some(_) => HookCommandOutcome::Warn  { message },  // 警告，工具继续执行
    None    => HookCommandOutcome::Warn  { ... },      // 信号终止，警告继续
}
```

- **exit 0**：允许，stdout 被追加到工具结果作为附加上下文
- **exit 2**：拒绝，工具执行被取消，LLM 收到拒绝原因
- **其他非零**：警告，工具执行继续，警告信息注入结果

##### 环境变量注入

Hook 进程以 JSON payload 作为 stdin，同时通过环境变量传递结构化信息（`hooks.rs:169-176`）：

```bash
# stdin: JSON payload
{
  "hook_event_name": "PreToolUse",
  "tool_name": "bash",
  "tool_input": { "command": "rm -rf /tmp/old" },
  "tool_input_json": "{\"command\":\"rm -rf /tmp/old\"}",
  "tool_output": null,
  "tool_result_is_error": false
}

# 环境变量
HOOK_EVENT=PreToolUse
HOOK_TOOL_NAME=bash
HOOK_TOOL_INPUT={"command":"rm -rf /tmp/old"}
HOOK_TOOL_IS_ERROR=0
HOOK_TOOL_OUTPUT=         # PostToolUse 时填充
```

##### 串行执行策略

多个 hook 命令串行执行，第一个 Deny 立即中止（`hooks.rs:128-161`）：

```rust
for command in commands {
    match Self::run_command(command, ...) {
        HookCommandOutcome::Deny { message } => {
            messages.push(message);
            return HookRunResult { denied: true, messages };  // 立即返回，后续 hook 不执行
        }
        HookCommandOutcome::Allow { message } => { ... }
        HookCommandOutcome::Warn  { message } => { ... }
    }
}
```

**跨平台**：Linux/macOS 用 `sh -lc`，Windows 用 `cmd /C`（`hooks.rs:239-255`）。

---

#### 1.2.7 流式执行引擎：SSE 解析与事件分发

Claw Code 的 API 接入是异步的（Tokio），但工具执行是同步阻塞的。两者通过 `AssistantEvent` 枚举桥接：

```rust
// conversation.rs:20-29
pub enum AssistantEvent {
    TextDelta(String),           // 增量文本（用于流式渲染）
    ToolUse { id, name, input }, // 工具调用声明
    Usage(TokenUsage),           // token 使用量
    MessageStop,                 // 流结束信号
}
```

##### 自研 SSE 解析器

Claw Code 没有使用任何第三方 SSE 库，在 `api/src/sse.rs` 和 `runtime/src/sse.rs` 中完整实现了 Anthropic API SSE 事件到 `AssistantEvent` 的映射：

```
content_block_start  (type=text)       → 准备接收 TextDelta
content_block_delta  (type=text_delta) → AssistantEvent::TextDelta
content_block_start  (type=tool_use)   → 记录工具 id + name
content_block_delta  (type=input_json_delta) → 累积 tool input JSON
content_block_stop                     → 完成当前 block
                                         → AssistantEvent::ToolUse { id, name, input }
message_delta        (usage)           → AssistantEvent::Usage
message_stop                           → AssistantEvent::MessageStop
```

完整的 SSE 解析是自研的优势——不依赖 SDK，可以精确控制每个事件的处理逻辑，也方便添加新的事件类型支持。

##### OpenAI 兼容传输

xAI 和 OpenAI 兼容接口通过 `api/src/providers/openai_compat.rs` 实现。OpenAI 的 `chat.completions` streaming 格式与 Anthropic API SSE 格式不同，通过独立解析器统一归一到 `AssistantEvent` 枚举：

```rust
// choices[0].delta.content → TextDelta
// choices[0].delta.tool_calls → ToolUse
// usage → Usage
```

---

### 1.3 高级特性

#### 1.3.1 权限系统：五级有序模型

`runtime/src/permissions.rs`（232 行含测试）实现了 5 种权限模式。与 Claude Code 的 6 种模式（枚举字符串）不同，Claw Code 使用**有序枚举**（`PartialOrd`）：

```rust
// permissions.rs:3-10
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub enum PermissionMode {
    ReadOnly,           // 只读，所有写操作拒绝
    WorkspaceWrite,     // 工作区写权限（文件读写）
    DangerFullAccess,   // 完全访问（含 Shell、网络等）
    Prompt,             // 每次提示用户决定
    Allow,              // 无条件放行（绕过所有检查）
}
```

有序枚举的核心优势：**`current_mode >= required_mode` 即放行**，授权逻辑可以用一行比较完成，无需复杂的 if-else 分支：

```rust
// permissions.rs:88-133：authorize() 完整逻辑
pub fn authorize(&self, tool_name, input, prompter) -> PermissionOutcome {
    let current  = self.active_mode();
    let required = self.required_mode_for(tool_name);

    // 明确放行：Allow 模式或当前模式已满足需求
    if current == Allow || current >= required {
        return PermissionOutcome::Allow;
    }

    // 提示用户：Prompt 模式，或 WorkspaceWrite 升级到 DangerFullAccess 时
    if current == Prompt
        || (current == WorkspaceWrite && required == DangerFullAccess)
    {
        return match prompter {
            Some(p) => match p.decide(&request) {
                Allow => PermissionOutcome::Allow,
                Deny  => PermissionOutcome::Deny { reason },
            },
            None => PermissionOutcome::Deny { reason: "...requires approval..." },
        };
    }

    // 默认拒绝
    PermissionOutcome::Deny { reason: "...requires {required} permission..." }
}
```

**特别设计**：`WorkspaceWrite` → `DangerFullAccess` 的升级路径会触发用户提示，而不是直接拒绝。这符合最小惊讶原则：用户选择了"工作区写"模式，遇到需要 Shell 执行的工具时，可以逐个决定是否放行。

##### 工具级权限需求

`PermissionPolicy` 维护一个 `BTreeMap<String, PermissionMode>`，声明每个工具所需的最低权限（`permissions.rs:49-53`）：

```rust
pub struct PermissionPolicy {
    active_mode: PermissionMode,
    tool_requirements: BTreeMap<String, PermissionMode>,
}
```

未在 map 中注册的工具默认需要 `DangerFullAccess`（保守策略）：

```rust
pub fn required_mode_for(&self, tool_name: &str) -> PermissionMode {
    self.tool_requirements.get(tool_name)
        .copied()
        .unwrap_or(PermissionMode::DangerFullAccess)  // 未知工具视为最高风险
}
```

##### 完整的单元测试覆盖

权限模块有 5 个单元测试覆盖关键路径（`permissions.rs:137-231`）：

| 测试名 | 覆盖场景 |
|-------|---------|
| `allows_tools_when_active_mode_meets_requirement` | 模式满足需求时直接放行 |
| `denies_read_only_escalations_without_prompt` | ReadOnly 不能升级到 Write/DangerFull |
| `prompts_for_workspace_write_to_danger_full_access_escalation` | WW→DFA 路径触发提示 |
| `honors_prompt_rejection_reason` | 用户拒绝时传递具体原因 |
| `allows_when_active_mode_is_allow` | Allow 模式绕过所有检查 |

---

#### 1.3.2 OAuth 2.0 PKCE 认证

Claw Code 实现了完整的 OAuth 2.0 PKCE（Proof Key for Code Exchange）认证流程，位于 `runtime/src/oauth.rs`。

##### PKCE 流程

```rust
// oauth.rs：核心数据结构
pub struct PkceCodePair {
    pub verifier: String,       // 随机生成的 code_verifier
    pub challenge: String,      // SHA-256(verifier) base64url 编码
    pub challenge_method: PkceChallengeMethod,  // S256
}

pub struct OAuthAuthorizationRequest {
    pub authorize_url: String,
    pub client_id: String,
    pub redirect_uri: String,   // http://localhost:4545/callback
    pub scopes: Vec<String>,
    pub state: String,          // 防 CSRF 随机状态
    pub code_challenge: String,
    pub code_challenge_method: PkceChallengeMethod,
    pub extra_params: BTreeMap<String, String>,
}
```

执行流程（`claw-cli/src/main.rs`）：

```
1. generate_pkce_pair()  → 生成 code_verifier + SHA-256 challenge
2. generate_state()      → 生成随机 CSRF state
3. 构建授权 URL          → 打开系统浏览器
4. TcpListener::bind(4545) → 监听本地回调
5. 解析回调参数           → parse_oauth_callback_request_target()
6. 验证 state 匹配        → 防止 CSRF
7. 交换 authorization_code → POST /token，携带 code_verifier
8. save_oauth_credentials() → 保存 access_token + refresh_token
```

##### 令牌存储

令牌以 JSON 格式保存在 `~/.claw/credentials.json`（与 Claude Code 的系统 keychain 存储不同，Claw Code 使用明文 JSON 文件，安全性较低但跨平台兼容性更好）。

---

#### 1.3.3 配置系统：三层来源合并

`runtime/src/config.rs`（1294 行）实现了三层配置合并（`User` < `Project` < `Local`）：

```rust
pub enum ConfigSource {
    User,     // ~/.claw/settings.json（用户全局）
    Project,  // {project}/.claw.json（项目级）
    Local,    // {project}/.claw.local.json（本地私有，不提交 git）
}
```

`RuntimeConfig` 通过 `BTreeMap<String, JsonValue>` 存储合并后的配置：

```rust
pub struct RuntimeConfig {
    merged: BTreeMap<String, JsonValue>,  // 按 ConfigSource 优先级合并
    loaded_entries: Vec<ConfigEntry>,      // 已加载的配置文件列表
    feature_config: RuntimeFeatureConfig,  // 解析后的运行时特性配置
}
```

`RuntimeFeatureConfig` 是从 JSON 解析的强类型运行时配置（`config.rs:47-56`）：

```rust
pub struct RuntimeFeatureConfig {
    hooks: RuntimeHookConfig,         // pre/post tool use hooks
    plugins: RuntimePluginConfig,     // 插件目录、注册表
    mcp: McpConfigCollection,         // MCP server 集合
    oauth: Option<OAuthConfig>,       // OAuth 配置
    model: Option<String>,            // 默认模型
    permission_mode: Option<ResolvedPermissionMode>,  // 默认权限模式
    sandbox: SandboxConfig,           // 沙箱配置
}
```

**没有 CLAW.md 等效的指导文档注入系统**——`CLAW.md` 文件存在但仅作为项目文档，不会自动注入系统提示。项目指导通过 `INSTRUCTIONS.md` 文件发现并注入（`prompt.rs`）。

---

#### 1.3.4 沙箱与隔离

`runtime/src/sandbox.rs` 定义了沙箱配置（目前是配置解析层，实际隔离机制依赖操作系统）：

```rust
// 三种文件系统隔离模式
pub enum FilesystemIsolationMode {
    Off,             // 无隔离（默认）
    WorkspaceOnly,   // 仅允许访问工作区目录（默认启用）
    AllowList,       // 显式白名单目录
}

pub struct SandboxConfig {
    pub enabled: Option<bool>,
    pub namespace_restrictions: Option<bool>,   // Linux 命名空间隔离
    pub network_isolation: Option<bool>,        // 网络访问限制
    pub filesystem_mode: Option<FilesystemIsolationMode>,
    pub allowed_mounts: Vec<String>,            // AllowList 模式的白名单
}
```

`SandboxStatus` 区分了"请求的隔离"和"实际生效的隔离"（`sandbox.rs:53-70`），因为某些隔离功能依赖特定内核版本或权限。

---

### 1.4 其他功能模块

#### 1.4.1 Token 计量与成本估算

`runtime/src/usage.rs`（310 行含测试）是代码库中**测试最完整**的模块，有 5 个单元测试。

```rust
pub struct TokenUsage {
    pub input_tokens: u32,
    pub output_tokens: u32,
    pub cache_creation_input_tokens: u32,  // Prompt Cache 写入
    pub cache_read_input_tokens: u32,      // Prompt Cache 命中
}

pub struct UsageTracker {
    latest_turn: TokenUsage,   // 最近一轮的 token 使用
    cumulative: TokenUsage,    // 跨轮次累积总量
    turns: u32,                // 总轮次数
}
```

`UsageTracker::from_session()` 从会话消息重建使用量统计，让恢复会话后也能正确显示历史消耗。

##### 模型定价硬编码

```rust
// usage.rs:56-78
pub fn pricing_for_model(model: &str) -> Option<ModelPricing> {
    if model.contains("haiku") {
        Some(ModelPricing { input: $1.0, output: $5.0, cache_create: $1.25, cache_read: $0.1 })
    } else if model.contains("opus") {
        Some(ModelPricing { input: $15.0, output: $75.0, cache_create: $18.75, cache_read: $1.5 })
    } else if model.contains("sonnet") {
        Some(ModelPricing { input: $15.0, output: $75.0, ... })
    } else {
        None  // 未知模型：输出 "pricing=estimated-default" 警告
    }
}
```

成本摘要行格式（`summary_lines_for_model()`）：

```
usage: total_tokens=1847530 input=1000000 output=500000 cache_write=100000
       cache_read=200000 estimated_cost=$54.6750 model=claude-sonnet-4-6
  cost breakdown: input=$15.0000 output=$37.5000 cache_write=$1.8750 cache_read=$0.3000
```

---

#### 1.4.2 系统提示构建

`runtime/src/prompt.rs` 负责构建发给 LLM 的系统提示。

##### ProjectContext：项目上下文发现

```rust
pub struct ProjectContext {
    pub cwd: PathBuf,
    pub current_date: String,
    pub git_status: Option<String>,   // `git status --short`
    pub git_diff: Option<String>,     // `git diff --stat HEAD`
    pub instruction_files: Vec<ContextFile>,  // 发现的 INSTRUCTIONS.md 等
}
```

`discover_instruction_files()` 在 CWD 及其父目录查找 `INSTRUCTIONS.md`、`AGENTS.md`、`.claude/CLAUDE.md` 等指导文件，上限 `MAX_INSTRUCTION_FILE_CHARS = 4,000` 字符/文件，全部内容合计上限 `MAX_TOTAL_INSTRUCTION_CHARS = 12,000` 字符：

```rust
// prompt.rs:40-41
const MAX_INSTRUCTION_FILE_CHARS: usize = 4_000;
const MAX_TOTAL_INSTRUCTION_CHARS: usize = 12_000;
```

##### SYSTEM_PROMPT_DYNAMIC_BOUNDARY

```rust
pub const SYSTEM_PROMPT_DYNAMIC_BOUNDARY: &str = "__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__";
```

系统提示分两段：边界之前是静态内容（工具描述、能力声明等），边界之后是动态内容（项目上下文、日期、git 状态、指导文件）。这个边界用于 Prompt Cache——静态段可以稳定命中缓存，动态段每次可能不同。

---

#### 1.4.3 斜杠命令系统

`commands/src/lib.rs` 定义了 22 个斜杠命令，分 5 个类别：

```rust
// commands/src/lib.rs:73+ ，SLASH_COMMAND_SPECS 数组
// Core flow：         /help, /status, /clear, /version
// Workspace & memory：/memory, /init, /config, /diff, /export
// Sessions & output： /resume, /compact, /cost
// Git & GitHub：      /git-status, /pr, /diff
// Automation：        /skills, /agents, /model, /permissions, /plugins, /hooks
```

`SlashCommandSpec` 携带命令元数据：

```rust
pub struct SlashCommandSpec {
    pub name: &'static str,
    pub aliases: &'static [&'static str],    // 别名（如 /? = /help）
    pub summary: &'static str,               // /help 时展示的简介
    pub argument_hint: Option<&'static str>, // 参数提示（如 "/resume <session-id>"）
    pub resume_supported: bool,              // 是否可在 resume 模式使用
    pub category: SlashCommandCategory,      // 分类（用于 /help 分组展示）
}
```

部分命令（如 `/compact`）直接调用 `compact_session()` 触发手动压缩；`/cost` 调用 `UsageTracker` 展示成本摘要；`/resume` 支持从历史 session 文件恢复对话。

---

#### 1.4.4 Python 移植工作空间

`/src/`（~1,673 行，纯标准库）是一个完整的"参考实现+审计工具"，与 Rust 实现并行存在：

##### 核心模块

| 模块 | 职责 | 行数 |
|------|------|------|
| `runtime.py` | 会话引导、命令/工具路由 | ~180 |
| `query_engine.py` | 对话轮次模拟（含 token 预算、自动压缩） | ~220 |
| `parity_audit.py` | 与归档 TS 源码对比，报告功能缺口 | ~120 |
| `context.py` | 项目上下文发现（Python 文件、测试文件、归档） | ~90 |
| `permissions.py` | 工具权限上下文 + 拒绝追踪 | ~80 |
| `session_store.py` | JSON 会话持久化 | ~70 |
| `transcript.py` | 事件日志记录 | ~60 |

##### 核心价值：parity_audit

`parity_audit.py` 对比 `reference_data/tools_snapshot.json`（100+ 工具）和 `reference_data/commands_snapshot.json`（200+ 命令）与 Rust 实现，自动报告功能缺口：

```bash
python3 -m src.main summary   # 查看移植进度
python3 -m src.main tools     # 列出工具清单（带状态）
python3 -m src.main commands  # 列出命令清单（带状态）
```

这是 clean-room 开发的关键工具——在不接触原始版权代码的前提下，通过对比公开文档和 snapshot 来衡量实现完整度。

---

#### 1.4.5 兼容性层：compat-harness

`crates/compat-harness` 是一个轻量辅助 crate，负责从归档 TypeScript 源码提取 manifest 信息（工具/命令/skill 元数据），供 IDE 集成和编辑器补全使用。

```rust
// compat_harness::extract_manifest()
// 解析归档的 TS 源码，提取工具/命令元数据
// 供 dump-manifests 子命令输出
```

---

### 1.5 架构亮点与局限

#### 架构亮点

**1. 泛型解耦，可测试性极高**

`ConversationRuntime<C: ApiClient, T: ToolExecutor>` 是整个代码库最重要的设计决策。任意测试可以用 mock 客户端和 mock 工具执行器替换真实实现，无需网络、无需文件系统，测试运行速度快、覆盖率高。这在 AI 代理系统中非常罕见——大多数同类工具的核心循环都难以单元测试。

**2. Hook 系统真正执行外部命令**

与很多系统"Hook 配置解析完整、执行管道未实现"不同，Claw Code 的 `HookRunner` 确实启动子进程、等待退出码、解析输出。格式化、lint、类型检查等工具链集成是真实可用的功能。

**3. 权限系统的有序枚举设计**

`PermissionMode: Ord` 让权限检查可以用 `>=` 一行完成，同时保留了清晰的语义层次。5 个单元测试覆盖所有关键路径，是代码库中设计最精良的模块之一。

**4. 压缩系统的递进合并**

`merge_compact_summaries()` 在多次压缩时保留历史摘要（Previously/Newly compacted context），避免了"压缩后历史记忆彻底丢失"的问题。这是一个需要仔细推敲才会设计出来的功能。

**5. 自研 SSE 解析器，零外部依赖**

完整的 Anthropic API SSE 解析器自研实现，不依赖任何第三方 SSE 库。可以精确控制每个事件的处理时机，方便支持新事件类型，也减少了供应链依赖风险。

**6. unsafe_code = "forbid"**

Cargo workspace 级别禁止 unsafe 代码，配合 Clippy pedantic 警告，提供了编译期的安全网。所有内存操作都在 Rust 所有权系统保护下。

---

#### 架构局限

**1. 工具串行执行**

`run_turn()` 里对 `pending_tool_uses` 的 `for` 循环是顺序迭代——N 个并发安全的只读工具（如多个 `read_file`）也会逐个串行执行。Claude Code 的 `StreamingToolExecutor` + `p-map` 可以并行执行多个只读工具，在大规模代码探索场景下延迟差异显著。

**2. 等待完整响应后执行工具**

`build_assistant_message()` 把所有 `AssistantEvent` 收集到 `Vec` 后才构建消息并执行工具，而不是在流式接收时即触发调度（Claude Code 的"边收边跑"模式）。对于长输出的工具调用，这意味着额外的等待延迟。

**3. Token 估算不准**

`len / 4 + 1` 的 token 估算在中文、代码密集等场景误差可能超过 50%，导致压缩触发时机不准确——可能在实际 token 还充裕时就触发压缩，也可能在实际已超限时没有及时压缩。

**4. 无 PTL 恢复链**

当 API 返回 prompt_too_long 错误（实际 token 超出模型上下文窗口）时，`run_turn()` 直接返回 `RuntimeError`，没有尝试压缩后重试的恢复路径。Claude Code 有完整的 collapse_drain → reactive_compact 两步恢复链。

**5. 无 max_tokens 升级**

LLM 因 `stop_reason: max_tokens` 中途停止输出时，没有自动升级 `max_tokens` 参数或注入"Resume from where you left off"消息的恢复机制。

**6. 无死循环检测**

没有 CoStrict 的 `DOOM_LOOP_THRESHOLD = 3`（检测完全相同工具调用连续出现）或 Claude Code 的连续权限拒绝计数。AI 可以无限重复同一个无效工具调用直到触发 `max_iterations`。

**7. 压缩为规则生成而非 LLM 生成**

`summarize_messages()` 的摘要完全基于关键词扫描和文本截断，不调用 LLM。生成的摘要语义保真度低——关键决策（"选择 PostgreSQL 而不是 SQLite"）可能被截断为无意义的片段，而不重要的高频操作（大量 `read_file`）占据了摘要的大部分篇幅。

**8. 持久化粒度粗**

会话以整体 JSON 序列化保存（`Session::save_to_path()`），没有增量写入、没有 WAL 保护。进程在写入中途崩溃可能导致 session 文件损坏。CoStrict 的 SQLite WAL 模式在这方面完全不同。

---

#### 核心数据流全景

```
用户输入
  ↓
parse_args() → CliAction
  ↓
LiveCli::new(model, ...) → 构建 ConversationRuntime<C, T>
  ↓
┌─────────────────────────────────────────────────────┐
│  run_turn(user_input, prompter)                    │
│                                                     │
│  1. session.messages.push(user_text)               │
│  ↓                                                  │
│  loop {                                             │
│    2. api_client.stream(request)                   │
│       → Vec<AssistantEvent>                        │
│    3. build_assistant_message(events)              │
│       → ConversationMessage + TokenUsage           │
│    4. session.messages.push(assistant_message)     │
│    5. if pending_tool_uses.is_empty() { break }    │
│    6. for each tool_use:                           │
│         a. permission_policy.authorize()           │
│         b. hook_runner.run_pre_tool_use()          │
│         c. tool_executor.execute()                 │
│         d. hook_runner.run_post_tool_use()         │
│         e. session.messages.push(tool_result)      │
│  }                                                  │
│  ↓                                                  │
│  return TurnSummary { messages, results, usage }   │
└─────────────────────────────────────────────────────┘
  ↓
render_markdown(output) → 终端显示
  ↓
session.save_to_path() → JSON 文件持久化
```

---

*本文档基于代码库直接分析，所有代码引用均指向具体文件和行号。*
*生成时间：2026-04-05 | 分析者：Claude Code (claude-sonnet-4-6)*
