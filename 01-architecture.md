# 整体架构对比：Claude Code vs Codex CLI

## Claude Code 实现

### 六层分层架构

Claude Code 采用**六层架构**，而非传统的 MVC 模式。这种设计的根本原因是需要管理三种不同生命周期的状态：**进程级**（State/基础设施）、**会话级**（UI/Hooks）、**轮次级**（Query/Services/Tools）。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CLAUDE CODE CLI                                 │
│                                                                         │
│  ┌──────────┐  ┌────────────┐  ┌──────────┐  ┌──────────────────────┐  │
│  │  main.tsx │─▶│ init()     │─▶│ launchRe │─▶│  <App>               │  │
│  │ (entry)   │  │ (bootstrap)│  │ pl()     │  │   └─ <REPL>          │  │
│  └──────────┘  └────────────┘  └──────────┘  │       └─ PromptInput  │  │
│                                               │       └─ Messages    │  │
│  ┌──────────────────────────────────────────┐ └──────────────────────┘  │
│  │           QueryEngine (per session)       │                          │
│  │  ┌──────────────────────────────────┐     │                          │
│  │  │  query() — async generator loop  │     │                          │
│  │  │  ┌────────────────────────────┐  │     │                          │
│  │  │  │ queryModelWithStreaming()  │  │     │                          │
│  │  │  │  ├─ Build system prompt    │  │     │                          │
│  │  │  │  ├─ Normalize messages     │  │     │                          │
│  │  │  │  ├─ Stream API response    │  │     │                          │
│  │  │  │  └─ Yield events           │  │     │                          │
│  │  │  └────────────────────────────┘  │     │                          │
│  │  │  ┌────────────────────────────┐  │     │                          │
│  │  │  │ runTools() orchestration   │  │     │                          │
│  │  │  │  ├─ Permission check       │  │     │                          │
│  │  │  │  ├─ Hook execution         │  │     │                          │
│  │  │  │  ├─ Concurrent/serial exec │  │     │                          │
│  │  │  │  └─ Yield tool results     │  │     │                          │
│  │  │  └────────────────────────────┘  │     │                          │
│  │  │  ┌────────────────────────────┐  │     │                          │
│  │  │  │ Compaction (auto/micro)    │  │     │                          │
│  │  │  │  ├─ Token budget tracking  │  │     │                          │
│  │  │  │  ├─ Auto-compact trigger   │  │     │                          │
│  │  │  └─ Message summarization  │  │     │                          │
│  │  └──────────────────────────────────┘     │                          │
│  └───────────────────────────────────────────┘                          │
│                                                                         │
│  ┌──────────────┐ ┌────────────┐ ┌───────────┐ ┌────────────────────┐  │
│  │ Tool Registry │ │ Permission │ │ Hook      │ │ MCP Clients        │  │
│  │ (40+ tools)   │ │ Engine     │ │ Engine    │ │ (stdio/sse/ws/sdk) │  │
│  └──────────────┘ └────────────┘ └───────────┘ └────────────────────┘  │
│                                                                         │
│  ┌──────────────┐ ┌────────────┐ ┌───────────┐ ┌────────────────────┐  │
│  │ Skill Loader  │ │ Plugin Mgr │ │ Analytics │ │ State Store        │  │
│  │ (fs/bundled/  │ │ (builtin/  │ │ (OTel +   │ │ (AppState +        │  │
│  │  mcp/managed) │ │  market)   │ │  1P logs) │ │  React contexts)   │  │
│  └──────────────┘ └────────────┘ └───────────┘ └────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 六层职责说明

| 层级 | 位置 | 职责 | 生命周期 |
|------|------|------|----------|
| **UI 层** | `src/components/`, `src/screens/` | 纯渲染层，React/Ink 组件，声明式终端 UI | 会话级 |
| **Hooks 层** | `src/hooks/` (70+ hooks) | 封装副作用和状态逻辑，可被多个 UI 组件复用 | 会话级 |
| **State 层** | `src/state/`, `src/bootstrap/state.ts` | 进程级生命周期状态（全局单例 + Zustand store） | 进程级 |
| **Query 层** | `src/query.ts`, `src/QueryEngine.ts` | 单轮对话的瞬态状态管理，async generator 核心循环 | 轮次级 |
| **Services 层** | `src/services/` (13 个子系统) | 无状态能力提供者（API 客户端、压缩算法、MCP 协议等） | 进程级 |
| **Tools 层** | `src/tools/` (40+ 工具) | 具有身份标识的执行单元（名称、描述、权限要求） | 轮次级 |

