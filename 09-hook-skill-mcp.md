# Hook、技能与 MCP 集成对比

## Claude Code 实现

### Hook 系统

Hooks 是用户定义的生命周期事件动作，是 Claude Code 的**可扩展性骨干**。

#### Hook 事件类型（13 种）

| 事件 | 触发时机 |
|------|----------|
| `SessionStart` | 启动/恢复/清除/压缩时 |
| `Stop` | Claude 结束响应前 |
| `UserPromptSubmit` | 用户提交时（exit code 2 = 阻止提交） |
| `PreToolUse` | 工具执行前（exit code 2 = 阻止并显示 stderr） |
| `PostToolUse` | 成功执行后 |
| `PostToolUseFail` | 失败执行后 |
| `SubagentStart/Stop` | 子 Agent 生成/完成 |
| `TaskCreated/Completed` | 任务注册/到达终态 |
| `PermissionDenied` | auto 模式拒绝时 |
| `ConfigChange` | 设置变更时 |
| `CwdChanged` | 工作目录变更时 |
| `FileChanged` | 监视文件变更时 |
| `Notification` | 通知发送时 |

#### Hook 类型（5 种）

| 类型 | 实现 | 特点 |
|------|------|------|
| **Command Hook** | Shell 命令 (bash/zsh) | Exit code 0=ok, 2=block, N=error |
| **Prompt Hook** | LLM 评估条件 (Haiku 模型) | 返回 `{ok: true/false, reason}` |
| **Agent Hook** | 完整 Agent + 工具 | 超时 60s，无递归 |
| **HTTP Hook** | HTTP 请求到端点 | JSON body + context |
| **Function Hook** | TS 回调（内存中） | 仅会话级，不持久化 |

### 技能系统

#### 技能来源（4 级优先级）

| 来源 | 位置 | 优先级 |
|------|------|--------|
| 托管技能 | 企业策略控制 | 最高 |
| 项目技能 | `./.claude/skills/` | 高 |
| 用户技能 | `~/.claude/skills/` | 中 |
| 内置技能 | 编译打包 | 最低 |

#### 技能执行模式

- **内联模式（默认）**：技能内容注入到当前对话中，模型视为当前轮次的一部分
- **Fork 模式（`context: "fork"`）**：创建子 Agent 独立执行，拥有自己的上下文和预算，返回结果文本给父 Agent

#### 条件技能（路径过滤）

技能可以配置 `paths` 字段，仅当模型编辑匹配的文件时才激活。

### MCP 集成

#### MCP 客户端架构

支持 4 种传输协议：
- **stdio**：本地进程 stdin/stdout 管道
- **SSE/HTTP**：远程 HTTP + EventSource + OAuth
- **WebSocket**：持久连接 + 二进制帧 + TLS/代理
- **local**：本地协议

工具名称规范化：服务器 `my-server` 的工具 `send_message` -> `mcp__my_server__send_message`

#### MCP 配置作用域（5 级）

| 作用域 | 位置 | 用例 |
|--------|------|------|
| `local` | `.claude/settings.local.json` | 用户本地服务器 |
| `user` | `~/.claude/settings.json` | 用户全局服务器 |
| `project` | `.claude/settings.json` | 团队共享服务器 |
| `dynamic` | 运行时注册 | 编程式服务器 |
| `enterprise` | MDM 策略 | 管理员管理服务器 |

---

## Codex CLI 实现

### MCP 集成

#### codex-rmcp-client

MCP 客户端通过 `codex-rmcp-client` crate 实现，核心是 `McpConnectionManager`：

```rust
/// MCP 连接管理器 -- 管理所有 MCP 服务器连接
pub struct McpConnectionManager {
    /// 已连接的 MCP 服务器
    connections: HashMap<String, McpConnection>,

    /// 工具信息缓存
    tool_info_cache: HashMap<String, Vec<ToolInfo>>,
}

/// MCP 连接
pub struct McpConnection {
    /// 服务器名称
    pub server_name: String,

    /// 服务器配置
    pub config: McpServerConfig,

    /// 客户端实例
    pub client: rmcp::Client,
}
```

#### config.toml 配置方式

```toml
[mcp_servers]
# stdio 传输
my-local-server = { command = "npx", args = ["-y", "@my/mcp-server"] }

# SSE 传输
my-remote-server = { url = "https://my-server.com/mcp" }

# 带环境变量的服务器
my-auth-server = {
    command = "python",
    args = ["-m", "my_mcp"],
    env = { API_KEY = "${MY_API_KEY}" }
}
```

#### MCP 工具暴露控制

MCP 工具通过 `McpToolSnapshot` 暴露给模型：

```rust
/// MCP 工具快照 -- 某个时刻的 MCP 工具列表
pub struct McpToolSnapshot {
    /// 服务器名称
    pub server_name: String,

    /// 该服务器提供的工具列表
    pub tools: Vec<McpTool>,
}

/// 工具命名规范: mcp__{server_name}__{tool_name}
/// 例如: mcp__my_server__search_files
pub fn mcp_tool_name(server_name: &str, tool_name: &str) -> String {
    format!("mcp__{}__{}", server_name, tool_name)
}
```

**重要设计 -- MCP 工具不受沙箱保护：**