### 层间数据流

六层之间的数据流遵循严格的单向依赖原则，上层可以调用下层，但下层不能直接回调上层（通过事件/yield 机制向上传递）：

```
┌─────────────────────────────────────────────────────────────────┐
│                        数据流方向                                │
│                                                                 │
│  UI 层 ──────▶ Hooks 层 ──────▶ State 层                       │
│    ▲              │                │                            │
│    │              │                ▼                            │
│    │              │           Query 层 ──────▶ Services 层      │
│    │              │                │                │            │
│    │              │                ▼                ▼            │
│    │              │           Tools 层 ◀────────────────        │
│    │              │                │                             │
│    │              │                ▼                             │
│    └──────────────┴──── yield StreamEvent ─────────┘            │
│                                                                 │
│  关键数据流路径：                                                 │
│  1. 用户输入 → UI → Query → API (Services) → StreamEvent → UI  │
│  2. 工具调用 → Query → Permission (Services) → Tools → Result  │
│  3. 状态变更 → State → React Context → UI 重渲染                │
│  4. 压缩触发 → Query → Compact (Services) → 消息截断 → Query   │
└─────────────────────────────────────────────────────────────────┘
```

**核心数据流路径详解：**

1. **用户输入流**：用户在 `PromptInput` 组件输入文本 -> `useSendMessage` Hook 处理 -> 调用 `QueryEngine.sendMessage()` -> 进入 `query()` async generator -> 调用 `queryModelWithStreaming()` (Services 层 API 客户端) -> yield `StreamEvent` -> UI 层通过 `useQueryEvents` Hook 消费事件并渲染
2. **工具执行流**：模型返回 `tool_use` block -> `StreamingToolExecutor` 收集完整参数 -> `runTools()` 编排 -> 权限检查 (Services 层) -> 工具执行 (Tools 层) -> 结果 yield 回 Query 层 -> 追加到 messages 数组
3. **状态传播流**：Bootstrap 初始化 `AppStateStore` -> React Context Provider 注入 -> 各 UI 组件通过 `useStore()` 消费 -> 状态变更触发 React 重渲染
4. **压缩数据流**：每轮 API 调用后检查 token 预算 -> 超阈值触发 `microCompact()` 或 `autoCompact()` -> 修改 messages 数组（原地截断/摘要替换）-> 下一轮 API 调用使用压缩后的上下文

### 状态生命周期管理

Claude Code 的状态管理核心挑战在于三种不同粒度的生命周期需要协调运作：

```
┌─────────────────────────────────────────────────────────────────┐
│                     状态生命周期分层                              │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  进程级状态 (Process-Lifetime)                           │   │
│  │  生命周期：从 main.tsx 启动到进程退出                     │   │
│  │  存储位置：bootstrap/state.ts (全局单例)                  │   │
│  │  包含内容：                                               │   │
│  │    • OAuth token、API 密钥                               │   │
│  │    • GrowthBook 特性标志实例                             │   │
│  │    • 全局设置 (settings.json)                            │   │
│  │    • OpenTelemetry 导出器                                │   │
│  │    • MCP 客户端连接池                                    │   │
│  │    • 文件系统缓存 (LRU, 100文件/25MB)                    │   │
│  │  特点：跨会话持久化，进程重启后需重新初始化                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  会话级状态 (Session-Lifetime)                           │   │
│  │  生命周期：从 /clear 或启动到 /clear 或退出               │   │
│  │  存储位置：React Context + Zustand store                  │   │
│  │  包含内容：                                               │   │
│  │    • 对话消息历史 (messages[])                           │   │
│  │    • 工具注册表 (当前可用工具)                            │   │
│  │    • 权限模式 (default/auto/bypass/plan)                 │   │
│  │    • 成本追踪器 (costTracker)                            │   │
│  │    • UI 状态 (输入焦点、通知、模态框)                     │   │
│  │    • 会话元数据 (session ID, 启动时间)                   │   │
│  │  特点：/clear 时重置，可持久化到磁盘恢复                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  轮次级状态 (Turn-Lifetime)                              │   │
│  │  生命周期：单次 query() 调用（用户发送到 Claude 完成响应） │   │
│  │  存储位置：query() async generator 闭包变量               │   │
│  │  包含内容：                                               │   │
│  │    • 当前轮次的 AbortController                          │   │
│  │    • StreamingToolExecutor 实例                          │   │
│  │    • 流式响应累积缓冲区                                  │   │
│  │    • 工具结果收集数组                                    │   │
│  │    • max-output-tokens 重试计数器                        │   │
│  │    • 临时文件句柄                                        │   │
│  │  特点：query() 返回后即被 GC 回收，不跨轮次保留          │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 为什么选择六层而非传统 MVC

| 传统 MVC 假设 | Claude Code 实际情况 | 六层架构的应对 |
|---------------|---------------------|---------------|
| 无状态请求 | 长对话需要维护完整消息历史 | **Query 层**：有状态的 async generator，持有跨轮次的消息数组 |
| 同步请求-响应 | 流式 SSE 响应 + 并行工具执行 | **Services 层**：流式 API 客户端 + StreamingToolExecutor |
| 单一数据模型 | 三种生命周期的状态混合 | **State 层**：分离进程级/会话级/轮次级状态 |
| 服务端渲染 | 终端实时 UI 更新 | **UI 层 + Hooks 层**：React/Ink 声明式渲染，yield 事件驱动更新 |
| 固定功能集 | 40+ 工具可动态加载/卸载 | **Tools 层**：具有身份标识的执行单元，支持延迟加载 |
| 单一入口 | CLI/SDK/MCP 服务器/IDE Bridge 多入口 | **分层解耦**：Query 层可被多种入口复用 |

**核心设计原则**：
1. **关注点分离**：每层只关心自己的职责，UI 不需要知道工具如何执行，Tools 不需要知道消息如何渲染
2. **生命周期隔离**：不同层的状态有不同的生命周期，避免"僵尸状态"泄漏
3. **可测试性**：Services 层无状态，可以独立单元测试；Tools 层有明确接口，可以 mock
4. **可扩展性**：新工具只需实现 Tool 接口；新 Hook 只需注册事件回调；新 MCP 服务器只需配置

---

## Codex CLI 实现

### Cargo Workspace 微服务化架构

Codex CLI 采用 **Cargo Workspace** 组织，包含 **60+ 个 Crate**，实现了高粒度的模块化。每个 Crate 职责单一，通过 workspace 依赖管理。

```
┌─────────────────────────────────────────────────────────────────┐
│                     Codex CLI (Rust)                             │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    入口层 (Entry Points)                  │   │
│  │  ┌─────────┐    ┌─────────┐    ┌─────────┐              │   │
│  │  │   cli   │    │   tui   │    │   exec  │              │   │
│  │  │ 多工具  │    │ 全屏 UI │    │ 无头    │              │   │
│  │  │ 入口    │    │ Ratatui │    │ 模式    │              │   │
│  │  └────┬────┘    └────┬────┘    └────┬────┘              │   │
│  └───────┼──────────────┼──────────────┼───────────────────┘   │
│          │              │              │                        │
│          └──────────────┼──────────────┘                        │
│                         ▼                                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    核心层 (Core)                          │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │  codex.rs — 主结构体 + 事件循环                   │   │   │
│  │  │  agent/ — Agent 循环核心逻辑                      │   │   │
│  │  │  tools/ — 内置工具实现                            │   │   │
│  │  │  sandboxing/ — 沙箱策略                           │   │   │
│  │  │  context_manager/ — 上下文管理                    │   │   │
│  │  │  guardian/ — 安全守护                             │   │   │
│  │  │  client.rs — API 客户端                           │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    基础设施层 (Infrastructure)             │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐  │   │
│  │  │sandboxing│ │  mcp-*   │ │  state   │ │  config   │  │   │
│  │  │linux/win │ │server/   │ │persist   │ │  loader   │  │   │
│  │  │          │ │client/   │ │          │ │           │  │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────────┘  │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐  │   │
│  │  │ codex-   │ │ models-  │ │  login   │ │ analytics │  │   │
│  │  │ client   │ │  manager  │ │  auth    │ │  otel     │  │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Workspace 组织策略