MCP 工具在 MCP 服务器进程中执行，**不受 Codex CLI 沙箱策略的约束**。这意味着：
- MCP 工具可以访问文件系统的任意位置
- MCP 工具可以发起网络请求
- 安全性依赖于 MCP 服务器自身的安全实现

这是有意的设计选择，因为 MCP 工具通常需要访问外部资源（如数据库、API），沙箱限制会使其无法正常工作。

#### MCP Server 模式

Codex CLI 自身也可以作为 MCP 服务器运行：

```bash
# 启动 Codex CLI 作为 MCP 服务器
codex mcp-server
```

```rust
/// Codex CLI 作为 MCP 服务器的实现
pub struct RmcpServer {
    /// 内部 Codex 实例
    codex: Arc<Codex>,

    /// 暴露的工具
    tools: Vec<McpTool>,
}

/// 暴露给外部客户端的工具
pub struct CodexMcpTool {
    /// 工具名称
    pub name: String,

    /// 工具描述
    pub description: String,
}

/// run_codex 工具 -- 允许外部 MCP 客户端通过 Codex 执行任务
pub struct RunCodexTool {
    pub prompt: String,
    pub cwd: Option<String>,
    pub model: Option<String>,
}
```

### Hook 系统

Codex CLI **没有独立的 Hook 系统**。其可扩展性主要通过以下机制实现：
- **插件系统**：通过 `plugin/*` API 管理插件
- **Guardian 守护者**：安全中间层决定操作是否自动执行
- **沙箱策略**：可配置的执行策略

### 技能系统

Codex CLI **没有独立的技能系统**。类似功能通过以下方式实现：
- **AGENTS.md**：项目级指令文件（类似 CLAUDE.md）
- **提示词模板**：通过配置文件定义
- **插件**：扩展工具能力

---

## 对比分析

### Hook 系统对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **Hook 事件数量** | 13 种生命周期事件 | 无独立 Hook 系统 |
| **Hook 类型** | 5 种（Command/Prompt/Agent/HTTP/Function） | 无 |
| **阻止能力** | 支持（exit code 2 阻止操作） | 通过 Guardian 实现 |
| **LLM 评估** | Prompt Hook（Haiku 模型评估） | 无 |
| **HTTP 集成** | HTTP Hook（请求外部端点） | 无 |
| **编程式扩展** | Function Hook（TS 回调） | 通过插件系统 |

### 技能系统对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **技能来源** | 4 级（托管/项目/用户/内置） | 无独立技能系统 |
| **执行模式** | 内联 + Fork 两种 | N/A |
| **路径过滤** | 支持（paths 字段） | N/A |
| **条件激活** | 支持文件匹配触发 | N/A |
| **企业控制** | 托管技能（最高优先级） | N/A |

### MCP 集成对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **传输协议** | 4 种（stdio/SSE/WebSocket/local） | 2 种（stdio/SSE） |
| **配置作用域** | 5 级（local/user/project/dynamic/enterprise） | 1 级（config.toml） |
| **工具命名** | `mcp__{server}__{tool}` | `mcp__{server}__{tool}`（一致） |
| **MCP Server 模式** | 不支持 | 支持（`codex mcp-server`） |
| **实现方式** | 自研实现 | 基于 rmcp 0.12 |
| **沙箱保护** | N/A | MCP 工具不受沙箱约束 |
| **工具缓存** | 延迟加载（ToolSearchTool） | McpToolSnapshot 快照 |

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| Hook 系统极其丰富（13 事件 x 5 类型），提供全方位生命周期控制 | Hook 系统复杂度高，学习曲线陡峭 |
| Prompt Hook 可利用 LLM 智能评估条件，灵活度极高 | Agent Hook 有 60s 超时限制，复杂任务可能不够 |
| 技能系统支持 4 级优先级和企业托管，适合团队协作 | 技能系统与 Hook 系统重叠，概念边界模糊 |
| MCP 支持 4 种传输协议，覆盖本地和远程场景 | 不支持作为 MCP Server 运行，无法被其他工具调用 |
| 5 级配置作用域精细控制 MCP 服务器可见性 | 自研 MCP 实现，与社区标准可能有偏差 |
| 延迟加载 MCP 工具 Schema，大幅节省 token | MCP 配置分散在多个 JSON 文件中，管理复杂 |
| Function Hook 支持内存中 TS 回调，开发体验好 | Function Hook 不持久化，仅限当前会话 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| 支持作为 MCP Server 运行，可被其他工具集成调用 | 没有独立 Hook 系统，可扩展性受限 |
| 基于 rmcp 0.12 标准，与社区生态兼容性好 | 仅支持 2 种传输协议（stdio/SSE），缺少 WebSocket |
| MCP 配置集中在 config.toml，管理简单 | 配置作用域单一，无法区分用户/项目/企业级别 |
| 工具信息缓存（McpToolSnapshot）提升查询效率 | MCP 工具不受沙箱保护，存在安全隐患 |
| 零依赖原生二进制，MCP 客户端无需额外运行时 | 没有独立技能系统，缺乏 Claude Code 的路径过滤等高级功能 |
| 插件系统提供了一定程度的可扩展性 | 插件系统无法实现 Hook 级别的生命周期控制 |