Codex CLI 的 Cargo Workspace 采用**分层组织策略**，将 60+ 个 Crate 按职责划分为清晰的层级：

| 层级 | Crate 数量 | 说明 |
|------|-----------|------|
| **入口层** | 3 | `cli`、`tui`、`exec` — 提供不同的用户交互模式 |
| **核心层** | 5 | `core`、`tools`、`protocol`、`codex-api`、`codex-client` — 业务逻辑主体 |
| **沙箱层** | 6 | `sandboxing`、`linux-sandbox`、`windows-sandbox-rs`、`process-hardening`、`execpolicy`、`shell-escalation` |
| **MCP 层** | 3 | `mcp-server`、`rmcp-client`、`mcp-types` |
| **模型层** | 5 | `codex-client`、`codex-api`、`chatgpt`、`lmstudio`、`ollama`、`models-manager` |
| **状态层** | 3 | `rollout`、`codex-state`、`state-db` |
| **配置层** | 2 | `codex-config`、`codex-features` |
| **认证层** | 2 | `codex-login`、`codex-keyring-store` |
| **遥测层** | 2 | `codex-analytics`、`codex-otel` |
| **工具层** | 3 | `codex-tools`、`codex-apply-patch`、`codex-shell-command` |
| **网络层** | 2 | `codex-network-proxy`、`codex-exec-server` |
| **其他** | ~20 | `codex-utils-*`、`codex-git-utils`、`codex-terminal-detection` 等 |

### Crate 依赖关系

```
                    ┌─────────┐
                    │   cli   │
                    └────┬────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         ┌────────┐ ┌────────┐ ┌────────┐
         │  tui   │ │  exec  │ │  core  │
         └───┬────┘ └───┬────┘ └───┬────┘
             │          │          │
             └──────────┼──────────┘
                        ▼
              ┌──────────────────┐
              │    protocol      │  ◄── 类型定义、Op/Event 枚举
              └────────┬─────────┘
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ codex-   │  │  tools   │  │sandboxing│
   │  api     │  │          │  │          │
   └────┬─────┘  └────┬─────┘  └────┬─────┘
        │             │             │
        ▼             ▼             ▼
   ┌──────────┐  ┌──────────┐  ┌──────────────┐
   │codex-    │  │apply-    │  │linux-sandbox │
   │client    │  │patch     │  │windows-      │
   └──────────┘  └──────────┘  │sandbox-rs    │
                               └──────────────┘
```

**依赖规则**：
- `protocol` 是最底层的类型定义 crate，几乎所有 crate 都依赖它
- `core` 依赖 `protocol`、`codex-api`、`tools`、`sandboxing`
- `cli`/`tui`/`exec` 仅依赖 `core` 和少量基础设施 crate
- 沙箱 crate 之间互不依赖，通过 `sandboxing` 抽象层桥接

### 为什么选择 60+ Crate 微服务化

选择如此高粒度的 Crate 拆分有以下几个关键原因：

1. **编译隔离**：修改 `apply-patch` 的解析器不需要重新编译 `core`，大幅缩短开发迭代时间。Rust 编译器以 crate 为增量编译单元，60+ crate 意味着 60+ 个并行编译任务。

2. **职责单一**：每个 crate 有明确的职责边界。例如 `codex-network-proxy` 只处理网络代理，`codex-keyring-store` 只处理密钥存储，便于独立测试和维护。

3. **条件编译**：平台特定代码可以独立为 crate，通过 `#[cfg(target_os)]` 控制编译。`linux-sandbox` 仅在 Linux 上编译，`windows-sandbox-rs` 仅在 Windows 上编译。

4. **依赖最小化**：`protocol` crate 不依赖任何网络库，可以安全地在轻量级上下文中使用。`apply-patch` 可以作为独立可执行文件运行（`standalone_executable`）。

5. **渐进式迁移**：从 TypeScript 迁移到 Rust 时，高粒度 crate 允许逐个模块迁移，降低大规模重构风险。

6. **安全审计**：安全关键代码（沙箱、认证）隔离在独立 crate 中，便于安全审计和代码审查。

---

## 对比分析

### 架构模式对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **架构风格** | 六层分层架构（单体） | Cargo Workspace 微服务化（60+ Crate） |
| **组织单位** | 目录/文件（1884 个 TS 文件） | Crate（60+ 独立编译单元） |
| **模块化粒度** | 中等（按功能目录划分） | 极高（每个 crate 职责单一） |
| **编译隔离** | 无（Bun bundle 整体打包） | 有（crate 级增量编译） |
| **依赖管理** | npm package.json | Cargo.toml workspace |

### 技术选型对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **语言** | TypeScript（严格模式） | Rust（Edition 2024） |
| **运行时** | Bun（JS 运行时） | 原生二进制（无运行时依赖） |
| **UI 框架** | React + Ink（声明式） | Ratatui + crossterm（命令式） |
| **异步模型** | Bun 内置 async/await | Tokio 1 async runtime |
| **构建系统** | Bun bundle | Bazel 9（CI）+ Cargo（开发） |
| **可复现构建** | 无 | Nix flake.nix |

### 状态管理对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **状态分层** | 三层（进程级/会话级/轮次级） | 两层（Session/ActiveTurn） |
| **进程级状态** | 全局单例 + Zustand store | Arc\<Session\> + watch channel |
| **会话级状态** | React Context + Zustand | Arc\<RwLock\<SessionState\>\> |
| **轮次级状态** | async generator 闭包 | TurnContext + CancellationToken |
| **状态传播** | yield StreamEvent + React Context | Event Queue (rx_event) |

### 入口模式对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **入口数量** | 单一入口（main.tsx） | 多入口（cli/tui/exec） |
| **交互模式** | REPL 交互式 | CLI/TUI/无头 三种模式 |
| **多入口支持** | 通过 SDK/MCP 桥接 | 原生多二进制入口 |

### 架构哲学差异

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **设计哲学** | 功能优先，单体集中 | 模块优先，微服务解耦 |
| **复杂度管理** | 层级抽象（六层） | crate 隔离（60+） |
| **扩展方式** | 实现 Tool 接口 + Hook 注册 | 新建 crate + 实现 trait |
| **平台适配** | Bun 运行时抽象 | 条件编译 `#[cfg(target_os)]` |

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| 六层架构清晰分离关注点，职责边界明确 | 单体架构，修改任一层可能影响整体 |
| 三种状态生命周期精细管理，避免状态泄漏 | 50 万行代码规模庞大，理解成本高 |
| React/Ink 声明式 UI 开发效率高 | 依赖 Bun 运行时，部署和分发受限 |
| async generator 流式模型优雅，事件驱动 | 无编译隔离，任意文件修改需全量重构建 |
| 丰富的 Hooks 层（70+）实现逻辑复用 | 闭源，无法审查内部架构决策 |
| 功能丰富（40+ 工具、87+ 命令、13 子系统） | 单一入口模式，无头执行需额外适配 |
| GrowthBook 特性标志灵活控制功能发布 | 无可复现构建，环境差异可能导致问题 |
| MCP 协议自研实现支持 4 种传输 | TypeScript 运行时性能不如 Rust 原生 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| 60+ Crate 微服务化，编译隔离极致 | 架构复杂度高，新贡献者学习曲线陡峭 |
| Rust 原生二进制，零运行时依赖 | Ratatui 命令式 UI 开发效率低于 React/Ink |
| 增量编译大幅缩短开发迭代时间 | crate 间接口维护成本高 |
| 条件编译优雅处理平台差异 | 60+ crate 的依赖关系管理复杂 |
| 多入口模式（CLI/TUI/Exec）灵活适配 | 功能相对精简（25+ 工具，无斜杠命令） |
| Nix flake 支持可复现构建 | 从 TypeScript 迁移，部分设计仍有 TS 痕迹 |
| 安全关键代码隔离在独立 crate，便于审计 | 无 Hooks 层等价物，逻辑复用机制较弱 |
| apply-patch 可独立运行为可执行文件 | protocol crate 作为底层依赖，变更影响面广 |
| Bazel 9 CI 构建系统成熟 | 无特性标志系统（无 GrowthBook 等价物） |
