# Claude Code 与 Codex CLI 源码架构深度分析

> 本文档基于对 Anthropic Claude Code 和 OpenAI Codex CLI 两个项目的源码级研究，从架构设计、核心模块、执行流程、设计模式、子系统实现等维度进行全面深度剖析与对比分析。文档涵盖代码级实现细节，包含大量源码片段、数据结构定义和架构图。

**文档规模**：约 7,700 行 | **覆盖模块**：50+ 子系统 | **代码示例**：100+ 段

---

## 目录

### 第 1 章 项目概览

- [1.1 基本信息](#11-基本信息)
- [1.2 技术栈对比](#12-技术栈对比)

### 第 2 章 Claude Code 架构深度分析

- [2.1 六层分层架构](#21-六层分层架构)
- [2.2 核心目录结构](#22-核心目录结构)
- [2.3 Agent 循环](#23-agent-循环)
  - [2.3.1 query() async generator 核心结构](#231-query-async-generator-核心结构)
  - [2.3.2 queryModelWithStreaming() 实现](#232-querymodelwithstreaming-实现)
  - [2.3.3 StreamingToolExecutor 并行执行机制](#233-streamingtoolexecutor-并行执行机制)
  - [2.3.4 max-output-tokens 恢复机制](#234-max-output-tokens-恢复机制)
  - [2.3.5 StreamEvent 类型定义](#235-streamevent-类型定义)
  - [2.3.6 QueryEngine 外层循环](#236-queryengine-外层循环)
- [2.4 工具系统](#24-工具系统)
  - [2.4.1 工具注册表](#241-工具注册表)
  - [2.4.2 Tool 接口完整定义](#242-tool-接口完整定义)
  - [2.4.3 buildTool() 工厂函数](#243-buildtool-工厂函数)
  - [2.4.4 runTools() 编排实现](#244-runtools-编排实现)
  - [2.4.5 isConcurrencySafe 判断逻辑](#245-isconcurrencysafe-判断逻辑)
  - [2.4.6 siblingAbortController 错误传播](#246-siblingabortcontroller-错误传播)
  - [2.4.7 延迟加载（ToolSearchTool）](#247-延迟加载toolsearchtool)
- [2.5 上下文与记忆管理](#25-上下文与记忆管理)
  - [2.5.1 多层记忆架构](#251-多层记忆架构)
  - [2.5.2 四层压缩策略](#252-四层压缩策略)
  - [2.5.3 microCompact 实现细节](#253-microcompact-实现细节)
  - [2.5.4 autoCompact 摘要提示词](#254-autocompact-摘要提示词)
  - [2.5.5 reactiveCompact 截断策略](#255-reactivecompact-截断策略)
  - [2.5.6 tokenBudget 计算方式](#256-tokenbudget-计算方式)
  - [2.5.7 持久记忆（memdir）](#257-持久记忆memdir)
  - [2.5.8 提示词缓存优化](#258-提示词缓存优化)
  - [2.5.9 LRU 文件状态缓存](#259-lru-文件状态缓存)
  - [2.5.10 Dream Task](#2510-dream-task)
- [2.6 权限与安全系统](#26-权限与安全系统)
- [2.7 多 Agent 编排](#27-多-agent-编排)
- [2.8 Hook 系统](#28-hook-系统)
- [2.9 技能与插件系统](#29-技能与插件系统)
- [2.10 MCP 集成](#210-mcp-集成)
- [2.11 特性标志与构建系统](#211-特性标志与构建系统)
- [2.12 配置系统](#212-配置系统)
- [2.13 状态管理](#213-状态管理)
- [2.14 错误处理与重试](#214-错误处理与重试)
- [2.15 会话恢复机制](#215-会话恢复机制)
- [2.16 遥测与可观测性](#216-遥测与可观测性)
- [2.17 认证与授权](#217-认证与授权)
- [2.18 IDE 集成](#218-ide-集成)
- [2.19 LSP 集成](#219-lsp-集成)

### 第 3 章 Codex CLI 架构深度分析

- [3.1 Cargo Workspace 微服务化架构](#31-cargo-workspace-微服务化架构)
- [3.2 核心目录结构](#32-核心目录结构)
- [3.3 Agent 循环](#33-agent-循环)
  - [3.3.1 Op/Event 提交-事件模式](#331-opevent-提交-事件模式)
  - [3.3.2 提交循环 (Submission Loop)](#332-提交循环-submission-loop)
  - [3.3.3 Responses API 调用格式](#333-responses-api-调用格式)
  - [3.3.4 SSE 流式处理](#334-sse-流式处理)
  - [3.3.5 工具调用结果反馈](#335-工具调用结果反馈)
  - [3.3.6 无状态请求 + 前缀缓存](#336-无状态请求--前缀缓存)
- [3.4 工具系统](#34-工具系统)
  - [3.4.1 内置工具](#341-内置工具)
  - [3.4.2 工具 Schema 系统](#342-工具-schema-系统)
  - [3.4.3 apply_patch.rs 文件编辑](#343-apply_patchrs-文件编辑)
- [3.5 沙箱系统](#35-沙箱系统)
  - [3.5.1 macOS Seatbelt 沙箱](#351-macos-seatbelt-沙箱)
  - [3.5.2 Linux Landlock + Bubblewrap](#352-linux-landlock--bubblewrap)
  - [3.5.3 Windows Restricted Token](#353-windows-restricted-token)
  - [3.5.4 沙箱策略注入系统提示词](#354-沙箱策略注入系统提示词)
  - [3.5.5 网络访问控制](#355-网络访问控制)
- [3.6 上下文管理](#36-上下文管理)
  - [3.6.1 compact.rs 压缩算法](#361-compactrs-压缩算法)
  - [3.6.2 摘要提示词模板](#362-摘要提示词模板)
  - [3.6.3 Token 感知截断](#363-token-感知截断)
  - [3.6.4 上下文窗口超时处理](#364-上下文窗口超时处理)
- [3.7 Guardian 安全守护系统](#37-guardian-安全守护系统)
- [3.8 多 Agent 系统](#38-多-agent-系统)
- [3.9 配置系统](#39-配置系统)
- [3.10 状态与会话管理](#310-状态与会话管理)
- [3.11 MCP 集成](#311-mcp-集成)
- [3.12 错误处理与重试](#312-错误处理与重试)
- [3.13 构建系统](#313-构建系统)
- [3.14 遥测与可观测性](#314-遥测与可观测性)
- [3.15 认证与授权](#315-认证与授权)
- [3.16 IDE 集成](#316-ide-集成)
- [3.17 实时通信](#317-实时通信)

### 第 4 章 对比分析

- [4.1 架构哲学差异](#41-架构哲学差异)
- [4.2 Agent Loop 差异](#42-agent-loop-差异)
- [4.3 工具系统差异](#43-工具系统差异)
- [4.4 上下文管理差异](#44-上下文管理差异)
- [4.5 安全模型差异](#45-安全模型差异)
- [4.6 多 Agent 系统差异](#46-多-agent-系统差异)
- [4.7 配置系统对比](#47-配置系统对比)
- [4.8 认证系统对比](#48-认证系统对比)
- [4.9 遥测对比](#49-遥测对比)
- [4.10 IDE 集成对比](#410-ide-集成对比)
- [4.11 错误处理对比](#411-错误处理对比)
- [4.12 技术栈与工程实践差异](#412-技术栈与工程实践差异)
- [4.13 综合对比表](#413-综合对比表)

### 第 5 章 关键设计模式总结

- [5.1 Claude Code 核心设计模式](#51-claude-code-核心设计模式)
- [5.2 Codex CLI 核心设计模式](#52-codex-cli-核心设计模式)
- [5.3 可借鉴的架构思想](#53-可借鉴的架构思想)
- [5.4 架构演进趋势](#54-架构演进趋势)

### 第 6 章 结论

- [6.1 核心发现总结](#61-核心发现总结)
- [6.2 适用场景建议](#62-适用场景建议)
- [6.3 未来展望](#63-未来展望)

### 参考来源

---

## 1. 项目概览

### 1.1 基本信息

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **开发商** | Anthropic | OpenAI |
| **仓库地址** | [github.com/anthropics/claude-code](https://github.com/anthropics/claude-code) | [github.com/openai/codex](https://github.com/openai/codex) |
| **许可证** | 闭源 | Apache 2.0 开源 |
| **主要语言** | TypeScript（严格模式） | Rust（从 TypeScript 迁移） |
| **运行时** | Bun | 原生二进制 |
| **代码规模** | ~50 万行，1884 个 TS 文件 | ~8 万行 Rust，60+ Crate |
| **内置工具** | 40+ | 25+ |
| **斜杠命令** | 87+ | N/A |
| **React Hooks** | 70+ | N/A |
| **后台服务** | 13 个子系统 | N/A |
| **API 后端** | Anthropic / Bedrock / Vertex / Foundry | OpenAI（支持 Ollama/LM Studio 本地模型） |
| **Stars** | — | 75,000+ |
| **贡献者** | — | 421+ |

### 1.2 技术栈对比

| 组件 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **主语言** | TypeScript（Bun 运行时） | Rust（Edition 2024） |
| **UI 框架** | React + Ink（终端渲染） | Ratatui 0.29 + crossterm 0.28 |
| **CLI 解析** | Commander.js | clap 4 |
| **异步运行时** | Bun 内置 | Tokio 1 |
| **HTTP 客户端** | 内置 fetch | reqwest 0.12 |
| **Schema 验证** | Zod v4 | serde + serde_json |
| **遥测** | OpenTelemetry + GrowthBook + Statsig + Sentry | 内置 OpenTelemetry SDK |
| **构建** | Bun bundle（特性标志死代码消除） | Bazel 9（CI）+ Cargo（开发） |
| **代码高亮** | 内置 | tree-sitter |
| **MCP 协议** | 自研实现（4 种传输） | rmcp 0.12 |
| **终端样式** | Chalk | crossterm |
| **可复现构建** | — | Nix（flake.nix） |

---

# Claude Code 架构深度分析

> 本章基于对 Anthropic Claude Code 源码的系统性逆向分析，从分层架构、核心模块、执行流程、设计模式等维度进行深度剖析。Claude Code 是一个约 50 万行 TypeScript 代码的复杂系统，运行在 Bun 运行时上，采用 React/Ink 实现终端 UI，包含 40+ 内置工具、87+ 斜杠命令、70+ React Hooks 和 13 个后台子系统。

---

## 2.1 六层分层架构

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

**六层职责说明：**

| 层级 | 位置 | 职责 | 生命周期 |
|------|------|------|----------|
| **UI 层** | `src/components/`, `src/screens/` | 纯渲染层，React/Ink 组件，声明式终端 UI | 会话级 |
| **Hooks 层** | `src/hooks/` (70+ hooks) | 封装副作用和状态逻辑，可被多个 UI 组件复用 | 会话级 |
| **State 层** | `src/state/`, `src/bootstrap/state.ts` | 进程级生命周期状态（全局单例 + Zustand store） | 进程级 |
| **Query 层** | `src/query.ts`, `src/QueryEngine.ts` | 单轮对话的瞬态状态管理，async generator 核心循环 | 轮次级 |
| **Services 层** | `src/services/` (13 个子系统) | 无状态能力提供者（API 客户端、压缩算法、MCP 协议等） | 进程级 |
| **Tools 层** | `src/tools/` (40+ 工具) | 具有身份标识的执行单元（名称、描述、权限要求） | 轮次级 |

### 2.1.1 层间数据流说明

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

### 2.1.2 状态生命周期管理策略

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

### 2.1.3 为什么选择六层而非传统 MVC

传统 MVC（Model-View-Controller）模式是为请求-响应式 Web 应用设计的，其核心假设是：
- 请求之间无状态（或通过 session 保持最小状态）
- 控制器处理请求后立即返回响应
- 视图根据模型数据渲染

Claude Code 的场景完全不同，六层架构的设计理由如下：

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

## 2.2 核心目录结构

```
claudecode/src/
├── main.tsx                 # 应用主入口（REPL 模式，~4,683行）
├── QueryEngine.ts           # SDK/headless 查询引擎入口 (~1,295行)
├── query.ts                 # 核心 Agent 循环 (~785KB) — async generator
├── Tool.ts                  # 工具类型接口 + ToolUseContext (792行)
├── tools.ts                 # 工具注册表 (getAllBaseTools/getTools/assembleToolPool)
├── commands.ts              # 命令注册表 + getSlashCommandToolSkills()
├── context.ts               # 全局 Context 工厂
├── cost-tracker.ts          # Token 费用追踪
├── replLauncher.tsx         # REPL 启动器
├── setup.ts                 # 初始化设置
├── Task.ts                  # 后台任务基类定义
├── tasks.ts                 # 任务系统入口
├── ink.ts                   # Ink 渲染引擎入口
├── history.ts               # 会话历史管理
│
├── entrypoints/             # 多入口点
│   ├── cli.tsx              # CLI 会话编排（从启动到 REPL 的主路径）
│   ├── init.ts              # 配置、遥测、OAuth、MDM 策略初始化
│   ├── mcp.ts               # MCP 服务器模式入口
│   └── sdk/                 # Agent SDK — 嵌入式编程 API
│
├── query/                   # 查询子模块
│   ├── config.ts            # QueryConfig 类型
│   ├── deps.ts              # 依赖注入 (callModel/microcompact/autocompact/uuid)
│   ├── stopHooks.ts         # 停止钩子处理
│   └── tokenBudget.ts       # Token 预算追踪
│
├── tools/                   # 40+ 工具实现
│   ├── AgentTool/           # 子 Agent 工具
│   ├── BashTool/            # Shell 命令执行
│   ├── FileReadTool/        # 文件读取
│   ├── FileEditTool/        # 文件编辑（精确替换）
│   ├── FileWriteTool/       # 文件写入
│   ├── GlobTool/            # 文件模式搜索
│   ├── GrepTool/            # 内容搜索 (ripgrep)
│   ├── MCPTool/             # MCP 工具桥接
│   ├── SkillTool/           # 技能执行
│   ├── ToolSearchTool/      # 工具懒加载发现
│   ├── WebFetchTool/        # Web 抓取
│   ├── WebSearchTool/       # Web 搜索
│   ├── TaskCreateTool/      # 任务创建
│   ├── SendMessageTool/     # 消息发送
│   ├── TeamCreateTool/      # 团队创建
│   └── ...                  # 更多工具
│
├── commands/                # 87+ 斜杠命令实现 (101 个子目录)
│   ├── commit.ts            # /commit
│   ├── clear/               # /clear
│   ├── compact/             # /compact
│   ├── config/              # /config
│   └── ...                  # 更多命令
│
├── services/                # 后台服务层 (13 个子系统)
│   ├── api/                 # API 客户端 (client.ts/claude.ts/withRetry.ts)
│   ├── analytics/           # 遥测分析 (GrowthBook + Statsig + OTel)
│   ├── compact/             # 上下文压缩 (micro/auto/reactive/snip)
│   ├── mcp/                 # MCP 协议实现 (config/transport/auth/lazy loading)
│   ├── oauth/               # OAuth 认证 (PKCE 流程)
│   ├── tools/               # 工具编排层
│   ├── lsp/                 # LSP 集成
│   ├── plugins/             # 插件服务
│   ├── autoDream/           # 自动梦境（会话间自主任务）
│   ├── extractMemories/     # 记忆提取
│   ├── SessionMemory/       # 会话记忆
│   └── voice.ts             # 语音服务
│
├── components/              # React/Ink UI 组件库 (~140 组件)
├── screens/                 # 全屏视图组件
│   ├── REPL.tsx             # 主交互式 REPL
│   ├── Doctor.tsx           # 环境诊断
│   └── ResumeConversation.tsx # 会话恢复
│
├── hooks/                   # React Hooks (70+)
│   ├── useCanUseTool.tsx    # 核心权限决策 Hook
│   ├── useTextInput.ts      # 文本输入
│   ├── useVimInput.ts       # Vim 模式输入
│   ├── toolPermission/      # 工具权限 UI 子系统
│   └── notifs/              # 通知子系统
│
├── skills/                  # 技能系统
│   └── bundled/             # 17 个内置技能
│
├── coordinator/             # 协调器模式（多 Worker 编排）
├── bridge/                  # Bridge 协议（IDE 双向通信，33 文件）
├── schemas/                 # Zod 验证 schema
├── state/                   # 状态管理 (AppState + Zustand store)
├── types/                   # 类型定义
├── utils/                   # 工具函数库（最大子目录，100+ 文件）
│   ├── permissions/         # 权限实现 (24 文件)
│   ├── model/               # 模型选择和路由
│   ├── memory/              # 记忆管理
│   ├── settings/            # 设置加载
│   ├── shell/               # Shell 工具
│   └── sandbox/             # 沙箱系统
│
├── vim/                     # Vim 模式实现（完整状态机）
├── voice/                   # 语音系统
├── memdir/                  # 记忆目录系统
├── plugins/                 # 插件系统入口
├── keybindings/             # 键绑定系统（50+ 动作）
├── constants/               # 全局常量
├── context/                 # Context 管理
├── remote/                  # 远程会话
├── server/                  # 嵌入式服务器 (LSP, Bridge)
├── bootstrap/               # 启动引导
├── native-ts/               # 原生 TypeScript 模块 (FFI 桥接)
└── ink/                     # Ink 渲染引擎扩展
```

### 2.2.1 关键文件详细说明

#### `src/query.ts` (~785KB) -- 主 Agent 循环

这是整个 Claude Code 最核心的文件，虽然体积巨大（~785KB，可能是打包或包含大量内联数据），但其核心逻辑是 `query()` async generator 函数。该函数实现了完整的 ReAct（Reasoning + Acting）循环：

- **入口**：`query()` 接收用户消息、工具列表、消息历史等参数
- **循环体**：`while (true)` 循环，每次迭代调用 API -> 处理流式响应 -> 执行工具 -> 检查终止条件
- **输出**：通过 `yield` 输出 `StreamEvent`（文本增量、工具调用开始/结束、思考增量等）
- **状态管理**：在闭包中维护 `messages[]` 数组、`abortController`、重试计数器等轮次级状态

```typescript
// query.ts 核心结构（简化示意）
export async function* query(
  params: QueryParams
): AsyncGenerator<StreamEvent | Message, void, undefined> {
  const { messages, tools, abortController, ... } = params;

  while (true) {
    // 1. 微压缩检查
    if (shouldMicroCompact(messages)) {
      microCompact(messages);
    }

    // 2. 调用 API（流式）
    const response = yield* queryModelWithStreaming({
      messages,
      tools,
      systemPrompt,
      abortController,
    });

    // 3. 收集工具调用
    const toolUseBlocks = response.message.content.filter(
      block => block.type === 'tool_use'
    );

    // 4. 如果没有工具调用，结束循环
    if (toolUseBlocks.length === 0) {
      return response.message;
    }

    // 5. 执行工具并收集结果
    const toolResults = yield* runTools(toolUseBlocks, ...);

    // 6. 将工具结果追加到消息历史
    messages.push({
      role: 'user',
      content: toolResults.map(r => ({
        type: 'tool_result',
        tool_use_id: r.toolUseId,
        content: r.content,
      })),
    });

    // 7. 检查终止条件
    if (response.stop_reason === 'end_turn') {
      return response.message;
    }
  }
}
```

#### `src/QueryEngine.ts` (~1,295行) -- 外层循环

QueryEngine 是 `query()` 的外层包装器，每个会话创建一个实例，负责：

- **会话级状态管理**：持有消息历史、文件缓存、成本追踪器
- **预算执行**：控制 token 使用量，在接近上限时触发压缩
- **重试逻辑**：处理 API 错误、速率限制、max-output-tokens 恢复
- **权限状态管理**：跟踪当前权限模式、拒绝计数
- **SDK 模式支持**：提供 `sendMessage()` / `streamMessage()` 等编程接口

```typescript
// QueryEngine.ts 核心结构（简化示意）
export class QueryEngine {
  private messages: Message[] = [];
  private fileCache: LRUCache<string, FileContent>;
  private costTracker: CostTracker;
  private abortController: AbortController;
  private permissionState: PermissionState;
  private retryCount: number = 0;

  async sendMessage(userMessage: string): Promise<void> {
    this.messages.push({ role: 'user', content: userMessage });

    const queryGenerator = query({
      messages: this.messages,
      tools: this.getAvailableTools(),
      abortController: this.abortController,
      microCompact: this.microCompact.bind(this),
      autoCompact: this.autoCompact.bind(this),
    });

    for await (const event of queryGenerator) {
      if (event.type === 'text_delta') {
        this.emit('text', event.delta);
      } else if (event.type === 'tool_result') {
        this.messages.push(event.message);
      }
    }
  }
}
```

#### `src/Tool.ts` -- Tool 接口定义

定义了所有工具必须实现的接口，是工具系统的"宪法"：

```typescript
export interface Tool {
  // 身份标识
  name: string;
  description: string;
  inputJSONSchema: JSONSchema;

  // 核心执行
  call(input: ToolInput, context: ToolUseContext): Promise<ToolResult>;

  // 可选能力
  validateInput?(input: unknown): ValidationResult;
  checkPermissions?(input: ToolInput, context: ToolUseContext): PermissionResult;

  // 并发与安全属性
  isConcurrencySafe(input: ToolInput): boolean;
  isReadOnly?: boolean;
  isDestructive?: boolean;

  // 生命周期
  isEnabled?(): boolean;
  canUseInNonInteractive?: boolean;

  // 延迟加载
  shouldDefer?: boolean;
  alwaysLoad?: boolean;

  // LLM 自动分类器输入
  toAutoClassifierInput?(input: ToolInput): string;
}
```

#### `src/services/tools/StreamingToolExecutor.ts` (~530行) -- 并行工具执行器

这是 Claude Code 中最精巧的组件之一，实现了"边接收边执行"的并行工具调用机制。详见 [2.3.3 节](#233-streamingtoolexecutor-并行执行机制)。

#### `src/services/compact/microCompact.ts` -- 微压缩

每次 Agent 循环迭代前执行，清除旧的工具结果以释放上下文空间：

```typescript
// microCompact.ts 核心逻辑（简化示意）
export function microCompact(messages: Message[]): Message[] {
  const result = [...messages];
  let clearedCount = 0;
  const maxClear = 20; // 最多清除20个旧工具结果

  for (let i = 0; i < result.length && clearedCount < maxClear; i++) {
    const msg = result[i];
    if (msg.role === 'user' && Array.isArray(msg.content)) {
      const newContent = msg.content.map(block => {
        if (block.type === 'tool_result' && isOldEnough(block, MAX_AGE)) {
          clearedCount++;
          return {
            ...block,
            content: '[Old tool result content cleared]',
          };
        }
        return block;
      });
      result[i] = { ...msg, content: newContent };
    }
  }
  return result;
}
```

#### `src/services/compact/compact.ts` -- 自动压缩

当 token 使用量超过阈值（约 167K）时触发，将旧消息发送给模型进行摘要：

```typescript
// compact.ts 核心逻辑（简化示意）
export async function autoCompact(
  messages: Message[],
  model: string,
  apiClient: AnthropicClient,
): Promise<Message[]> {
  // 1. 找到压缩边界（保留最近N条消息不压缩）
  const boundaryIndex = findCompactBoundary(messages, KEEP_RECENT);

  // 2. 将边界前的消息发送给模型摘要
  const oldMessages = messages.slice(0, boundaryIndex);
  const summary = await generateSummary(oldMessages, model, apiClient);

  // 3. 替换为压缩边界标记
  return [
    {
      role: 'user',
      content: `[Previous conversation summary]\n${summary}`,
    },
    ...messages.slice(boundaryIndex),
  ];
}
```

#### `src/services/api/withRetry.ts` -- 重试逻辑

实现了指数退避 + 抖动的重试策略，处理 API 调用的瞬态错误：

```typescript
// withRetry.ts 核心逻辑（简化示意）
export async function withRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions = {},
): Promise<T> {
  const {
    maxRetries = 3,
    baseDelay = 1000,
    maxDelay = 30000,
    retryableErrors = [429, 529, 500, 502, 503],
  } = options;

  let lastError: Error;
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      const status = (error as ApiError).status;

      if (!retryableErrors.includes(status) && attempt < maxRetries) {
        throw error; // 不可重试的错误直接抛出
      }

      // 指数退避 + 抖动
      const delay = Math.min(
        baseDelay * Math.pow(2, attempt) + Math.random() * 1000,
        maxDelay,
      );

      // 尊重 retry-after 头
      const retryAfter = (error as ApiError).headers?.['retry-after'];
      const actualDelay = retryAfter
        ? parseInt(retryAfter) * 1000
        : delay;

      await sleep(actualDelay);
    }
  }
  throw lastError;
}
```

#### `src/services/api/errors.ts` -- 错误分类

将 API 错误分类为不同类型，以便上层采取不同的恢复策略：

```typescript
// errors.ts 错误分类（简化示意）
export function classifyError(error: unknown): ErrorCategory {
  if (!(error instanceof ApiError)) {
    return 'network_error';
  }

  const { status, message } = error;

  if (status === 429) return 'rate_limit';
  if (status === 529) return 'overloaded';
  if (status >= 500 && status < 600) return 'server_error';
  if (status === 401) return 'auth_error';
  if (status === 403) return 'forbidden';
  if (message.includes('prompt_too_long')) return 'context_overflow';

  return 'unknown';
}
```

#### `src/services/analytics/growthbook.ts` -- GrowthBook 特性标志

GrowthBook 是 Claude Code 的特性标志系统，支持编译时死代码消除（DCE）和运行时远程配置：

```typescript
// growthbook.ts 核心逻辑（简化示意）
import { GrowthBook } from '@growthbook/growthbook';

export function initGrowthBook(attributes: GrowthBookAttributes): GrowthBook {
  const gb = new GrowthBook({
    apiHost: 'https://cdn.growthbook.io',
    clientKey: process.env.GROWTHBOOK_CLIENT_KEY,
    attributes: {
      organizationUUID: attributes.orgId,
      accountUUID: attributes.accountId,
      email: attributes.email,
      platform: process.platform,
      claudeCodeVersion: VERSION,
    },
    enableDevMode: false,
    trackingCallback: (experiment, result) => {
      statsig.logEvent('feature_flag_evaluation', {
        experimentId: experiment.key,
        variationId: result.variationId,
      });
    },
  });

  // 初始化后开始远程配置轮询
  gb.init().then(() => {
    gb.setAttributes(updatedAttributes);
  });

  return gb;
}

// 特性标志使用示例
export function isFeatureEnabled(gb: GrowthBook, flag: string): boolean {
  // 编译时 DCE：bun:bundle 的 feature() 在构建时剥离
  if (feature('COORDINATOR_MODE') && flag === 'coordinator') return true;
  // 运行时检查
  return gb.isOn(flag);
}
```

#### `src/bootstrap/state.ts` -- Bootstrap 全局状态

进程级全局状态的单一定义点，在应用启动时初始化：

```typescript
// state.ts 核心结构（简化示意）
export interface BootstrapState {
  // 目录与会话
  cwd: string;
  homeDir: string;
  sessionId: string;
  conversationId: string;

  // 成本追踪
  totalCost: number;
  totalInputTokens: number;
  totalOutputTokens: number;

  // 性能指标
  startupTime: number;
  apiLatencyP50: number;
  apiLatencyP99: number;

  // 认证安全
  authToken: string | null;
  authMethod: 'oauth' | 'api_key' | 'bare' | null;
  mdmPolicy: MDMPolicy | null;

  // 遥测
  otelExporter: OTelExporter | null;
  statsigClient: StatsigClient | null;
  growthBook: GrowthBook | null;

  // Hooks
  hooks: Map<HookEvent, Hook[]>;

  // 特性标志
  featureFlags: Map<string, boolean>;

  // 设置
  settings: Settings;
}

// 全局单例
let globalState: BootstrapState | null = null;

export function getBootstrapState(): BootstrapState {
  if (!globalState) {
    throw new Error('Bootstrap state not initialized');
  }
  return globalState;
}

export function initBootstrapState(partial: Partial<BootstrapState>): void {
  globalState = { ...defaultState, ...partial };
}
```

#### `src/state/AppStateStore.ts` -- AppState Store

会话级状态管理，基于 Zustand-like 的自定义 store 实现：

```typescript
// AppStateStore.ts 核心结构（简化示意）
export interface AppState {
  // 会话
  messages: Message[];
  isStreaming: boolean;
  currentModel: string;

  // 权限
  permissionMode: PermissionMode;
  deniedCount: number;
  consecutiveDenials: number;

  // UI
  inputText: string;
  notifications: Notification[];
  activeModal: ModalType | null;

  // 成本
  sessionCost: number;
  sessionInputTokens: number;
  sessionOutputTokens: number;

  // 操作
  addMessage: (msg: Message) => void;
  setPermissionMode: (mode: PermissionMode) => void;
  incrementDeniedCount: () => void;
  setStreaming: (streaming: boolean) => void;
}
```

#### `src/state/store.ts` (~20行) -- Zustand-like Store 实现

Claude Code 没有直接使用 Zustand，而是实现了一个极简的响应式 store（约 20 行代码）：

```typescript
// store.ts -- 极简响应式 Store 实现（约20行）
type Listener<T> = (state: T) => void;

export function createStore<T extends object>(initialState: T) {
  let state = initialState;
  const listeners = new Set<Listener<T>>();

  return {
    getState: () => state,
    setState: (partial: Partial<T>) => {
      state = { ...state, ...partial };
      listeners.forEach(listener => listener(state));
    },
    subscribe: (listener: Listener<T>) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
  };
}
```

这个实现非常精简，核心思想是：
- `getState()` 返回当前状态的快照
- `setState()` 合并部分状态并通知所有订阅者
- `subscribe()` 注册监听器，返回取消订阅函数

#### `src/utils/settings/settings.ts` -- 配置加载

负责从多个来源加载和合并配置：

```typescript
// settings.ts 核心逻辑（简化示意）
export async function loadSettings(): Promise<Settings> {
  // 5 级优先级加载
  const [flagSettings, localSettings, projectSettings,
         userSettings, policySettings] = await Promise.all([
    loadFeatureFlagSettings(),    // 最低优先级
    loadLocalSettings(),          // .claude/settings.local.json
    loadProjectSettings(),        // .claude/settings.json
    loadUserSettings(),           // ~/.claude/settings.json
    loadPolicySettings(),         // MDM 策略（最高优先级）
  ]);

  // 按优先级合并
  return mergeWith(
    {},
    flagSettings,
    localSettings,
    projectSettings,
    userSettings,
    policySettings,
    settingsMergeCustomizer,
  );
}
```

#### `src/utils/sessionRestore.ts` -- 会话恢复

从磁盘恢复之前的会话状态：

```typescript
// sessionRestore.ts 核心逻辑（简化示意）
export async function restoreSession(
  sessionId: string,
): Promise<SessionRestoreResult> {
  // 1. 读取会话元数据
  const metadata = await readSessionMetadata(sessionId);

  // 2. 读取 JSONL 转录
  const transcript = await readTranscript(sessionId);

  // 3. 重建消息历史
  const messages = transcript
    .filter(event => event.type === 'message')
    .map(event => event.message);

  // 4. 恢复工具注册表和权限状态
  const tools = await rebuildToolRegistry(metadata.toolState);
  const permissions = restorePermissionState(metadata.permissionState);

  return { messages, tools, permissions, metadata };
}
```

#### `src/utils/sessionStorage.ts` -- 会话持久化

将会话状态持久化到磁盘：

```typescript
// sessionStorage.ts 核心逻辑（简化示意）
export async function persistSession(
  sessionId: string,
  state: SessionState,
): Promise<void> {
  // 1. 写入会话元数据 (sessions.json)
  await appendSessionMetadata(sessionId, {
    id: sessionId,
    lastUpdated: Date.now(),
    messageCount: state.messages.length,
    totalTokens: state.totalTokens,
  });

  // 2. 追加事件到 JSONL 转录
  await appendToTranscript(sessionId, {
    type: 'message',
    timestamp: Date.now(),
    message: state.messages[state.messages.length - 1],
  });
}
```

#### `src/main.tsx` (~4,683行) -- REPL 引导程序

应用的主入口文件，虽然体积巨大（~4,683行），但核心职责是引导整个应用启动序列：

```typescript
// main.tsx 启动序列（简化示意）
async function main() {
  startupProfiler.checkpoint('start');

  // 1. 初始化（配置、遥测、OAuth、MDM）
  await init();

  // 2. 加载认证
  const auth = await loadAuth();

  // 3. 加载特性标志
  const growthBook = await loadGrowthBook(auth);

  // 4. 检查配额
  await checkQuota(auth);

  // 5. 获取系统上下文（git 状态、分支信息）
  const systemContext = await getSystemContext();

  // 6. 获取用户上下文（CLAUDE.md 文件、日期）
  const userContext = await getUserContext();

  // 7. 加载工具注册表
  const tools = getAllBaseTools();

  // 8. 加载斜杠命令和技能
  const commands = getCommands();

  // 9. 启动 REPL
  await launchRepl({
    auth,
    growthBook,
    systemContext,
    userContext,
    tools,
    commands,
  });
}

main().catch(handleFatalError);
```

启动时执行**并行预取**：MDM 策略读取、Keychain 预取、特性标志检查，然后进行核心初始化。重模块通过 `dynamic import()` 延迟加载（如 OpenTelemetry ~400KB、gRPC ~700KB）。

---

## 2.3 Agent 循环（大幅扩充）

Claude Code 的 Agent 循环是整个系统的"心脏"，采用**流式异步生成器（AsyncGenerator）双层循环**模型。这一节将深入分析其每个组件的实现细节。

### 2.3.1 `query()` async generator 核心结构

`query()` 是 Claude Code 的核心函数，以 async generator 的形式实现了一个完整的 ReAct（Reasoning + Acting）循环。它的设计精妙之处在于：通过 `yield` 将流式事件实时推送给 UI，同时保持内部循环状态。

```typescript
// query.ts -- query() 完整伪代码
export async function* query(
  params: QueryParams,
): AsyncGenerator<StreamEvent | Message, void, undefined> {
  const {
    messages,           // 消息历史（可变引用）
    tools,              // 可用工具列表
    systemPrompt,       // 系统提示词
    abortController,    // 中止控制器
    model,              // 模型名称
    maxTokens,          // 最大输出 token 数
    microCompact,       // 微压缩函数
    autoCompact,        // 自动压缩函数
    callModel,          // API 调用函数（依赖注入）
  } = params;

  let maxOutputTokensRetries = 0;
  const MAX_OUTPUT_TOKENS_RETRIES = 3;

  // ═══════════════════════════════════════════════════════════════
  // 主循环 -- 持续运行直到模型决定结束或用户中止
  // ═══════════════════════════════════════════════════════════════
  while (true) {
    // ─── 步骤 1: 检查中止信号 ─────────────────────────────────
    if (abortController.signal.aborted) {
      yield { type: 'aborted' };
      return;
    }

    // ─── 步骤 2: 微压缩 ──────────────────────────────────────
    // 每次迭代前检查是否需要微压缩
    // 微压缩是轻量级的：仅清除旧的工具结果，不调用 LLM
    if (microCompact) {
      const compacted = microCompact(messages);
      if (compacted) {
        yield { type: 'micro_compact', clearedCount: compacted.clearedCount };
      }
    }

    // ─── 步骤 3: 调用 LLM API（流式） ─────────────────────────
    // yield* 将子 generator 的事件直接传递给调用者
    const response = yield* queryModelWithStreaming({
      model,
      maxTokens,
      messages,
      tools: tools.map(t => ({
        name: t.name,
        description: t.description,
        input_schema: t.inputJSONSchema,
      })),
      systemPrompt,
      abortController,
    });

    // ─── 步骤 4: 处理响应 ─────────────────────────────────────
    const { message, stop_reason } = response;

    // 将助手消息追加到历史
    messages.push(message);

    // ─── 步骤 5: 检查终止条件 ─────────────────────────────────
    if (stop_reason === 'end_turn') {
      // 模型认为任务完成
      yield { type: 'end_turn', message };
      return;
    }

    if (stop_reason === 'max_tokens') {
      // 模型输出被截断，需要恢复
      maxOutputTokensRetries++;
      if (maxOutputTokensRetries >= MAX_OUTPUT_TOKENS_RETRIES) {
        yield { type: 'max_tokens_exceeded', message };
        return;
      }
      // 继续循环，让模型从截断处继续
      yield { type: 'max_tokens_retry', attempt: maxOutputTokensRetries };
      continue;
    }

    // ─── 步骤 6: 收集工具调用 ─────────────────────────────────
    const toolUseBlocks = message.content.filter(
      (block): block is ToolUseBlock => block.type === 'tool_use'
    );

    if (toolUseBlocks.length === 0) {
      // 没有工具调用且不是 end_turn，异常终止
      yield { type: 'unexpected_stop', stop_reason, message };
      return;
    }

    // ─── 步骤 7: 执行工具 ─────────────────────────────────────
    // yield* 将工具执行的事件也传递给调用者
    const toolResults = yield* runTools({
      toolUseBlocks,
      tools,
      messages,
      abortController,
    });

    // ─── 步骤 8: 将工具结果追加到消息历史 ─────────────────────
    messages.push({
      role: 'user',
      content: toolResults.map(result => ({
        type: 'tool_result' as const,
        tool_use_id: result.toolUseId,
        content: result.content,
        is_error: result.isError,
      })),
    });

    // ─── 步骤 9: 检查自动压缩 ─────────────────────────────────
    // 自动压缩是重量级的：调用 LLM 生成摘要
    if (autoCompact && shouldAutoCompact(messages)) {
      const compacted = await autoCompact(messages);
      if (compacted) {
        messages.length = 0;
        messages.push(...compacted);
        yield { type: 'auto_compact', newMessageCount: compacted.length };
      }
    }

    // ─── 继续循环 ─────────────────────────────────────────────
    // 工具结果已追加，下一轮迭代将让模型看到工具执行结果
  }
}
```

**关键设计决策：**

1. **`yield*` 委托**：`queryModelWithStreaming()` 和 `runTools()` 都是 async generator，使用 `yield*` 将它们的事件直接传递给外层消费者，避免事件缓冲延迟
2. **原地修改 messages**：`messages` 数组是通过引用传递的，`query()` 直接修改它，调用者（QueryEngine）持有的引用也会看到变化
3. **微压缩在循环头部**：确保每次 API 调用前上下文尽可能精简
4. **自动压缩在循环尾部**：在工具结果追加后检查，因为工具结果可能显著增加上下文大小
5. **max_tokens 恢复**：最多重试 3 次，避免无限循环

### 2.3.2 `queryModelWithStreaming()` 实现

该函数负责构建 API 请求、处理流式响应、并 yield 事件：

```typescript
// services/api/claude.ts -- queryModelWithStreaming() 完整实现
async function* queryModelWithStreaming(
  params: StreamParams,
): AsyncGenerator<StreamEvent, ApiResponse, undefined> {
  const {
    model, maxTokens, messages, tools,
    systemPrompt, abortController,
  } = params;

  // ─── 构建 API 请求体 ──────────────────────────────────────
  const request: CreateMessageRequest = {
    model,
    max_tokens: maxTokens,
    messages: normalizeMessages(messages),
    system: buildSystemPrompt(systemPrompt),
    tools: tools.map(tool => ({
      name: tool.name,
      description: tool.description,
      input_schema: tool.input_schema,
      cache_control: { type: 'ephemeral' },
    })),
    thinking: model.includes('claude-3.7') ? {
      type: 'enabled',
      budget_tokens: Math.floor(maxTokens * 0.8),
    } : undefined,
    metadata: { user_id: getUserId() },
  };

  // ─── 系统提示词缓存优化 ───────────────────────────────────
  if (request.system && Array.isArray(request.system)) {
    request.system.forEach((block, index) => {
      if (block.type === 'text') {
        if (index === 0) {
          block.cache_control = { type: 'ephemeral' };
        }
      }
    });
  }

  // ─── 调用 Anthropic SDK stream API ─────────────────────────
  let stream: AnthropicStream;
  try {
    stream = await anthropic.messages.stream(request, {
      signal: abortController.signal,
      headers: { 'anthropic-beta': 'interleaved-thinking-2025-05-14' },
    });
  } catch (error) {
    yield { type: 'api_error', error };
    throw error;
  }

  // ─── 处理流式响应 ─────────────────────────────────────────
  const contentBlocks: ContentBlock[] = [];
  let currentBlock: Partial<ContentBlock> | null = null;
  let stopReason: StopReason | null = null;

  try {
    for await (const event of stream) {
      if (abortController.signal.aborted) {
        stream.abort();
        yield { type: 'aborted' };
        return;
      }

      switch (event.type) {
        case 'message_start':
          yield { type: 'message_start', message: event.message,
                  usage: event.usage };
          break;

        case 'content_block_start':
          currentBlock = {
            type: event.content_block.type,
            id: event.content_block.id,
          };
          if (event.content_block.type === 'tool_use') {
            currentBlock.name = event.content_block.name;
            currentBlock.input = {};
            yield { type: 'tool_use_start',
                    id: event.content_block.id,
                    name: event.content_block.name };
          } else if (event.content_block.type === 'thinking') {
            yield { type: 'thinking_start',
                    id: event.content_block.id };
          } else if (event.content_block.type === 'text') {
            yield { type: 'text_start',
                    id: event.content_block.id };
          }
          break;

        case 'content_block_delta':
          if (event.delta.type === 'text_delta') {
            yield { type: 'text_delta', delta: event.delta.text,
                    id: currentBlock?.id };
            if (currentBlock) {
              currentBlock.text = (currentBlock.text || '') + event.delta.text;
            }
          } else if (event.delta.type === 'thinking_delta') {
            yield { type: 'thinking_delta', delta: event.delta.thinking,
                    id: currentBlock?.id };
          } else if (event.delta.type === 'input_json_delta') {
            if (currentBlock) {
              currentBlock.inputJson =
                (currentBlock.inputJson || '') + event.delta.partial_json;
            }
            yield { type: 'tool_input_delta', id: currentBlock?.id,
                    delta: event.delta.partial_json };
          }
          break;

        case 'content_block_stop':
          if (currentBlock) {
            if (currentBlock.type === 'tool_use' && currentBlock.inputJson) {
              try {
                currentBlock.input = JSON.parse(currentBlock.inputJson);
              } catch {
                currentBlock.input = {};
                currentBlock.parseError = true;
              }
            }
            contentBlocks.push(currentBlock as ContentBlock);
            currentBlock = null;
          }
          break;

        case 'message_delta':
          stopReason = event.delta.stop_reason;
          yield { type: 'message_delta', delta: event.delta,
                  usage: event.usage };
          break;

        case 'message_stop':
          yield { type: 'message_stop' };
          break;
      }
    }
  } catch (error) {
    if (abortController.signal.aborted) {
      yield { type: 'aborted' };
      return;
    }
    yield { type: 'stream_error', error };
    throw error;
  }

  return {
    message: { role: 'assistant', content: contentBlocks },
    stop_reason: stopReason!,
    usage: stream.finalMessage()?.usage,
  };
}
```

### 2.3.3 StreamingToolExecutor 并行执行机制

`StreamingToolExecutor`（~530行）是 Claude Code 中最精巧的组件之一。它实现了一个**事件驱动的状态机**，能够在模型仍在生成响应时就开始执行已完成的工具调用，显著减少端到端延迟。

#### 设计原理

传统的 Agent 循环是"等待完整响应 -> 解析所有工具调用 -> 逐个执行"。StreamingToolExecutor 打破了这个限制：

```
传统模式：
  模型生成 [tool1参数...] [tool2参数...] [tool3参数...] -> 全部完成 -> 执行tool1 -> 执行tool2 -> 执行tool3

StreamingToolExecutor 模式：
  模型生成 [tool1参数完成] -> 立即执行tool1
           [tool2参数完成] -> 立即执行tool2（与tool1并行）
           [tool3参数完成] -> 等待tool1/tool2完成后执行tool3（如果不安全并行）
```

#### 核心数据结构

```typescript
// StreamingToolExecutor.ts -- 核心数据结构

// 正在接收参数的工具（参数尚未完整）
interface PendingBlock {
  id: string;              // content_block id
  name: string;            // 工具名称
  inputJson: string;       // 已接收的 JSON 字符串片段
  startedAt: number;       // 开始时间戳
}

// 正在运行的工具任务
interface RunningTool {
  id: string;              // content_block id
  name: string;            // 工具名称
  input: unknown;          // 已解析的完整输入
  promise: Promise<ToolResult>; // 执行 Promise
  abortController: AbortController; // 独立的中止控制器
  startedAt: number;       // 开始执行时间戳
}

// 执行结果（带顺序信息）
interface ToolResultWithOrder {
  id: string;
  name: string;
  result: ToolResult;
  order: number;           // 原始出现顺序（保证确定性）
}

export class StreamingToolExecutor {
  // 状态机核心
  private pendingBlocks: Map<string, PendingBlock> = new Map();
  private runningTools: RunningTool[] = [];
  private completedResults: ToolResultWithOrder[] = [];
  private nextOrder: number = 0;

  // 并发控制
  private maxConcurrent: number = 10;
  private siblingAbortController: AbortController;

  // 事件回调
  private onToolStart?: (id: string, name: string) => void;
  private onToolComplete?: (id: string, name: string, result: ToolResult) => void;
  private onToolError?: (id: string, name: string, error: Error) => void;
}
```

#### 事件驱动状态机

```typescript
// StreamingToolExecutor.ts -- 事件处理方法

class StreamingToolExecutor {
  /**
   * 处理来自流式响应的三种事件
   * 这是状态机的核心驱动方法
   */
  onEvent(event: StreamEvent): void {
    switch (event.type) {
      // ─── 工具调用开始：创建 PendingBlock ─────────────────
      case 'content_block_start':
        if (event.content_block.type === 'tool_use') {
          const block: PendingBlock = {
            id: event.content_block.id,
            name: event.content_block.name,
            inputJson: '',
            startedAt: Date.now(),
          };
          this.pendingBlocks.set(block.id, block);
        }
        break;

      // ─── 参数增量：追加到 PendingBlock ────────────────────
      case 'content_block_delta':
        if (event.delta.type === 'input_json_delta') {
          const block = this.pendingBlocks.get(event.id);
          if (block) {
            block.inputJson += event.delta.partial_json;
          }
        }
        break;

      // ─── 工具调用结束：尝试调度执行 ──────────────────────
      case 'content_block_stop':
        const block = this.pendingBlocks.get(event.id);
        if (block) {
          this.pendingBlocks.delete(event.id);

          // 解析完整的 JSON 输入
          let input: unknown;
          try {
            input = JSON.parse(block.inputJson);
          } catch (error) {
            this.completedResults.push({
              id: block.id, name: block.name,
              result: { content: `Error: Invalid JSON input: ${error.message}`,
                        isError: true },
              order: this.nextOrder++,
            });
            break;
          }

          // 调度执行
          this.scheduleExecution(block.id, block.name, input);
        }
        break;
    }
  }
}
```

#### 并发安全的调度执行

```typescript
// StreamingToolExecutor.ts -- scheduleExecution()

class StreamingToolExecutor {
  private async scheduleExecution(
    id: string, name: string, input: unknown,
  ): Promise<void> {
    // ─── 并发安全判断 ───────────────────────────────────────
    const tool = this.findTool(name);
    const isSafe = tool?.isConcurrencySafe(input) ?? false;

    // 如果不安全且已有工具在运行，等待它们完成
    if (!isSafe && this.runningTools.length > 0) {
      await Promise.all(this.runningTools.map(t => t.promise));
    }

    // 检查并发上限
    if (this.runningTools.length >= this.maxConcurrent) {
      await Promise.race(this.runningTools.map(t => t.promise));
    }

    // ─── 创建执行任务 ───────────────────────────────────────
    const abortController = new AbortController();
    const onSiblingAbort = () => abortController.abort();
    this.siblingAbortController.signal.addEventListener('abort', onSiblingAbort);
    const order = this.nextOrder++;

    const promise = this.executeTool(id, name, input, abortController)
      .then(result => {
        this.completedResults.push({ id, name, result, order });
        this.onToolComplete?.(id, name, result);
        return result;
      })
      .catch(error => {
        if (abortController.signal.aborted) return;
        const errorResult: ToolResult = {
          content: `Error: ${error.message}`, isError: true,
        };
        this.completedResults.push({ id, name, result: errorResult, order });
        this.onToolError?.(id, name, error);
      })
      .finally(() => {
        this.runningTools = this.runningTools.filter(t => t.id !== id);
        this.siblingAbortController.signal.removeEventListener('abort', onSiblingAbort);
      });

    this.runningTools.push({
      id, name, input, promise, abortController, startedAt: Date.now(),
    });
    this.onToolStart?.(id, name);
  }
}
```

#### 结果缓冲与顺序保证

```typescript
class StreamingToolExecutor {
  /**
   * 等待所有工具执行完成，并按原始顺序返回结果
   * 这保证了即使并行执行，结果的顺序也是确定性的
   */
  async collectResults(): Promise<ToolResultWithOrder[]> {
    await Promise.all(this.runningTools.map(t => t.promise));
    return [...this.completedResults].sort((a, b) => a.order - b.order);
  }

  abortAll(): void {
    this.siblingAbortController.abort();
    for (const tool of this.runningTools) {
      tool.abortController.abort();
    }
  }
}
```

**结果缓冲机制的关键设计**：
- `nextOrder` 计数器在 `content_block_start` 时递增，记录工具调用的原始顺序
- 并行执行的工具可能以任意顺序完成，但 `collectResults()` 按原始顺序返回
- 这保证了模型看到的工具结果顺序与它发出调用的顺序一致，避免混淆

### 2.3.4 max-output-tokens 恢复机制

当模型的 `stop_reason === 'max_tokens'` 时，说明模型的输出被截断了。Claude Code 的处理逻辑如下：

```typescript
// 在 query() 的 while-true 循环中：
if (stop_reason === 'max_tokens') {
  maxOutputTokensRetries++;

  if (maxOutputTokensRetries >= MAX_OUTPUT_TOKENS_RETRIES) {
    yield {
      type: 'max_tokens_exceeded',
      message: 'Model output was truncated after 3 retries. ' +
               'Consider breaking your task into smaller steps.',
    };
    return;
  }

  // 构建恢复消息：告诉模型从截断处继续
  messages.push({
    role: 'user',
    content: '[Model output was truncated due to max_tokens limit. ' +
             'Please continue from where you left off.]',
  });

  yield {
    type: 'max_tokens_retry',
    attempt: maxOutputTokensRetries,
  };

  continue; // 下一轮迭代将让模型看到截断提示并继续生成
}
```

**恢复机制的设计考虑**：
- **最多 3 次重试**：避免无限循环
- **用户消息而非系统消息**：使用 `role: 'user'` 提示模型继续
- **透明通知**：通过 yield 事件通知 UI 层显示重试状态
- **不增加 max_tokens**：重试时使用相同的 max_tokens 值

### 2.3.5 StreamEvent 类型定义

基于 Anthropic SSE 协议的完整类型定义：

```typescript
// types/stream.ts -- StreamEvent 完整类型定义

type StopReason = 'end_turn' | 'max_tokens' | 'stop_sequence' | 'tool_use';

interface Usage {
  input_tokens: number;
  output_tokens: number;
  cache_creation_input_tokens?: number;
  cache_read_input_tokens?: number;
}

type ContentBlock = TextBlock | ToolUseBlock | ThinkingBlock;

interface TextBlock {
  type: 'text'; id?: string; text: string;
}

interface ToolUseBlock {
  type: 'tool_use'; id: string; name: string;
  input: Record<string, unknown>;
  inputJson?: string; parseError?: boolean;
}

interface ThinkingBlock {
  type: 'thinking'; id: string; thinking: string;
}

interface Message {
  role: 'user' | 'assistant';
  content: string | ContentBlock[];
}

type StreamEvent =
  | MessageStartEvent | TextStartEvent | TextDeltaEvent
  | ThinkingStartEvent | ThinkingDeltaEvent
  | ToolUseStartEvent | ToolInputDeltaEvent
  | MessageDeltaEvent | MessageStopEvent
  | ToolExecutionStartEvent | ToolExecutionCompleteEvent
  | ToolExecutionErrorEvent | MicroCompactEvent | AutoCompactEvent
  | AbortedEvent | MaxTokensRetryEvent | MaxTokensExceededEvent
  | ApiErrorEvent | StreamErrorEvent;

// API 流事件
interface MessageStartEvent { type: 'message_start'; message: AssistantMessage; usage: Usage; }
interface TextStartEvent { type: 'text_start'; id: string; }
interface TextDeltaEvent { type: 'text_delta'; delta: string; id?: string; }
interface ThinkingStartEvent { type: 'thinking_start'; id: string; }
interface ThinkingDeltaEvent { type: 'thinking_delta'; delta: string; id?: string; }
interface ToolUseStartEvent { type: 'tool_use_start'; id: string; name: string; }
interface ToolInputDeltaEvent { type: 'tool_input_delta'; id: string; delta: string; }
interface MessageDeltaEvent { type: 'message_delta'; delta: { stop_reason?: StopReason }; usage: Usage; }
interface MessageStopEvent { type: 'message_stop'; }

// 工具执行事件
interface ToolExecutionStartEvent { type: 'tool_execution_start'; id: string; name: string; input: unknown; }
interface ToolExecutionCompleteEvent { type: 'tool_execution_complete'; id: string; name: string; result: ToolResult; }
interface ToolExecutionErrorEvent { type: 'tool_execution_error'; id: string; name: string; error: Error; }

// 压缩事件
interface MicroCompactEvent { type: 'micro_compact'; clearedCount: number; }
interface AutoCompactEvent { type: 'auto_compact'; newMessageCount: number; }

// 控制事件
interface AbortedEvent { type: 'aborted'; }
interface MaxTokensRetryEvent { type: 'max_tokens_retry'; attempt: number; }
interface MaxTokensExceededEvent { type: 'max_tokens_exceeded'; message: string; }
interface ApiErrorEvent { type: 'api_error'; error: Error; }
interface StreamErrorEvent { type: 'stream_error'; error: Error; }
```

### 2.3.6 QueryEngine 外层循环

QueryEngine 是 `query()` 的外层包装器，提供会话级的管理能力：

```typescript
// QueryEngine.ts -- 外层循环完整实现

export class QueryEngine {
  private messages: Message[] = [];
  private fileCache: LRUCache<string, FileContent>;
  private costTracker: CostTracker;
  private abortController: AbortController = new AbortController();
  private permissionState: PermissionState;
  private tokenBudget: TokenBudget;
  private retryState: RetryState = { count: 0, lastError: null, nextRetryAt: 0 };

  constructor(private deps: QueryEngineDeps, private config: QueryConfig) {
    this.fileCache = new LRUCache({ max: 100, maxSize: 25 * 1024 * 1024 });
    this.costTracker = new CostTracker();
    this.tokenBudget = new TokenBudget(config.maxTokens || 200000);
    this.permissionState = {
      mode: config.permissionMode || 'default',
      deniedCount: 0, consecutiveDenials: 0,
    };
  }

  async sendMessage(userMessage: string): Promise<void> {
    this.messages.push({ role: 'user', content: userMessage });

    const estimatedTokens = estimateMessageTokens(this.messages);
    if (estimatedTokens > this.tokenBudget.getThreshold('compact')) {
      await this.autoCompact();
    }

    let attempt = 0;
    const maxAttempts = 3;

    while (attempt < maxAttempts) {
      try {
        const queryGenerator = query({
          messages: this.messages,
          tools: this.getAvailableTools(),
          abortController: this.abortController,
          model: this.config.model,
          maxTokens: this.config.maxOutputTokens || 16384,
          systemPrompt: this.buildSystemPrompt(),
          microCompact: this.microCompact.bind(this),
          autoCompact: this.shouldAutoCompact.bind(this),
          callModel: this.deps.callModel,
        });

        for await (const event of queryGenerator) {
          this.handleStreamEvent(event);
        }

        this.retryState.count = 0;
        return;

      } catch (error) {
        const category = classifyError(error);
        switch (category) {
          case 'rate_limit':
            await this.sleep(this.getRetryAfter(error));
            attempt++; continue;
          case 'overloaded':
            await this.sleep(Math.pow(2, attempt) * 1000);
            attempt++; continue;
          case 'context_overflow':
            await this.forceCompact();
            attempt++; continue;
          case 'auth_error':
          case 'forbidden':
            throw error;
          default:
            if (attempt >= maxAttempts - 1) throw error;
            attempt++; continue;
        }
      }
    }
  }

  private handleStreamEvent(event: StreamEvent): void {
    switch (event.type) {
      case 'text_delta': this.emit('text', event.delta); break;
      case 'thinking_delta': this.emit('thinking', event.delta); break;
      case 'tool_execution_start':
        this.emit('toolStart', { id: event.id, name: event.name }); break;
      case 'tool_execution_complete':
        this.emit('toolComplete', { id: event.id, name: event.name, result: event.result }); break;
      case 'micro_compact':
        this.emit('compact', { type: 'micro', count: event.clearedCount }); break;
      case 'auto_compact':
        this.emit('compact', { type: 'auto', count: event.newMessageCount }); break;
      case 'api_error': this.emit('error', event.error); break;
    }
  }

  abort(): void { this.abortController.abort(); }

  getStats(): SessionStats {
    return {
      messageCount: this.messages.length,
      totalCost: this.costTracker.getTotalCost(),
      inputTokens: this.costTracker.getInputTokens(),
      outputTokens: this.costTracker.getOutputTokens(),
      cacheHitRate: this.costTracker.getCacheHitRate(),
    };
  }
}
```

---

## 2.4 工具系统（大幅扩充）

### 2.4.1 工具注册表

```
┌─────────────────────────────────────────────────────────────┐
│                      工具注册表                               │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 核心工具 (始终加载)                                   │    │
│  │  BashTool, FileReadTool, FileEditTool, FileWriteTool│    │
│  │  GlobTool, GrepTool, AgentTool, SkillTool           │    │
│  │  TaskCreate/Get/Update/List/Output/Stop             │    │
│  │  EnterPlanMode, ExitPlanMode, WebFetch, WebSearch   │    │
│  │  ToolSearchTool, SendMessageTool                    │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 特性门控工具                                         │    │
│  │  KAIROS:        SendUserFile, PushNotification       │    │
│  │  MONITOR_TOOL:  MonitorTool                          │    │
│  │  COORDINATOR:   TeamCreate, TeamDelete               │    │
│  │  AGENT_TRIGGERS:CronCreate, CronDelete, CronList     │    │
│  │  WORKFLOW:      WorkflowTool                         │    │
│  │  WEB_BROWSER:   WebBrowserTool                      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 延迟加载工具 (通过 ToolSearchTool)                    │    │
│  │  MCP 工具 (全部)                                      │    │
│  │  shouldDefer=true 的工具                              │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 动态 MCP 工具                                         │    │
│  │  运行时从连接的 MCP 服务器发现                         │    │
│  │  名称规范化: mcp__{server}__{tool}                    │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

**工具加载流程**：

```
getAllBaseTools()
    |
    +-- 创建核心工具实例（BashTool, FileReadTool, ...）
    |       每个工具通过 buildTool() 工厂函数创建
    |
    +-- 检查特性标志（GrowthBook）
    |       if (!feature('COORDINATOR_MODE')) -> 过滤 TeamCreate/TeamDelete
    |       if (!feature('WEB_BROWSER')) -> 过滤 WebBrowserTool
    |
    +-- 加载 MCP 工具
    |       从配置中读取 MCP 服务器列表
    |       连接每个服务器并获取工具列表
    |       创建 MCPTool 代理（shouldDefer=true）
    |
    +-- 返回工具池
            assembleToolPool(coreTools, mcpTools, featureGatedTools)
```

### 2.4.2 Tool 接口完整定义

```typescript
// Tool.ts -- 完整的 Tool 接口定义

export interface ToolUseContext {
  cwd: string;
  homeDir: string;
  abortController: AbortController;
  toolUseId: string;
  sessionId: string;
  fileCache: LRUCache<string, FileContent>;
  costTracker: CostTracker;
  permissionMode: PermissionMode;
  isNonInteractive: boolean;
  model: string;
}

export interface ToolResult {
  content: string;
  isError?: boolean;
  metadata?: {
    fileChanges?: FileChange[];
    exitCode?: number;
    duration?: number;
  };
}

export type PermissionResult =
  | { allowed: true }
  | { allowed: false; reason: string; block?: boolean };

export interface Tool {
  name: string;
  description: string;
  inputJSONSchema: JSONSchema;
  call(input: Record<string, unknown>, context: ToolUseContext): Promise<ToolResult>;
  validateInput?(input: unknown): { valid: boolean; error?: string; };
  checkPermissions?(input: Record<string, unknown>, context: ToolUseContext): PermissionResult;
  isConcurrencySafe(input: Record<string, unknown>): boolean;
  isReadOnly?: boolean;
  isDestructive?: boolean;
  isEnabled?(): boolean;
  canUseInNonInteractive?: boolean;
  shouldDefer?: boolean;
  alwaysLoad?: boolean;
  toAutoClassifierInput?(input: Record<string, unknown>): string;
}
```

### 2.4.3 `buildTool()` 工厂函数和默认值

```typescript
const TOOL_DEFAULTS: Partial<Tool> = {
  isReadOnly: false,
  isDestructive: false,
  canUseInNonInteractive: true,
  shouldDefer: false,
  alwaysLoad: false,
  isEnabled: () => true,
};

function buildTool(
  definition: Partial<Tool> & Pick<Tool, 'name' | 'call'>
): Tool {
  return {
    ...TOOL_DEFAULTS,
    ...definition,
    description: definition.description || '',
    inputJSONSchema: definition.inputJSONSchema || { type: 'object', properties: {} },
    isConcurrencySafe: definition.isConcurrencySafe || (() => false),
  };
}

// FileReadTool -- 只读、并发安全
const FileReadTool = buildTool({
  name: 'Read',
  description: 'Reads a file from the local filesystem...',
  inputJSONSchema: fileReadSchema,
  call: async (input, context) => { /* ... */ },
  isReadOnly: true,
  isConcurrencySafe: () => true,
});

// BashTool -- 非只读、非并发安全、破坏性
const BashTool = buildTool({
  name: 'Bash',
  description: 'Executes a bash command...',
  inputJSONSchema: bashSchema,
  call: async (input, context) => { /* ... */ },
  isReadOnly: false,
  isDestructive: true,
  isConcurrencySafe: () => false,
  checkPermissions: (input, context) => {
    const command = input.command as string;
    if (isDangerousCommand(command)) {
      return { allowed: false, reason: 'Dangerous command detected' };
    }
    return { allowed: true };
  },
});
```

### 2.4.4 `runTools()` 编排实现

```typescript
async function* runTools(
  params: RunToolsParams,
): AsyncGenerator<StreamEvent, ToolResult[], undefined> {
  const { toolUseBlocks, tools, messages, abortController,
          permissionMode, hooks } = params;

  const siblingAbortController = new AbortController();
  const onMainAbort = () => siblingAbortController.abort();
  abortController.signal.addEventListener('abort', onMainAbort);

  try {
    const safeBatch: ToolUseBlock[] = [];
    const unsafeBatch: ToolUseBlock[] = [];

    for (const block of toolUseBlocks) {
      const tool = tools.find(t => t.name === block.name);
      if (!tool) {
        yield { type: 'tool_execution_error', id: block.id,
                name: block.name, error: new Error(`Unknown tool: ${block.name}`) };
        continue;
      }

      const permission = await checkToolPermission(tool, block.input, permissionMode, hooks);
      if (!permission.allowed) {
        yield { type: 'tool_execution_error', id: block.id,
                name: block.name, error: new Error(permission.reason) };
        if (permission.block) break;
        continue;
      }

      const hookResult = await hooks.run('PreToolUse', {
        toolName: block.name, toolInput: block.input,
      });
      if (hookResult?.blocked) {
        yield { type: 'tool_execution_error', id: block.id,
                name: block.name, error: new Error(hookResult.reason || 'Blocked by hook') };
        continue;
      }

      if (tool.isConcurrencySafe(block.input)) {
        safeBatch.push(block);
      } else {
        unsafeBatch.push(block);
      }
    }

    const results: ToolResult[] = [];

    // 并行执行安全批次
    const safeResults = await Promise.all(
      safeBatch.map(async (block) => {
        const tool = tools.find(t => t.name === block.name)!;
        yield { type: 'tool_execution_start', id: block.id,
                name: block.name, input: block.input };
        try {
          const result = await tool.call(block.input, {
            cwd: params.cwd, abortController: siblingAbortController,
            toolUseId: block.id,
          });
          yield { type: 'tool_execution_complete', id: block.id,
                  name: block.name, result };
          await hooks.run('PostToolUse', {
            toolName: block.name, toolInput: block.input, toolResult: result,
          });
          return { ...result, toolUseId: block.id };
        } catch (error) {
          if (!siblingAbortController.signal.aborted && !(error instanceof AbortError)) {
            siblingAbortController.abort();
          }
          yield { type: 'tool_execution_error', id: block.id,
                  name: block.name, error: error as Error };
          return { content: `Error: ${(error as Error).message}`,
                   isError: true, toolUseId: block.id };
        }
      })
    );
    results.push(...safeResults);

    // 串行执行不安全批次
    for (const block of unsafeBatch) {
      if (siblingAbortController.signal.aborted) break;
      const tool = tools.find(t => t.name === block.name)!;
      yield { type: 'tool_execution_start', id: block.id,
              name: block.name, input: block.input };
      try {
        const result = await tool.call(block.input, {
          cwd: params.cwd, abortController: siblingAbortController,
          toolUseId: block.id,
        });
        yield { type: 'tool_execution_complete', id: block.id,
                name: block.name, result };
        results.push({ ...result, toolUseId: block.id });
      } catch (error) {
        yield { type: 'tool_execution_error', id: block.id,
                name: block.name, error: error as Error };
        results.push({ content: `Error: ${(error as Error).message}`,
                         isError: true, toolUseId: block.id });
        if (!(error instanceof AbortError)) siblingAbortController.abort();
      }
    }

    return results;
  } finally {
    abortController.signal.removeEventListener('abort', onMainAbort);
  }
}
```

### 2.4.5 `isConcurrencySafe` 判断逻辑

```typescript
const CONCURRENCY_SAFE_TOOLS = new Set([
  'Read', 'Glob', 'Grep', 'WebFetch', 'WebSearch',
  'TaskGet', 'TaskList', 'TaskOutput', 'ToolSearch', 'EnterPlanMode',
]);

const CONCURRENCY_UNSAFE_TOOLS = new Set([
  'Bash', 'Write', 'Edit', 'AgentTool', 'SkillTool',
  'TaskCreate', 'TaskUpdate', 'TaskStop', 'SendMessage',
  'TeamCreate', 'TeamDelete', 'ExitPlanMode',
]);

function isConcurrencySafe(toolName: string, input: unknown): boolean {
  if (toolName.startsWith('mcp__')) return false;
  if (CONCURRENCY_SAFE_TOOLS.has(toolName)) return true;
  if (CONCURRENCY_UNSAFE_TOOLS.has(toolName)) return false;
  return false; // 未知工具默认不安全
}
```

### 2.4.6 `siblingAbortController` 错误传播

```
中止信号传播链：

用户按 Ctrl+C
  -> mainAbortController.abort()
    -> query() 检测到中止 -> yield 'aborted' -> return
    -> siblingAbortController.abort() (通过事件监听)
      -> RunningTool1.abortController.abort()
      -> RunningTool2.abortController.abort()
      -> RunningTool3.abortController.abort()

工具 A 执行失败（非 AbortError）
  -> siblingAbortController.abort()
    -> RunningToolB.abortController.abort() -> 工具 B 收到 AbortError
    -> RunningToolC.abortController.abort() -> 工具 C 收到 AbortError

注意：AbortError 不触发兄弟中止（避免级联失败）

中止控制器层级关系：

mainAbortController (进程级)
  -> siblingAbortController (批次级，每次 runTools 创建新的)
     -> toolAbortController_1 (工具级)
     -> toolAbortController_2 (工具级)
     -> toolAbortController_3 (工具级)
```

### 2.4.7 延迟加载（ToolSearchTool）

```
┌─────────────────────────────────────────────────────────────────┐
│                    延迟加载流程                                   │
│                                                                 │
│  系统提示（仅包含工具名称列表，无 schema）                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Available tools:                                        │   │
│  │   Read, Write, Edit, Bash, Glob, Grep, ...             │   │
│  │   mcp__slack__send_message (use ToolSearch for schema)  │   │
│  │   mcp__github__create_issue (use ToolSearch for schema) │   │
│  │   mcp__jira__update_ticket (use ToolSearch for schema)  │   │
│  │   ... (50+ more MCP tools)                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  模型决定使用 mcp__slack__send_message                           │
│       |                                                         │
│       v                                                         │
│  Step 1: 调用 ToolSearch({ query: "select:mcp__slack__send_message"})│
│       |                                                         │
│       v                                                         │
│  Step 2: ToolSearch 返回完整 schema (name, description, input_schema)│
│       |                                                         │
│       v                                                         │
│  Step 3: 模型使用获取的 schema 调用工具                          │
│                                                                 │
│  Token 节省估算：                                                 │
│  * 50 个 MCP 工具，每个 schema ~500 tokens                       │
│  * 全部加载：50 * 500 = 25,000 tokens（每次 API 调用）           │
│  * 延迟加载：仅 ~100 tokens（名称列表）+ 按需 ~500 tokens       │
│  * 节省：~24,400 tokens/轮次（对于未使用的工具）                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.5 上下文与记忆管理（大幅扩充）

### 2.5.1 多层记忆架构

```
┌─────────────────────────────────────────────────────┐
│                 上下文窗口 (默认 200K, 扩展至 1M)     │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │ 系统提示 (每个会话固定)                           │  │
│  │  |- 基础 CLI 指令                               │  │
│  │  |- 工具描述（非延迟加载的）                      │  │
│  │  |- MCP 服务器指令                              │  │
│  │  └- 模式特定指导                                │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │ 用户上下文 (以 system-reminder 注入)             │  │
│  │  |- CLAUDE.md 层级 (项目记忆)                   │  │
│  │  |    /etc/claude-code/CLAUDE.md  (全局)        │  │
│  │  |    ~/.claude/CLAUDE.md         (用户)        │  │
│  │  |    ./CLAUDE.md                 (项目)        │  │
│  │  |    ./.claude/CLAUDE.md         (项目)        │  │
│  │  |    ./.claude/rules/*.md        (项目规则)    │  │
│  │  |    ./CLAUDE.local.md           (本地)        │  │
│  │  |- Git 状态 (分支、最近提交)                     │  │
│  │  └- 当前日期                                    │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │ 对话消息                                        │  │
│  │  |- [压缩边界 -- 旧消息摘要]                      │  │
│  │  |- 用户消息                                    │  │
│  │  |- 助手消息 (文本 + tool_use)                  │  │
│  │  |- 工具结果                                    │  │
│  │  └- ... (随每轮增长)                            │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │ 输出预留 (~20K tokens)                          │  │
│  └────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### 2.5.2 四层压缩策略

```
Token 使用率 ──────────────────────────────────────────▶

0%              80%        85%        90%       98%
|               |          |          |          |
|  正常         | 微压缩    | 自动压缩  | 会话记忆  | 反应式
|  运行         | (清除     | (完整     | 压缩     | 压缩
|               |  旧工具   |  摘要     | (提取    | (API
|               |  结果)    |  旧消息)  |  到记忆) | 错误触发)
```

1. **微压缩（Micro-Compact）**：清除旧工具结果（替换为 `[Old tool result content cleared]`），针对 FileRead、Bash、Grep 等工具，基于时间阈值
2. **自动压缩（Auto-Compact）**：在约 167K tokens 时触发，发送旧消息给模型进行摘要，替换为压缩边界标记
3. **会话记忆压缩（Session Memory Compact）**：提取关键信息到持久会话记忆，保持 10K-40K tokens
4. **反应式压缩（Reactive Compact）**：由 API 的 `prompt_too_long` 错误触发，截断最旧的消息组

### 2.5.3 microCompact 实现细节

```typescript
const MICRO_COMPACT_CONFIG = {
  maxClearCount: 20,
  maxAgeInTurns: 3,
  maxContentSize: 10000,
  persistThreshold: 50000,
};

export function microCompact(messages: Message[]): {
  clearedCount: number;
  persistedFiles: string[];
} {
  let clearedCount = 0;
  const persistedFiles: string[] = [];
  const currentTurnIndex = messages.length;

  for (let i = 0; i < messages.length
       && clearedCount < MICRO_COMPACT_CONFIG.maxClearCount; i++) {
    const msg = messages[i];
    if (msg.role !== 'user' || !Array.isArray(msg.content)) continue;

    const newContent = msg.content.map(block => {
      if (block.type !== 'tool_result') return block;
      const turnAge = currentTurnIndex - i;
      if (turnAge <= MICRO_COMPACT_CONFIG.maxAgeInTurns) return block;

      const contentSize = typeof block.content === 'string'
        ? block.content.length : JSON.stringify(block.content).length;
      if (contentSize <= MICRO_COMPACT_CONFIG.maxContentSize) return block;

      if (contentSize > MICRO_COMPACT_CONFIG.persistThreshold) {
        const filePath = persistToolResult(block.tool_use_id, block.content);
        persistedFiles.push(filePath);
      }

      clearedCount++;
      return { ...block, content: '[Old tool result content cleared]' };
    });

    messages[i] = { ...msg, content: newContent };
  }

  return { clearedCount, persistedFiles };
}
```

### 2.5.4 autoCompact 摘要提示词

```typescript
const COMPACT_SYSTEM_PROMPT = `You are a conversation summarizer. Your task is to
create a concise but comprehensive summary of the conversation so far.

The summary MUST preserve the following information:
1. **User's original request**: What the user asked for, including any specific
   requirements or constraints
2. **Work completed**: What actions were taken, files modified, commands run,
   and their outcomes
3. **Current state**: Where we are in the task -- what's done, what's in
   progress, what's pending
4. **Key decisions**: Any important decisions made during the conversation
   (architecture choices, approaches selected/rejected)
5. **Errors encountered**: Any errors that occurred and how they were resolved
6. **Context for continuation**: Any information needed to continue the task
   without losing progress

Format the summary as a structured document with clear sections.`;

const COMPACT_BOUNDARY_MARKER = `[COMPACT_SUMMARY]
The conversation above this line has been summarized to save context space.
The summary preserves all essential information needed to continue the task.

--- SUMMARY START ---
{summary}
--- SUMMARY END ---`;
```

### 2.5.5 reactiveCompact 截断策略

```typescript
export function reactiveCompact(
  messages: Message[], maxTokens: number,
): Message[] {
  const targetTokens = Math.floor(maxTokens * 0.8);
  let tailTokens = 0;
  let cutIndex = messages.length;

  for (let i = messages.length - 1; i >= 0; i--) {
    const msgTokens = estimateMessageTokens([messages[i]]);
    if (tailTokens + msgTokens > targetTokens) {
      cutIndex = i + 1; break;
    }
    tailTokens += msgTokens;
  }

  if (cutIndex >= messages.length) return messages.slice(-5);

  return [
    { role: 'user',
      content: '[Previous messages were truncated due to context length limits.]' },
    ...messages.slice(cutIndex),
  ];
}
```

### 2.5.6 tokenBudget 计算方式

```typescript
/**
 * 估算消息的 token 数量
 * 公式：字符数 / 3（字符数/4 * 4/3 的简化）
 * 已知精度问题：
 * - 对纯中文文本会低估（中文 1 字符约等于 1-2 tokens）
 * - 对大量代码会高估（代码 token 效率较高）
 * - 误差范围：约 -20% 到 +50%
 */
export function estimateMessageTokens(messages: Message[]): number {
  let totalChars = 0;
  for (const msg of messages) {
    if (typeof msg.content === 'string') {
      totalChars += msg.content.length;
    } else if (Array.isArray(msg.content)) {
      for (const block of msg.content) {
        if (block.type === 'text') totalChars += block.text?.length || 0;
        else if (block.type === 'tool_use') totalChars += JSON.stringify(block.input).length;
        else if (block.type === 'tool_result')
          totalChars += typeof block.content === 'string'
            ? block.content.length : JSON.stringify(block.content).length;
        else if (block.type === 'thinking') totalChars += block.thinking?.length || 0;
      }
    }
  }
  return Math.max(1, Math.ceil(totalChars / 3));
}

export class TokenBudget {
  private maxTokens: number;
  private usedTokens: number = 0;
  constructor(maxTokens: number) { this.maxTokens = maxTokens; }
  getThreshold(type: 'compact'): number {
    return type === 'compact' ? Math.floor(this.maxTokens * 0.80) : this.maxTokens;
  }
  update(usage: Usage): void {
    this.usedTokens += usage.input_tokens + usage.output_tokens;
  }
  getUsageRatio(): number { return this.usedTokens / this.maxTokens; }
  shouldAutoCompact(): boolean { return this.getUsageRatio() >= 0.85; }
}
```

### 2.5.7 持久记忆（memdir）

```
~/.claude/projects/<project-slug>/memory/
├── MEMORY.md           # 索引文件 (最多 200 行)
├── user_role.md        # 用户类型：角色、偏好
├── feedback_testing.md # 反馈类型：要重复/避免的行为
├── project_auth.md     # 项目类型：持续工作上下文
└── reference_docs.md   # 参考类型：外部系统指针
```

### 2.5.8 提示词缓存优化

通过 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 标记将静态指令（全局缓存）与动态会话内容分离，最大化 API 提示词缓存命中率。

```typescript
const systemPrompt = [
  {
    type: 'text',
    text: STATIC_SYSTEM_INSTRUCTIONS,
    cache_control: { type: 'ephemeral' },  // 标记为可缓存
  },
  {
    type: 'text',
    text: '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__',  // 动态边界标记
  },
  {
    type: 'text',
    text: dynamicSessionContent,  // 不标记 cache_control
  },
];
```

**缓存效果**：静态部分（~30K tokens）在首次请求后被 API 缓存，后续请求只需发送动态部分（~5K tokens）+ 缓存引用，节省约 85% 的输入 token 成本和延迟。

### 2.5.9 LRU 文件状态缓存

```typescript
this.fileCache = new LRUCache<string, FileContent>({
  max: 100,           // 最多缓存 100 个文件
  maxSize: 25 * 1024 * 1024,  // 总大小上限 25MB
  sizeCalculation: (value: FileContent) => {
    return typeof value.content === 'string'
      ? value.content.length : Buffer.byteLength(value.content);
  },
});
// 使用场景：FileReadTool 读取时先查缓存，FileEditTool 编辑时更新缓存，
// 跨轮次保持文件状态，避免冗余读取和变更检测
```

### 2.5.10 Dream Task

独特的后台记忆整合机制，在用户空闲时自动回顾会话并更新记忆文件，实现跨会话学习持久化：

```
用户空闲（超过 5 分钟无交互）
  -> autoDream 服务启动（后台子 Agent）
  -> 回顾当前会话历史，提取关键信息
  -> 更新 ~/.claude/projects/<slug>/memory/ 下的记忆文件
  -> 下次会话启动时，Claude 已经"记住"了之前的学习
```

---

## 2.6 权限与安全系统（大幅扩充）

### 权限管道

```
工具调用到达
     |
     v
┌──────────────┐    ┌────────────┐
│ 检查模式     │───>│ bypass     │──> 允许（跳过所有检查）
│              │    │ Permissions │
│              │    └────────────┘
│              │    ┌────────────┐
│              │───>│ dontAsk    │──> 拒绝（阻止所有）
└──────┬───────┘    └────────────┘
       |
       v
┌──────────────┐
│ 应用规则     │
│  1. Deny     │──> 匹配 -> 拒绝
│  2. Allow    │──> 匹配 -> 允许
│  3. Ask      │──> 匹配 -> 提示用户
└──────┬───────┘
       | 无规则匹配
       v
┌──────────────┐
│ Auto 模式?   │──YES──> LLM 分类器
│              │         |- 白名单工具? -> 允许
│              │         |- 分类器说安全? -> 允许
│              │         └- 分类器说不安全? -> 拒绝
│              │             (>3 连续或 >20 总计 -> 回退到 ASK)
└──────┬───────┘
       | 非 auto 模式
       v
┌──────────────┐
│ 模式特定默认 │──> ASK 用户
│  acceptEdits │──> 允许 cwd 内文件编辑，其他 ASK
│  plan        │──> 暂停并显示计划
└──────────────┘
```

### 权限模式

| 模式 | 符号 | 行为 |
|------|------|------|
| `default` | `>` | 所有非只读工具都需要询问 |
| `acceptEdits` | `>>` | 自动允许 cwd 内文件编辑 |
| `plan` | `?` | 在工具调用之间暂停以供审查 |
| `bypassPermissions` | `!` | 跳过所有检查（危险） |
| `auto` | `A` | LLM 分类器决定（特性门控） |

### 2.6.1 权限管道完整代码流程

```typescript
export async function checkToolPermission(
  tool: Tool, input: Record<string, unknown>,
  mode: PermissionMode, rules: PermissionRule[],
  hooks: HookEngine, autoClassifier: AutoClassifier | null,
  stats: PermissionStats,
): Promise<PermissionResult> {
  // 第 1 层：模式前置检查
  if (mode === 'bypassPermissions') {
    if (isBypassDisabledByRemote()) {
      return { allowed: false, reason: 'Bypass mode disabled by remote policy' };
    }
    return { allowed: true };
  }
  if (mode === 'dontAsk') {
    if (tool.isReadOnly) return { allowed: true };
    return { allowed: false, reason: 'dontAsk mode: non-readonly tools blocked' };
  }

  // 第 2 层：只读工具自动放行
  if (tool.isReadOnly) return { allowed: true };

  // 第 3 层：工具级权限检查
  if (tool.checkPermissions) {
    const toolPermission = tool.checkPermissions(input, context);
    if (!toolPermission.allowed) return toolPermission;
  }

  // 第 4 层：规则匹配
  for (const rule of rules) {
    if (matchesRule(tool.name, input, rule)) {
      switch (rule.action) {
        case 'deny':
          stats.deniedCount++; stats.consecutiveDenials++;
          return { allowed: false, reason: rule.reason || 'Denied by rule' };
        case 'allow':
          stats.consecutiveDenials = 0;
          return { allowed: true };
        case 'ask': break;
      }
    }
  }

  // 第 5 层：acceptEdits 模式特殊处理
  if (mode === 'acceptEdits') {
    if ((tool.name === 'Write' || tool.name === 'Edit') && isWithinCwd(input.file_path)) {
      return { allowed: true };
    }
  }

  // 第 6 层：Auto 模式 -- LLM 分类器
  if (mode === 'auto' && autoClassifier) {
    if (isAutoModeDisabledByRemote()) return await askUser(tool, input);
    if (AUTO_WHITELIST_TOOLS.has(tool.name)) return { allowed: true };
    if (stats.consecutiveDenials >= 3 || stats.deniedCount >= 20) {
      return await askUser(tool, input);
    }
    const classifierInput = tool.toAutoClassifierInput?.(input)
      || `${tool.name}: ${JSON.stringify(input)}`;
    const isSafe = await autoClassifier.classify(classifierInput);
    if (isSafe) { stats.consecutiveDenials = 0; return { allowed: true }; }
    else { stats.deniedCount++; stats.consecutiveDenials++;
            return { allowed: false, reason: 'Auto-classifier: unsafe operation' }; }
  }

  // 第 7 层：用户确认
  return await askUser(tool, input);
}
```

### 2.6.2 安全机制 7 层详细说明

| 层级 | 机制 | 实现位置 | 说明 |
|------|------|----------|------|
| **1** | 危险文件保护 | `utils/permissions/dangerousFiles.ts` | `.gitconfig`、`.bashrc`、`.zshrc`、`.mcp.json`、`/etc/` 下的系统文件被阻止修改 |
| **2** | 危险命令检测 | `BashTool.checkPermissions()` | `rm -rf /`、`git push --force`、`DROP TABLE`、`chmod 777`、`> /dev/sda` 等模式匹配 |
| **3** | Bypass 权限终止开关 | `growthbook.ts` + `isBypassDisabledByRemote()` | GrowthBook 特性门可以远程禁用 bypass 模式 |
| **4** | Auto 模式断路器 | Statsig 实时门 | Statsig 门可以远程禁用 auto 模式，连续拒绝 >3 或总计 >20 时自动回退到 ASK |
| **5** | 拒绝追踪 | `PermissionStats` | 跟踪 `deniedCount` 和 `consecutiveDenials`，超过阈值时切换到更保守的模式 |
| **6** | 技能范围收窄 | `SkillTool` | 编辑 `.claude/skills/X/` 时提供窄范围权限 |
| **7** | MCP Shell 阻止 | `MCPTool` | MCP 来源的技能永不执行 shell 命令，防止远程代码注入 |

### 2.6.3 LLM 自动分类器（auto 模式）

```typescript
const AUTO_WHITELIST_TOOLS = new Set([
  'Read', 'Glob', 'Grep', 'WebFetch', 'WebSearch',
  'TaskGet', 'TaskList', 'TaskOutput', 'ToolSearch',
  'EnterPlanMode', 'ExitPlanMode',
]);

export class AutoClassifier {
  private consecutiveDenials: number = 0;
  private totalDenials: number = 0;

  async classify(input: string): Promise<boolean> {
    if (this.consecutiveDenials >= 3 || this.totalDenials >= 20) return false;
    if (isAutoModeDisabledByRemote()) return false;

    try {
      const response = await anthropic.messages.create({
        model: 'claude-3-haiku-20240307',
        max_tokens: 1,
        messages: [{
          role: 'user',
          content: `Is this tool call safe to execute without user confirmation?
Tool call: ${input}
Respond with only "Y" (safe) or "N" (unsafe).`,
        }],
      });

      const isSafe = response.content[0].text.trim() === 'Y';
      if (!isSafe) { this.consecutiveDenials++; this.totalDenials++; }
      else { this.consecutiveDenials = 0; }
      return isSafe;
    } catch { return false; }
  }
}
```

---

## 2.7 多 Agent 编排

Claude Code 支持**三个级别的多 Agent 执行**：

```
┌────────────────────────────────────────────────────────────────┐
│  级别 1: 子 Agent (AgentTool)                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 主 Agent 通过 AgentTool 生成子 Agent                      │  │
│  │  * 隔离的文件缓存（从父 Agent 克隆）                      │  │
│  │  * 独立的 AbortController                                 │  │
│  │  * 独立的转录记录 (JSONL 侧链)                            │  │
│  │  * 过滤的工具池（按 Agent 定义）                          │  │
│  │  * 以文本形式返回结果给父 Agent                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  级别 2: 协调器模式 (多 Worker)                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ CLAUDE_CODE_COORDINATOR_MODE=1                           │  │
│  │  * 系统提示重写为编排模式                                  │  │
│  │  * 通过 AgentTool 生成受限工具的 Worker                   │  │
│  │  * XML task-notification 协议传递结果                    │  │
│  │  * 协调器聚合并响应用户                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  级别 3: 团队模式 (持久化团队)                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ TeamCreateTool 创建命名团队                               │  │
│  │  * 团队文件持久化到 ~/.claude/teams/{name}.json           │  │
│  │  * InProcessTeammates 在同一进程中运行                    │  │
│  │  * SendMessageTool 在队友间路由消息                       │  │
│  │  * 共享 scratchpad 文件系统进行知识交换                    │  │
│  │  * 结构化关闭协议 (request -> approve)                     │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

**Fork 子 Agent（提示缓存优化）**：当从同一上下文生成多个 Agent 时，使用 Fork 机制最大化 API 提示缓存命中。所有子 Agent 共享相同的前缀（父对话历史），仅最后一条指令不同。

---

## 2.8 Hook 系统

Hooks 是用户定义的生命周期事件动作，是 Claude Code 的**可扩展性骨干**。

### Hook 事件类型

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

### Hook 类型

| 类型 | 实现 | 特点 |
|------|------|------|
| **Command Hook** | Shell 命令 (bash/zsh) | Exit code 0=ok, 2=block, N=error |
| **Prompt Hook** | LLM 评估条件 (Haiku 模型) | 返回 `{ok: true/false, reason}` |
| **Agent Hook** | 完整 Agent + 工具 | 超时 60s，无递归 |
| **HTTP Hook** | HTTP 请求到端点 | JSON body + context |
| **Function Hook** | TS 回调（内存中） | 仅会话级，不持久化 |

---

## 2.9 技能与插件系统

### 技能来源

| 来源 | 位置 | 优先级 |
|------|------|--------|
| 托管技能 | 企业策略控制 | 最高 |
| 项目技能 | `./.claude/skills/` | 高 |
| 用户技能 | `~/.claude/skills/` | 中 |
| 内置技能 | 编译打包 | 最低 |

### 技能执行模式

- **内联模式（默认）**：技能内容注入到当前对话中，模型视为当前轮次的一部分
- **Fork 模式（`context: "fork"`）**：创建子 Agent 独立执行，拥有自己的上下文和预算，返回结果文本给父 Agent

### 条件技能（路径过滤）

技能可以配置 `paths` 字段，仅当模型编辑匹配的文件时才激活。

---

## 2.10 MCP 集成

### MCP 客户端架构

支持 4 种传输协议：
- **stdio**：本地进程 stdin/stdout 管道
- **SSE/HTTP**：远程 HTTP + EventSource + OAuth
- **WebSocket**：持久连接 + 二进制帧 + TLS/代理
- **local**：本地协议

工具名称规范化：服务器 `my-server` 的工具 `send_message` -> `mcp__my_server__send_message`

### MCP 配置作用域

| 作用域 | 位置 | 用例 |
|--------|------|------|
| `local` | `.claude/settings.local.json` | 用户本地服务器 |
| `user` | `~/.claude/settings.json` | 用户全局服务器 |
| `project` | `.claude/settings.json` | 团队共享服务器 |
| `dynamic` | 运行时注册 | 编程式服务器 |
| `enterprise` | MDM 策略 | 管理员管理服务器 |

---

## 2.11 特性标志与构建系统

Claude Code 运行在 Bun（非 Node.js）上，关键影响：
- 原生 JSX/TSX 支持无需转译
- `bun:bundle` 特性标志用于死代码消除

```typescript
import { feature } from 'bun:bundle'
if (feature('VOICE_MODE')) {
  // 此代码在构建时被完全剥离（如果标志未激活）
}
```

| 标志 | 功能 |
|------|------|
| `PROACTIVE` | 主动 Agent 模式 |
| `KAIROS` | Kairos 子系统 |
| `BRIDGE_MODE` | IDE bridge 集成 |
| `DAEMON` | 后台守护进程模式 |
| `VOICE_MODE` | 语音输入/输出 |
| `AGENT_TRIGGERS` | 触发式 Agent 动作 |
| `MONITOR_TOOL` | 监控工具 |
| `COORDINATOR_MODE` | 多 Agent 协调器 |
| `WORKFLOW_SCRIPTS` | 工作流自动化脚本 |

---

## 2.12 配置系统（新增章节）

### 2.12.1 CLAUDE.md 加载优先级和合并策略

CLAUDE.md 文件是 Claude Code 的项目级指令系统，支持 4 级目录层级和惰性加载机制：

```
┌─────────────────────────────────────────────────────────────────┐
│                 CLAUDE.md 4 级目录层级                           │
│                                                                 │
│  优先级    位置                              作用域              │
│  ──────    ────                              ────              │
│  最高      /etc/claude-code/CLAUDE.md       系统级（管理员）     │
│  高        ~/.claude/CLAUDE.md              用户级（全局）       │
│  中        ./.claude/CLAUDE.md              项目级（团队共享）   │
│  中        ./.claude/rules/*.md             项目规则（按文件匹配）│
│  低        ./CLAUDE.md                      项目级（本地）       │
│  最低      ./CLAUDE.local.md                本地级（不提交 Git） │
│                                                                 │
│  惰性加载机制：                                                 │
│  1. 启动时仅加载系统级和用户级 CLAUDE.md                        │
│  2. 项目级 CLAUDE.md 在进入项目目录时加载                      │
│  3. rules/*.md 按文件路径匹配条件加载                          │
│  4. CLAUDE.local.md 仅在本地存在时加载                          │
│  5. 文件变更通过 settingsChangeDetector + debounce 触发重新加载 │
└─────────────────────────────────────────────────────────────────┘
```

```typescript
// CLAUDE.md 加载逻辑（简化示意）
async function loadClaudeMdFiles(cwd: string): Promise<ClaudeMdContent[]> {
  const files: ClaudeMdContent[] = [];

  // 1. 系统级
  const systemPath = '/etc/claude-code/CLAUDE.md';
  if (await exists(systemPath)) {
    files.push({ path: systemPath, content: await readFile(systemPath), priority: 0 });
  }

  // 2. 用户级
  const userPath = path.join(os.homedir(), '.claude', 'CLAUDE.md');
  if (await exists(userPath)) {
    files.push({ path: userPath, content: await readFile(userPath), priority: 1 });
  }

  // 3. 项目级
  const projectPaths = [
    path.join(cwd, '.claude', 'CLAUDE.md'),
    path.join(cwd, 'CLAUDE.md'),
  ];
  for (const p of projectPaths) {
    if (await exists(p)) {
      files.push({ path: p, content: await readFile(p), priority: 2 });
    }
  }

  // 4. 项目规则（按路径匹配）
  const rulesDir = path.join(cwd, '.claude', 'rules');
  if (await exists(rulesDir)) {
    const ruleFiles = await readdir(rulesDir);
    for (const ruleFile of ruleFiles.filter(f => f.endsWith('.md'))) {
      files.push({
        path: path.join(rulesDir, ruleFile),
        content: await readFile(path.join(rulesDir, ruleFile)),
        priority: 2,
        isRule: true,
      });
    }
  }

  // 5. 本地级
  const localPath = path.join(cwd, 'CLAUDE.local.md');
  if (await exists(localPath)) {
    files.push({ path: localPath, content: await readFile(localPath), priority: 3 });
  }

  return files.sort((a, b) => a.priority - b.priority);
}
```

### 2.12.2 settings.json 5 级优先级

```
┌─────────────────────────────────────────────────────────────────┐
│                 settings.json 5 级优先级                         │
│                                                                 │
│  优先级（从低到高）                                             │
│                                                                 │
│  1. flagSettings      特性标志默认值                            │
│     来源：代码内硬编码的默认值                                  │
│                                                                 │
│  2. localSettings     本地设置                                  │
│     来源：.claude/settings.local.json                           │
│     用途：开发者个人偏好，不提交到 Git                          │
│                                                                 │
│  3. projectSettings   项目设置                                  │
│     来源：.claude/settings.json                                 │
│     用途：团队共享的项目配置，提交到 Git                        │
│                                                                 │
│  4. userSettings      用户设置                                  │
│     来源：~/.claude/settings.json                               │
│     用途：用户全局偏好，跨项目生效                              │
│                                                                 │
│  5. policySettings    策略设置（最高优先级）                     │
│     来源：MDM 策略文件                                          │
│     用途：企业管理员强制策略，不可被用户覆盖                    │
│                                                                 │
│  合并策略：mergeWith + settingsMergeCustomizer                  │
│  - 数组字段：高优先级替换低优先级（非追加）                     │
│  - 对象字段：深度合并                                          │
│  - 布尔字段：高优先级覆盖低优先级                               │
│                                                                 │
│  热重载：settingsChangeDetector + debounce                      │
│  - 文件监视器检测 settings.json 变更                            │
│  - debounce 300ms 避免频繁重载                                  │
│  - 重载后触发 ConfigChange Hook                                │
└─────────────────────────────────────────────────────────────────┘
```

```typescript
// settings.ts -- 合并策略
import { mergeWith } from 'lodash';
import { settingsMergeCustomizer } from './settingsMerge';

function settingsMergeCustomizer(objValue: any, srcValue: any, key: string) {
  // 数组字段：替换而非追加
  if (Array.isArray(objValue) && Array.isArray(srcValue)) {
    return srcValue;
  }
  // 其他情况：使用 lodash 默认的深度合并
}

export async function loadSettings(): Promise<Settings> {
  const [flagSettings, localSettings, projectSettings,
         userSettings, policySettings] = await Promise.all([
    loadFeatureFlagSettings(),
    loadLocalSettings(),
    loadProjectSettings(),
    loadUserSettings(),
    loadPolicySettings(),
  ]);

  return mergeWith(
    {}, flagSettings, localSettings, projectSettings,
    userSettings, policySettings, settingsMergeCustomizer,
  );
}
```

### 2.12.3 GrowthBook 特性标志系统

```
┌─────────────────────────────────────────────────────────────────┐
│                 GrowthBook 特性标志系统                          │
│                                                                 │
│  初始化属性：                                                   │
│  {                                                              │
│    organizationUUID: string,    // 组织 ID                     │
│    accountUUID: string,         // 账户 ID                     │
│    email: string,               // 用户邮箱                    │
│    platform: string,            // 操作系统 (darwin/linux/win32)│
│    claudeCodeVersion: string,   // Claude Code 版本号          │
│  }                                                              │
│                                                                 │
│  编译时 DCE vs 运行时远程配置轮询：                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 编译时 (bun:bundle feature())                           │   │
│  │  - 构建时完全剥离未激活的代码分支                        │   │
│  │  - 减小打包体积（如 VOICE_MODE ~200KB）                 │   │
│  │  - 适用于永久性功能开关                                  │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │ 运行时 (GrowthBook.isOn())                              │   │
│  │  - 从 CDN 轮询最新配置                                  │   │
│  │  - 支持实时开关（无需重新构建）                          │   │
│  │  - 支持 A/B 测试和灰度发布                               │   │
│  │  - 适用于需要灵活控制的开关                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Feature flag 随机词对命名（防止猜测）：                        │
│  - tengu_frond_boric        -> 禁用 bypass 权限                │
│  - alpine_ripple_flux       -> 控制 auto 模式可用性            │
│  - cobalt_glade_sprout      -> 会话镜像功能开关                │
│  - ember_pine_quartz        -> 协调器模式开关                  │
│                                                                 │
│  关键远程控制能力：                                             │
│  1. 禁用 bypass 权限（安全紧急响应）                           │
│  2. 控制 auto 模式可用性（安全策略调整）                       │
│  3. 会话镜像（企业审计）                                       │
│  4. 功能灰度发布（按用户百分比）                               │
│  5. A/B 测试（不同配置的性能对比）                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.13 状态管理（新增章节）

### 2.13.1 Bootstrap State 完整结构

```typescript
export interface BootstrapState {
  // ─── 目录与会话 ─────────────────────────────────────────
  cwd: string;                    // 当前工作目录
  homeDir: string;                // 用户主目录
  sessionId: string;              // 会话唯一 ID
  conversationId: string;         // 对话 ID

  // ─── 成本追踪 ───────────────────────────────────────────
  totalCost: number;              // 累计成本（美元）
  totalInputTokens: number;       // 累计输入 token 数
  totalOutputTokens: number;      // 累计输出 token 数
  totalCacheReadTokens: number;   // 累计缓存读取 token 数
  totalCacheCreationTokens: number; // 累计缓存创建 token 数

  // ─── 性能指标 ───────────────────────────────────────────
  startupTime: number;            // 启动耗时（ms）
  apiLatencyP50: number;          // API 延迟 P50（ms）
  apiLatencyP99: number;          // API 延迟 P99（ms）
  toolExecutionCount: number;     // 工具执行总次数
  toolExecutionErrors: number;    // 工具执行错误次数

  // ─── 认证安全 ───────────────────────────────────────────
  authToken: string | null;       // OAuth token
  authMethod: 'oauth' | 'api_key' | 'bare' | null;
  mdmPolicy: MDMPolicy | null;    // MDM 策略

  // ─── 遥测 ───────────────────────────────────────────────
  otelExporter: OTelExporter | null;
  statsigClient: StatsigClient | null;
  growthBook: GrowthBook | null;

  // ─── Hooks ───────────────────────────────────────────────
  hooks: Map<HookEvent, Hook[]>;

  // ─── 特性标志 ───────────────────────────────────────────
  featureFlags: Map<string, boolean>;

  // ─── 设置 ───────────────────────────────────────────────
  settings: Settings;
}
```

### 2.13.2 QueryEngine 状态管理

```typescript
interface QueryEngineState {
  // 预算执行
  tokenBudget: TokenBudget;
  estimatedTokens: number;

  // 重试状态
  retryCount: number;
  lastRetryError: Error | null;
  nextRetryAt: number;

  // 权限状态
  permissionMode: PermissionMode;
  deniedCount: number;
  consecutiveDenials: number;
  lastDeniedTool: string | null;

  // 会话统计
  messageCount: number;
  toolCallCount: number;
  compactCount: number;
}
```

### 2.13.3 React Context 使用方式

Claude Code 使用自定义的极简 Store 实现替代 Zustand，通过 React Context 注入：

```typescript
// store.ts -- createStore (~20行)
type Listener<T> = (state: T) => void;

export function createStore<T extends object>(initialState: T) {
  let state = initialState;
  const listeners = new Set<Listener<T>>();

  return {
    getState: () => state,
    setState: (partial: Partial<T>) => {
      state = { ...state, ...partial };
      listeners.forEach(listener => listener(state));
    },
    subscribe: (listener: Listener<T>) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
  };
}

// AppStateProvider -- 防嵌套机制
export function AppStateProvider({ children }: { children: React.ReactNode }) {
  const storeRef = useRef<AppStore | null>(null);

  if (!storeRef.current) {
    storeRef.current = createStore<AppState>(defaultAppState);
  }

  // 防嵌套：如果已经存在 Provider，直接复用
  const existingStore = useContext(AppStoreContext);
  if (existingStore) {
    return <>{children}</>;
  }

  return (
    <AppStoreContext.Provider value={storeRef.current}>
      {children}
    </AppStoreContext.Provider>
  );
}
```

**核心 Context 列表**：

| Context | 用途 | 生命周期 |
|---------|------|----------|
| `AppStoreContext` | 全局应用状态（消息、权限、成本） | 会话级 |
| `NotificationsContext` | 通知队列和显示 | 会话级 |
| `StatsContext` | 会话统计信息 | 会话级 |
| `ModalContext` | 模态框状态管理 | 会话级 |
| `OverlayContext` | 覆盖层（diff viewer 等） | 会话级 |
| `VoiceContext` | 语音输入/输出状态 | 会话级 |
| `MailboxContext` | 多 Agent 消息路由 | 会话级 |

---

## 2.14 错误处理与重试（新增章节）

### 2.14.1 withRetry 重试策略

```typescript
// services/api/withRetry.ts -- 指数退避 + 抖动

export interface RetryOptions {
  maxRetries?: number;        // 默认 3
  baseDelay?: number;         // 默认 1000ms
  maxDelay?: number;          // 默认 30000ms
  jitter?: number;            // 默认 1000ms
  retryableStatuses?: number[]; // 默认 [429, 529, 500, 502, 503]
}

export async function withRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions = {},
): Promise<T> {
  const {
    maxRetries = 3,
    baseDelay = 1000,
    maxDelay = 30000,
    jitter = 1000,
    retryableStatuses = [429, 529, 500, 502, 503],
  } = options;

  let lastError: Error;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      const apiError = error as ApiError;
      const status = apiError?.status;

      // 不可重试的错误
      if (status && !retryableStatuses.includes(status)) {
        throw error;
      }

      // 最后一次尝试，不再重试
      if (attempt >= maxRetries) {
        throw error;
      }

      // 计算延迟：指数退避 + 随机抖动
      const exponentialDelay = baseDelay * Math.pow(2, attempt);
      const jitterDelay = Math.random() * jitter;
      const delay = Math.min(exponentialDelay + jitterDelay, maxDelay);

      // 尊重 retry-after 头（如果有）
      const retryAfter = apiError?.headers?.['retry-after'];
      const actualDelay = retryAfter
        ? Math.max(parseInt(retryAfter) * 1000, delay)
        : delay;

      await sleep(actualDelay);
    }
  }

  throw lastError!;
}
```

### 2.14.2 错误分类（classifyError）

```typescript
// services/api/errors.ts -- 错误分类

export type ErrorCategory =
  | 'rate_limit'      // 429 - 速率限制
  | 'overloaded'      // 529 - 服务过载
  | 'server_error'    // 5xx - 服务器错误
  | 'auth_error'      // 401 - 认证失败
  | 'forbidden'       // 403 - 禁止访问
  | 'context_overflow' // prompt_too_long
  | 'network_error'   // 网络连接错误
  | 'timeout'         // 请求超时
  | 'unknown';        // 未知错误

export function classifyError(error: unknown): ErrorCategory {
  // 非 API 错误
  if (!(error instanceof ApiError)) {
    if (error instanceof TypeError && error.message.includes('fetch')) {
      return 'network_error';
    }
    if (error instanceof DOMException && error.name === 'AbortError') {
      return 'timeout';
    }
    return 'network_error';
  }

  const { status, message } = error;

  if (status === 429) return 'rate_limit';
  if (status === 529) return 'overloaded';
  if (status >= 500 && status < 600) return 'server_error';
  if (status === 401) return 'auth_error';
  if (status === 403) return 'forbidden';
  if (message?.includes('prompt_too_long')) return 'context_overflow';
  if (message?.includes('timeout')) return 'timeout';

  return 'unknown';
}
```

### 2.14.3 速率限制处理

```typescript
// 速率限制处理策略

// 1. 尊重 retry-after 头
function getRetryAfterMs(error: ApiError): number | null {
  const retryAfter = error.headers?.['retry-after'];
  if (!retryAfter) return null;

  // retry-after 可以是秒数或 HTTP 日期
  const seconds = parseInt(retryAfter, 10);
  if (!isNaN(seconds)) return seconds * 1000;

  const date = new Date(retryAfter);
  if (!isNaN(date.getTime())) {
    return Math.max(0, date.getTime() - Date.now());
  }

  return null;
}

// 2. 组织级限制检测
function isOrgRateLimit(error: ApiError): boolean {
  return error.status === 429
    && error.message?.includes('rate limit')
    && error.message?.includes('organization');
}

// 组织级限制不重试，直接通知用户升级计划
if (isOrgRateLimit(error)) {
  throw new OrgRateLimitError(
    'Organization rate limit reached. Please upgrade your plan or wait.'
  );
}
```

### 2.14.4 工具执行失败处理

```typescript
// 工具执行失败的差异化处理

try {
  const result = await tool.call(input, context);
  // ...
} catch (error) {
  if (error instanceof AbortError) {
    // AbortError：被 siblingAbortController 或用户中止
    // 不触发兄弟中止，静默处理
    return { content: '[Tool execution aborted]', isError: true };
  }

  if (error instanceof PermissionDeniedError) {
    // 权限拒绝：记录但不中止其他工具
    return { content: `Permission denied: ${error.message}`, isError: true };
  }

  if (error instanceof ValidationError) {
    // 输入验证失败：记录错误但不中止
    return { content: `Validation error: ${error.message}`, isError: true };
  }

  // 其他错误：触发兄弟中止
  siblingAbortController.abort();
  return { content: `Error: ${error.message}`, isError: true };
}
```

---

## 2.15 会话恢复机制（新增章节）

### 2.15.1 会话持久化到磁盘

```
┌─────────────────────────────────────────────────────────────────┐
│                 会话持久化文件结构                                │
│                                                                 │
│  ~/.claude/projects/<project-slug>/                            │
│  ├── sessions.json              # 会话索引                      │
│  │   {                                                          │
│  │     "sessions": [                                          │
│  │       {                                                      │
│  │         "id": "abc123",                                     │
│  │         "title": "Implement auth feature",                  │
│  │         "createdAt": "2026-04-13T10:00:00Z",                │
│  │         "lastUpdated": "2026-04-13T11:30:00Z",              │
│  │         "messageCount": 42,                                 │
│  │         "totalTokens": 156000,                              │
│  │         "totalCost": 0.42,                                  │
│  │         "transcriptPath": "sessions/abc123.jsonl"           │
│  │       },                                                     │
│  │       ...                                                    │
│  │     ]                                                        │
│  │   }                                                          │
│  │                                                              │
│  └── sessions/                                                │
│      └── abc123.jsonl            # JSONL 转录文件                │
│          {"type":"message","ts":...,"msg":{...}}               │
│          {"type":"tool_call","ts":...,"tool":{...}}            │
│          {"type":"checkpoint","ts":...,"state":{...}}          │
│          ...                                                    │
└─────────────────────────────────────────────────────────────────┘
```

```typescript
// SessionMetadata 接口定义
export interface SessionMetadata {
  id: string;
  title: string;
  createdAt: string;          // ISO 8601
  lastUpdated: string;        // ISO 8601
  messageCount: number;
  totalTokens: number;
  totalCost: number;
  transcriptPath: string;
  model: string;
  permissionMode: PermissionMode;
}

// JSONL 转录事件类型
type TranscriptEvent =
  | { type: 'message'; timestamp: number; message: Message; }
  | { type: 'tool_call'; timestamp: number; toolName: string;
      input: unknown; result: ToolResult; }
  | { type: 'checkpoint'; timestamp: number; state: CheckpointState; }
  | { type: 'compact'; timestamp: number; clearedCount: number; }
  | { type: 'permission_decision'; timestamp: number;
      toolName: string; allowed: boolean; reason?: string; };
```

### 2.15.2 持久化时机

```
┌─────────────────────────────────────────────────────────────────┐
│                 持久化时机                                       │
│                                                                 │
│  1. 每次消息交换完成后                                         │
│     - 用户消息发送后                                           │
│     - 助手响应完成后                                           │
│     追加到 JSONL 转录文件                                     │
│                                                                 │
│  2. 每次工具调用完成后                                         │
│     - 工具名称、输入、结果                                     │
│     - 执行时长、退出码                                         │
│     追加到 JSONL 转录文件                                     │
│                                                                 │
│  3. Checkpoint 时刻                                             │
│     - 自动压缩后                                               │
│     - 会话空闲超过阈值                                         │
│     - 创建完整的消息快照                                       │
│     追加 checkpoint 事件到 JSONL                               │
│                                                                 │
│  4. sessions.json 更新时机                                     │
│     - 每次消息交换后更新 lastUpdated 和统计                    │
│     - 使用 debounce 5s 避免频繁写入                            │
└─────────────────────────────────────────────────────────────────┘
```

### 2.15.3 恢复会话时的状态重建

```typescript
// 4 步恢复过程

export async function restoreSession(
  sessionId: string,
): Promise<SessionRestoreResult> {
  // 步骤 1: 读取会话元数据
  const metadata = await readSessionMetadata(sessionId);
  if (!metadata) {
    throw new Error(`Session ${sessionId} not found`);
  }

  // 步骤 2: 读取并解析 JSONL 转录
  const transcriptPath = path.join(
    getProjectDir(), 'sessions', `${sessionId}.jsonl`
  );
  const transcriptLines = await readFile(transcriptPath, 'utf-8');
  const events = transcriptLines
    .split('\n')
    .filter(line => line.trim())
    .map(line => JSON.parse(line));

  // 步骤 3: 重建消息历史
  const messages: Message[] = [];
  let lastCheckpoint: CheckpointState | null = null;

  for (const event of events) {
    switch (event.type) {
      case 'message':
        messages.push(event.message);
        break;
      case 'checkpoint':
        lastCheckpoint = event.state;
        break;
      // 其他事件类型用于审计，不参与消息重建
    }
  }

  // 如果有 checkpoint，从 checkpoint 恢复（跳过中间事件）
  if (lastCheckpoint) {
    messages.length = 0;
    messages.push(...lastCheckpoint.messages);
  }

  // 步骤 4: 恢复工具注册表和权限状态
  const tools = await rebuildToolRegistry(metadata.model);
  const permissionState = {
    mode: metadata.permissionMode || 'default',
    deniedCount: 0,
    consecutiveDenials: 0,
  };

  return {
    messages,
    tools,
    permissionState,
    metadata,
    stats: {
      totalTokens: metadata.totalTokens,
      totalCost: metadata.totalCost,
      messageCount: metadata.messageCount,
    },
  };
}
```

**已知脆弱性**：
- **attachment 不持久化**：用户上传的文件附件（如图片）不会被持久化到磁盘，恢复会话时这些内容会丢失
- **fork-session 缓存前缀问题**：从 fork 的子 Agent 恢复会话时，API 提示词缓存前缀可能与原始会话不同，导致缓存未命中

---

## 2.16 遥测与可观测性（新增章节）

### 2.16.1 三层遥测架构

```
┌─────────────────────────────────────────────────────────────────┐
│                 三层遥测架构                                     │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 第 1 层：Statsig（运营指标）                              │   │
│  │  - 用户行为分析                                          │   │
│  │  - 功能使用统计                                          │   │
│  │  - A/B 实验数据                                          │   │
│  │  - 实时仪表盘                                            │   │
│  │  数据流向：Claude Code -> Statsig CDN                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 第 2 层：Sentry（错误日志）                               │   │
│  │  - 未捕获异常                                            │   │
│  │  - API 错误                                              │   │
│  │  - 工具执行失败                                          │   │
│  │  - 崩溃报告                                              │   │
│  │  数据流向：Claude Code -> Sentry CDN                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 第 3 层：OpenTelemetry（管理员监控）                      │   │
│  │  - 管理员可配置的导出器                                  │   │
│  │  - 自定义 Span 和 Metric                                 │   │
│  │  - 与企业可观测性平台集成                                │   │
│  │  数据流向：Claude Code -> OTEL Collector                 │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.16.2 8 种核心指标

| 指标名称 | 类型 | 说明 |
|----------|------|------|
| `claude_code.session.count` | Counter | 会话总数 |
| `lines_of_code.count` | Counter | 生成/修改的代码行数 |
| `pull_request.count` | Counter | 创建的 PR 数量 |
| `commit.count` | Counter | 创建的 commit 数量 |
| `cost.usage` | Gauge | 当前会话成本（美元） |
| `token.usage` | Gauge | 当前会话 token 使用量 |
| `code_edit_tool.decision` | Histogram | 代码编辑工具的决策时间 |
| `active_time.total` | Gauge | 用户活跃时间（秒） |

### 2.16.3 5 种核心事件

| 事件名称 | 触发时机 | 包含数据 |
|----------|----------|----------|
| `user_prompt` | 用户提交消息 | 消息长度、模式类型 |
| `tool_result` | 工具执行完成 | 工具名称、执行时长、是否成功 |
| `api_request` | API 调用完成 | 模型、token 用量、延迟、缓存命中率 |
| `api_error` | API 调用失败 | 错误类型、状态码、重试次数 |
| `tool_decision` | 权限决策完成 | 工具名称、决策结果（允许/拒绝/询问） |

### 2.16.4 隐私保护措施

```
┌─────────────────────────────────────────────────────────────────┐
│                 隐私保护措施                                     │
│                                                                 │
│  1. Prompt 默认脱敏                                            │
│     - 用户 prompt 内容在发送到遥测前进行哈希处理                │
│     - 仅记录 prompt 的 token 数量，不记录内容                   │
│     - 工具输入中的敏感信息（路径、命令）进行模式替换             │
│                                                                 │
│  2. OTEL_LOG_USER_PROMPTS 环境变量                             │
│     - 默认 false：不记录用户 prompt 内容                        │
│     - 设为 true：记录完整的 prompt（仅用于调试）                │
│     - 仅在用户明确启用时生效                                    │
│                                                                 │
│  3. Bedrock/Vertex 禁用非必要流量                              │
│     - 使用 AWS Bedrock 或 GCP Vertex 时                        │
│     - 自动禁用 Statsig 和 GrowthBook 遥测                      │
│     - 仅保留 OpenTelemetry（由企业管理员控制）                  │
│     - 确保数据不离开企业基础设施                                │
│                                                                 │
│  4. 数据最小化原则                                            │
│     - 仅收集必要的指标和事件                                    │
│     - 不记录对话内容（除非用户明确启用）                        │
│     - 错误日志中自动脱敏文件路径和命令内容                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.17 认证与授权（新增章节）

### 2.17.1 OAuth 2.0 PKCE 流程

Claude Code 使用 OAuth 2.0 PKCE（Proof Key for Code Exchange）进行认证，而非传统的 OAuth 授权码流程。PKCE 的选择理由：
- **安全性**：即使授权码被截获，攻击者也无法交换 token（没有 code_verifier）
- **无需 client_secret**：CLI 应用无法安全存储 client_secret，PKCE 消除了这个需求
- **公共客户端友好**：原生应用和 CLI 工具的最佳实践

```
┌─────────────────────────────────────────────────────────────────┐
│                 OAuth 2.0 PKCE 8 步流程                         │
│                                                                 │
│  Step 1: 生成 PKCE 参数                                        │
│    code_verifier = randomURLSafeString(128)                    │
│    code_challenge = SHA256(code_verifier) -> base64url          │
│                                                                 │
│  Step 2: 打开浏览器授权页面                                    │
│    GET https://console.anthropic.com/oauth/authorize            │
│      ?client_id=claude-code                                    │
│      &response_type=code                                       │
│      &redirect_uri=http://localhost:PORT/callback              │
│      &code_challenge={code_challenge}                          │
│      &code_challenge_method=S256                               │
│      &scope=openid profile email                                │
│                                                                 │
│  Step 3: 用户在浏览器中登录并授权                               │
│    Anthropic 授权服务器验证用户身份                              │
│                                                                 │
│  Step 4: 回调重定向到本地 HTTP 服务器                           │
│    Claude Code 启动临时 HTTP 服务器监听回调                     │
│    GET http://localhost:PORT/callback?code=AUTH_CODE            │
│                                                                 │
│  Step 5: 用授权码交换 token                                     │
│    POST https://console.anthropic.com/oauth/token               │
│      {                                                          │
│        grant_type: 'authorization_code',                        │
│        code: AUTH_CODE,                                         │
│        redirect_uri: 'http://localhost:PORT/callback',          │
│        client_id: 'claude-code',                                │
│        code_verifier: CODE_VERIFIER  // PKCE 关键步骤           │
│      }                                                          │
│                                                                 │
│  Step 6: 获取 access_token + refresh_token                     │
│    {                                                          │
│      access_token: 'eyJ...',                                   │
│      refresh_token: 'eyJ...',                                  │
│      expires_in: 3600,                                         │
│      token_type: 'Bearer'                                      │
│    }                                                          │
│                                                                 │
│  Step 7: 存储 token 到安全存储                                  │
│    macOS -> Keychain                                           │
│    Windows/Linux -> ~/.claude/credentials.json (加密)           │
│                                                                 │
│  Step 8: 关闭临时 HTTP 服务器                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.17.2 Token 管理

```typescript
// Token 管理策略

interface TokenManager {
  accessToken: string | null;
  refreshToken: string | null;
  expiresAt: number | null;

  // 5 分钟缓冲：在 token 过期前 5 分钟主动刷新
  REFRESH_BUFFER_MS: 300_000;

  async getValidToken(): Promise<string> {
    // 检查是否有有效 token
    if (this.accessToken && this.expiresAt) {
      const now = Date.now();
      const buffer = this.REFRESH_BUFFER_MS;

      if (now < this.expiresAt - buffer) {
        return this.accessToken; // token 仍然有效
      }

      // 即将过期，主动刷新
      try {
        await this.refreshAccessToken();
        return this.accessToken!;
      } catch (error) {
        // 刷新失败，使用旧 token 直到它真正过期
        if (now < this.expiresAt) {
          return this.accessToken!;
        }
        throw error;
      }
    }

    // 没有 token，需要重新认证
    throw new Error('No valid token. Please run claude auth login.');
  }

  private async refreshAccessToken(): Promise<void> {
    if (!this.refreshToken) {
      throw new Error('No refresh token available');
    }

    const response = await fetch('https://console.anthropic.com/oauth/token', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        grant_type: 'refresh_token',
        refresh_token: this.refreshToken,
        client_id: 'claude-code',
      }),
    });

    if (!response.ok) {
      // 刷新 token 也过期了，需要重新认证
      this.clearTokens();
      throw new Error('Refresh token expired. Please re-authenticate.');
    }

    const tokens = await response.json();
    this.accessToken = tokens.access_token;
    this.refreshToken = tokens.refresh_token;
    this.expiresAt = Date.now() + tokens.expires_in * 1000;

    // 持久化到安全存储
    await this.storeTokens();
  }
}
```

### 2.17.3 认证解析链（6 级优先级）

```
┌─────────────────────────────────────────────────────────────────┐
│                 认证解析链（6 级优先级）                          │
│                                                                 │
│  优先级    来源                    说明                          │
│  ──────    ────                    ────                          │
│  最高      3P context              第三方集成（IDE/SDK）提供     │
│            的预配置认证                                           │
│                                                                 │
│  高        bare mode                CLAUDE_CODE_BARE=1           │
│            无认证模式              直接使用 API（无 OAuth）       │
│                                                                 │
│  中高      managed OAuth            企业 MDM 管理的 OAuth        │
│            MDM 策略提供            策略强制指定认证方式          │
│                                                                 │
│  中        explicit tokens          环境变量显式指定              │
│            ANTHROPIC_API_KEY        API key（直接使用）          │
│            ANTHROPIC_AUTH_TOKEN     OAuth token（直接使用）      │
│                                                                 │
│  中低      OAuth                    本地存储的 OAuth token       │
│            Keychain/credentials     自动刷新                    │
│                                                                 │
│  最低      API key                  ~/.claude/credentials        │
│            本地存储的 API key       中的 api_key 字段            │
└─────────────────────────────────────────────────────────────────┘
```

### 2.17.4 安全存储

```
┌─────────────────────────────────────────────────────────────────┐
│                 安全存储策略                                     │
│                                                                 │
│  macOS:                                                         │
│    - 使用系统 Keychain 存储 OAuth token 和 refresh token        │
│    - 服务名称: "com.anthropic.claude-code"                      │
│    - Keychain 缓存 TTL: 5 分钟（避免频繁 Keychain 访问）       │
│    - Stale-while-error: Keychain 访问失败时使用内存缓存         │
│                                                                 │
│  Windows / Linux:                                               │
│    - 使用 ~/.claude/credentials.json（明文）                    │
│    - 文件权限设置为 600（仅用户可读写）                         │
│    - 无 Keychain 缓存（每次直接读取文件）                       │
│    - 未来计划支持 libsecret (Linux) 和 Credential Manager (Win) │
│                                                                 │
│  Token 刷新调度：                                               │
│    - 启动时检查 token 有效性                                    │
│    - 每次 API 调用前检查 token 有效性                          │
│    - 过期前 5 分钟主动刷新                                      │
│    - 刷新失败时使用旧 token 直到真正过期                        │
│    - refresh_token 过期时触发重新认证流程                       │
└─────────────────────────────────────────────────────────────────┘
```

### 2.17.5 MDM 策略

```
┌─────────────────────────────────────────────────────────────────┐
│                 MDM 策略（3 平台路径）                           │
│                                                                 │
│  macOS:                                                         │
│    /Library/Managed Preferences/com.anthropic.claude-code.plist  │
│    - 通过 MDM 配置文件推送                                      │
│    - 支持强制权限模式、禁用功能、配置 MCP 服务器               │
│                                                                 │
│  Windows:                                                       │
│    HKLM\Software\Policies\Anthropic\ClaudeCode                  │
│    - 通过注册表推送                                             │
│    - 支持 Group Policy 配置                                     │
│                                                                 │
│  Linux:                                                         │
│    /etc/claude-code/policy.json                                 │
│    - 通过配置管理工具推送                                       │
│    - 支持 Ansible/Puppet/Chef 集成                              │
│                                                                 │
│  MDM 可控制的策略：                                             │
│  - permissionMode: 强制权限模式                                 │
│  - disabledFeatures: 禁用功能列表                               │
│  - allowedMcpServers: 允许的 MCP 服务器白名单                   │
│  - maxCostPerSession: 单会话最大成本                            │
│  - auditLogEnabled: 是否启用审计日志                            │
│  - telemetryEnabled: 是否启用遥测                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.18 IDE 集成（新增章节）

### 2.18.1 Bridge 协议架构

```
┌─────────────────────────────────────────────────────────────────┐
│                 Bridge 协议架构                                   │
│                                                                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────┐ │
│  │ IDE          │     │ Bridge Layer │     │ Claude Code Core │ │
│  │ Extension    │<───>│ (33 files)   │<───>│ (query.ts, etc)  │ │
│  │              │     │              │     │                  │ │
│  │ - VS Code    │     │ - Protocol   │     │ - QueryEngine    │ │
│  │ - JetBrains  │     │ - Transport  │     │ - Tool Registry  │ │
│  │ - Neovim     │     │ - Auth       │     │ - Permission     │ │
│  │ - Emacs      │     │ - Lifecycle  │     │ - Hooks          │ │
│  └──────────────┘     └──────────────┘     └──────────────────┘ │
│                                                                 │
│  通信协议：JSON-RPC over stdio / WebSocket                      │
│                                                                 │
│  核心文件列表（13 个关键文件）：                                 │
│  bridge/                                                        │
│  ├── protocol.ts           # Bridge 协议定义                    │
│  ├── transport.ts          # 传输层抽象（stdio/ws）              │
│  ├── auth.ts               # Bridge 认证                        │
│  ├── lifecycle.ts          # 生命周期管理                       │
│  ├── messageRouter.ts      # 消息路由                           │
│  ├── sessionManager.ts     # 会话管理                           │
│  ├── notificationHandler.ts # 通知处理                          │
│  ├── fileWatcher.ts        # 文件监视同步                       │
│  ├── diffViewer.ts         # Diff 查看器集成                    │
│  ├── planMode.ts           # Plan 模式集成                      │
│  ├── ideMcpServer.ts       # IDE MCP 服务器                     │
│  ├── contextProvider.ts    # 上下文提供者                       │
│  └── healthCheck.ts        # 健康检查                           │
└─────────────────────────────────────────────────────────────────┘
```

### 2.18.2 VS Code 集成

- **官方扩展**：Claude Code 提供 VS Code 扩展，通过 Bridge 协议与核心通信
- **IDE MCP 服务器**：每个 IDE 实例启动一个 MCP 服务器，Claude Code 通过 MCP 获取 IDE 上下文（打开的文件、选中的代码、诊断信息等）
- **实时同步**：文件变更、光标位置、选区变更实时同步到 Claude Code
- **diff viewer**：工具执行结果中的文件变更通过 IDE 的 diff viewer 展示
- **Plan 模式**：在 IDE 中显示 Claude 的执行计划，用户可以审查后批准

---

## 2.19 LSP 集成（新增章节）

### 2.19.1 LSPClient.ts

Claude Code 内置了 LSP（Language Server Protocol）客户端，可以直接与语言服务器通信获取代码智能信息：

```typescript
// services/lsp/LSPClient.ts -- JSON-RPC over stdio

export class LSPClient {
  private process: ChildProcess;
  private messageId: number = 0;
  private pendingRequests: Map<number, {
    resolve: (result: any) => void;
    reject: (error: Error) => void;
  }> = new Map();
  private eventQueue: LSPEvent[] = [];
  private initialized: boolean = false;

  constructor(
    private serverCommand: string,
    private serverArgs: string[],
    private rootUri: string,
  ) {}

  async start(): Promise<void> {
    // 启动语言服务器进程
    this.process = spawn(this.serverCommand, this.serverArgs, {
      stdio: ['pipe', 'pipe', 'pipe'],
      cwd: this.rootUri,
    });

    // 监听 stdout 的 JSON-RPC 消息
    this.process.stdout.on('data', (data: Buffer) => {
      const messages = parseLSPMessages(data);
      for (const msg of messages) {
        this.handleMessage(msg);
      }
    });

    // 发送 initialize 请求
    await this.sendRequest('initialize', {
      processId: process.pid,
      rootUri: this.rootUri,
      capabilities: {
        textDocument: {
          completion: { completionItem: { snippetSupport: false } },
          hover: { contentFormat: ['markdown', 'plaintext'] },
          definition: { linkSupport: true },
          references: {},
          typeDefinition: { linkSupport: true },
          callHierarchy: { prepareSupport: true },
        },
      },
    });

    // 发送 initialized 通知
    this.sendNotification('initialized', {});
    this.initialized = true;
  }

  // 延迟队列：确保服务器初始化完成后再处理请求
  private async ensureInitialized(): Promise<void> {
    if (!this.initialized) {
      await new Promise(resolve => {
        const check = () => {
          if (this.initialized) resolve();
          else setTimeout(check, 100);
        };
        check();
      });
    }
  }
}
```

### 2.19.2 LSPServerManager.ts

管理多个 LSP 服务器实例，按文件扩展名路由到对应的语言服务器：

```typescript
// services/lsp/LSPServerManager.ts -- 多实例管理

export class LSPServerManager {
  private clients: Map<string, LSPClient> = new Map();
  private extensionMap: Map<string, string> = new Map();

  constructor() {
    // 文件扩展名 -> 语言服务器映射
    this.extensionMap.set('.ts', 'typescript');
    this.extensionMap.set('.tsx', 'typescript');
    this.extensionMap.set('.js', 'typescript');
    this.extensionMap.set('.jsx', 'typescript');
    this.extensionMap.set('.py', 'python');
    this.extensionMap.set('.go', 'gopls');
    this.extensionMap.set('.rs', 'rust-analyzer');
    this.extensionMap.set('.java', 'jdtls');
    this.extensionMap.set('.cpp', 'clangd');
    this.extensionMap.set('.c', 'clangd');
  }

  // 按需启动：第一次访问某语言时才启动对应的服务器
  async getClientForFile(filePath: string): Promise<LSPClient | null> {
    const ext = path.extname(filePath);
    const language = this.extensionMap.get(ext);

    if (!language) return null;

    if (!this.clients.has(language)) {
      const serverConfig = this.getServerConfig(language);
      if (!serverConfig) return null;

      const client = new LSPClient(
        serverConfig.command,
        serverConfig.args,
        this.rootUri,
      );
      await client.start();
      this.clients.set(language, client);
    }

    return this.clients.get(language)!;
  }

  // 关闭所有服务器
  async dispose(): Promise<void> {
    for (const client of this.clients.values()) {
      await client.dispose();
    }
    this.clients.clear();
  }
}
```

### 2.19.3 支持的操作

| 操作 | LSP 方法 | 用途 |
|------|----------|------|
| 代码导航 | `textDocument/definition` | 跳转到定义 |
| 代码信息 | `textDocument/hover` | 获取类型信息、文档 |
| 查找引用 | `textDocument/references` | 查找所有引用位置 |
| 类型定义 | `textDocument/typeDefinition` | 跳转到类型定义 |
| 调用层次 | `callHierarchy/incomingCalls` | 查找调用者 |
| 调用层次 | `callHierarchy/outgoingCalls` | 查找被调用者 |
| 符号搜索 | `workspace/symbol` | 全局符号搜索 |
| 文件生命周期 | `textDocument/didOpen` | 通知服务器文件已打开 |
| 文件生命周期 | `textDocument/didChange` | 通知服务器文件已变更 |
| 文件生命周期 | `textDocument/didClose` | 通知服务器文件已关闭 |

---

> 本章完。下一章将深入分析 Claude Code 的工具实现细节、MCP 协议栈、以及与 Codex CLI 的架构对比。---
# Codex CLI 架构深度分析

## 3.1 Cargo Workspace 微服务化架构

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
│  │  │ client   │ │ manager  │ │  auth    │ │  otel     │  │   │
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

## 3.2 核心目录结构

### 顶层结构

```
openai/codex/
├── codex-rs/              # [核心] Rust 实现 — 整个项目的主体
├── codex-cli/             # [已废弃] 旧版 TypeScript/Node.js 实现
├── sdk/                   # SDK 相关代码
├── docs/                  # 项目文档
├── scripts/               # 辅助脚本
├── .codex/skills/         # Codex 自身使用的项目级技能定义
├── AGENTS.md              # AI 代理开发指南
├── BUILD.bazel            # Bazel 构建配置
├── MODULE.bazel           # Bazel 模块定义
├── flake.nix              # Nix 构建支持
├── justfile               # just 任务运行器配置
└── pnpm-workspace.yaml    # pnpm monorepo 工作区定义
```

### codex-rs/ 核心 Crate 分类

**三大入口 Crate：**

| Crate | 路径 | 职责 |
|-------|------|------|
| **`cli`** | `codex-rs/cli/` | CLI 多工具入口，提供子命令路由（tui、exec、sandbox、mcp 等），使用 clap 进行参数解析 |
| **`tui`** | `codex-rs/tui/` | 全屏终端 UI，基于 Ratatui 框架构建，处理用户交互、键盘输入、聊天界面渲染 |
| **`exec`** | `codex-rs/exec/` | 无头 CLI，用于自动化/非交互式执行 |

**核心业务逻辑 Crate：**

| Crate | 路径 | 职责 |
|-------|------|------|
| **`core`** | `codex-rs/core/` | 最核心的 crate，包含 Agent 循环、模型调用、工具执行、沙箱管理、状态管理等 |
| **`tools`** | `codex-rs/tools/` | 共享工具相关代码：工具 schema 定义、工具规范（ToolSpec）、MCP 工具适配器等 |
| **`protocol`** | `codex-rs/protocol/` | 通信协议定义 |

**core/ 内部模块结构（核心中的核心）：**

```
core/src/
├── agent/                  # Agent 循环核心逻辑
├── codex.rs                # 主 Codex 结构体和事件循环 (~2000行)
├── codex_thread.rs         # Codex 线程管理
├── codex_delegate.rs       # Codex 委托模式
├── client.rs               # OpenAI API 客户端
├── tools/                  # 内置工具实现
├── sandboxing/             # 沙箱策略和执行
├── state/                  # 会话状态管理
├── config/                 # 配置模型
├── context_manager/        # 上下文管理（对话历史压缩等）
├── guardian/               # 安全守护（命令审批策略）
├── instructions/           # 系统指令管理
├── memories/               # 记忆系统（跨会话持久化）
├── plugins/                # 插件系统
├── tasks/                  # 任务管理 (mod.rs: 467行)
├── snapshots/              # 文件快照
├── apply_patch.rs          # 补丁应用（文件编辑核心）
├── safety.rs               # 安全策略
├── mcp.rs                  # MCP 集成
├── message_history.rs      # 消息历史管理
├── compact.rs              # 上下文压缩 (442行)
├── compact_remote.rs       # 远程上下文压缩
├── file_watcher.rs         # 文件监视器
├── hook_runtime.rs         # Hook 运行时
├── project_doc.rs          # 项目文档发现 (codex.md) (315行)
├── error.rs                # 错误处理 (659行)
├── truncate.rs             # 截断工具 (363行)
├── seatbelt.rs             # macOS Seatbelt 沙箱 (623行)
├── model_provider_info.rs  # 模型提供商信息 (396行)
└── lib.rs                  # 库入口
```

**沙箱与安全 Crate：**

| Crate | 路径 | 职责 |
|-------|------|------|
| **`sandboxing`** | `codex-rs/sandboxing/` | 沙箱抽象层 |
| **`linux-sandbox`** | `codex-rs/linux-sandbox/` | Linux 沙箱实现（Landlock + Bubblewrap） |
| **`windows-sandbox-rs`** | `codex-rs/windows-sandbox-rs/` | Windows 沙箱实现 |
| **`process-hardening`** | `codex-rs/process-hardening/` | 进程加固 |
| **`execpolicy`** | `codex-rs/execpolicy/` | 执行策略定义 |
| **`shell-escalation`** | `codex-rs/shell-escalation/` | Shell 权限升级管理 |

**MCP 相关 Crate：**

| Crate | 路径 | 职责 |
|-------|------|------|
| **`mcp-server`** | `codex-rs/mcp-server/` | MCP 服务器实现 |
| **`rmcp-client`** | `codex-rs/rmcp-client/` | MCP 客户端 |
| **`mcp-types`** | `codex-rs/mcp-types/` | MCP 类型定义 |

**模型与 API 客户端 Crate：**

| Crate | 路径 | 职责 |
|-------|------|------|
| **`codex-client`** | `codex-rs/codex-client/` | Codex 客户端封装 |
| **`codex-api`** | `codex-rs/codex-api/` | Codex API 定义 |
| **`chatgpt`** | `codex-rs/chatgpt/` | ChatGPT 登录/认证集成 |
| **`lmstudio`** | `codex-rs/lmstudio/` | LM Studio 本地模型支持 |
| **`ollama`** | `codex-rs/ollama/` | Ollama 本地模型支持 |
| **`models-manager`** | `codex-rs/models-manager/` | 模型管理 |

### 关键源文件索引表

| 模块 | 源文件路径 | 行数 |
|------|-----------|------|
| Agent 核心 | codex-rs/core/src/codex.rs | ~2000 |
| 任务执行 | codex-rs/core/src/tasks/mod.rs | 467 |
| SSE 流处理 | codex-rs/codex-api/src/sse/responses.rs | ~400 |
| HTTP 端点 | codex-rs/codex-api/src/endpoint/responses.rs | 151 |
| WebSocket 端点 | codex-rs/codex-api/src/endpoint/responses_websocket.rs | 832 |
| 错误处理 | codex-rs/core/src/error.rs | 659 |
| 压缩算法 | codex-rs/core/src/compact.rs | 442 |
| 截断工具 | codex-rs/core/src/truncate.rs | 363 |
| Patch 解析 | codex-rs/apply-patch/src/parser.rs | 741 |
| Patch 应用 | codex-rs/apply-patch/src/lib.rs | 1672 |
| Seatbelt 沙箱 | codex-rs/core/src/seatbelt.rs | 623 |
| Landlock 沙箱 | codex-rs/linux-sandbox/src/landlock.rs | ~300 |
| 调试沙箱 | codex-rs/cli/src/debug_sandbox.rs | 558 |
| 项目文档发现 | codex-rs/core/src/project_doc.rs | 315 |
| Rollout 记录 | codex-rs/rollout/src/recorder.rs | 1111 |
| 协议定义 | codex-rs/protocol/src/protocol.rs | ~450 |
| 模型提供商 | codex-rs/core/src/model_provider_info.rs | 396 |

---

## 3.3 Agent 循环（大幅扩充）

Codex CLI 的 Agent 循环采用经典的 **ReAct（Reasoning + Acting）模式**，以请求-响应式循环运行。整个循环建立在 **Op/Event 提交-事件模式** 之上，通过 Submission Queue 和 Event Queue 实现异步通信。

### 3.3.1 Op/Event 提交-事件模式

Codex CLI 的核心通信架构采用 **SQ/EQ（Submission Queue / Event Queue）模式**，这是整个系统的通信骨干。

```
┌─────────────────────────────────────────────────────────────────┐
│                    SQ/EQ 通信架构                                │
│                                                                  │
│  ┌──────────┐     Submission      ┌──────────────┐              │
│  │  Client   │────Queue (tx_sub)──▶│   Codex      │              │
│  │ (TUI/Exec)│                     │   Agent      │              │
│  │           │◀──Event Queue──────│   Loop       │              │
│  │           │     (rx_event)      │              │              │
│  └──────────┘                      └──────────────┘              │
│       │                                  │                       │
│       │  Op::UserInput                  │ EventMsg              │
│       │  Op::Interrupt                  │ EventMsg::ItemStarted  │
│       │  Op::ExecApproval               │ EventMsg::ItemCompleted│
│       │  Op::Shutdown                   │ EventMsg::TurnAborted  │
│       ▼                                  ▼                       │
│  ┌──────────┐                      ┌──────────────┐              │
│  │ 用户输入   │                      │ 模型响应     │              │
│  │ 审批决策   │                      │ 工具执行结果  │              │
│  │ 中断信号   │                      │ 状态变更通知  │              │
│  └──────────┘                      └──────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

**核心数据结构 — Submission：**

```rust
/// Submission Queue Entry - requests from user
#[derive(Debug, Clone, Deserialize, Serialize, JsonSchema)]
pub struct Submission {
    /// Unique id for this Submission to correlate with Events
    pub id: String,
    /// Payload
    pub op: Op,
    /// Optional W3C trace carrier propagated across async submission handoffs.
    pub trace: Option<W3cTraceContext>,
}
```

**核心数据结构 — Codex 结构体：**

`codex.rs`（~2000 行）定义了整个 Agent 系统的核心结构体，包含：

```rust
pub struct Codex {
    // 通信通道
    tx_sub: Sender<Submission>,      // 提交队列发送端
    rx_event: Receiver<EventMsg>,    // 事件队列接收端

    // 状态管理
    agent_status: Arc<watch::Sender<AgentStatus>>,  // Agent 状态广播
    session: Arc<Session>,           // 会话实例

    // 配置
    config: Config,                  // 全局配置

    // 服务
    services: Arc<SessionServices>,  // 会话级服务集合
}
```

**核心数据结构 — Session 结构体：**

```rust
pub struct Session {
    /// 唯一会话标识符
    pub conversation_id: ThreadId,

    /// 会话状态（包含消息历史、token 用量等）
    pub state: Arc<RwLock<SessionState>>,

    /// 会话级服务集合
    pub services: Arc<SessionServices>,

    // ... 其他字段
}
```

**核心数据结构 — ActiveTurn 和 CancellationToken：**

```rust
/// 表示当前正在进行的轮次
pub struct ActiveTurn {
    pub turn_id: String,
    pub cancellation_token: CancellationToken,
    pub turn_context: Arc<TurnContext>,
    // ...
}

/// Tokio 的 CancellationToken 用于协作式取消
/// 当用户发送 Interrupt 时，通过 cancel() 通知所有正在运行的任务
```

### 3.3.2 提交循环 (Submission Loop)

提交循环是 Codex 结构体的核心事件循环，持续从 `rx_sub` 接收 Submission 并分发处理。

```
┌──────────────────────────────────────────────────────────────┐
│                    Submission Loop                            │
│                                                               │
│   loop {                                                      │
│       match rx_sub.recv().await {                             │
│           Op::UserInput { items, .. } => {                   │
│               spawn_task(items, turn_context)                 │
│               // 构建上下文 → 调用 API → 流式处理 → 工具执行  │
│           }                                                   │
│                                                               │
│           Op::Interrupt => {                                  │
│               abort_all_tasks()                               │
│               // 取消所有正在运行的任务                         │
│               // 发送 TurnAborted 事件                         │
│           }                                                   │
│                                                               │
│           Op::ExecApproval { approved } => {                  │
│               resolve_pending_approval(approved)              │
│               // 将用户的审批决策传递给等待中的工具执行          │
│           }                                                   │
│                                                               │
│           Op::Shutdown => {                                   │
│               shutdown()                                      │
│               break;                                          │
│           }                                                   │
│       }                                                       │
│   }                                                           │
└──────────────────────────────────────────────────────────────┘
```

**Op 枚举完整定义（核心变体）：**

```rust
#[derive(Debug, Clone, Deserialize, Serialize, PartialEq, JsonSchema)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum Op {
    /// 中断当前任务（不终止后台终端进程）
    Interrupt,

    /// 终止所有后台终端进程
    CleanBackgroundTerminals,

    /// 开始实时对话流
    RealtimeConversationStart(ConversationStartParams),

    /// 发送音频输入到实时对话流
    RealtimeConversationAudio(ConversationAudioParams),

    /// 发送文本输入到实时对话流
    RealtimeConversationText(ConversationTextParams),

    /// 关闭实时对话流
    RealtimeConversationClose,

    /// 请求实时对话支持的语音列表
    RealtimeConversationListVoices,

    /// 传统用户输入
    UserInput {
        items: Vec<UserInput>,
        final_output_json_schema: Option<Value>,
        responsesapi_client_metadata: Option<HashMap<String, String>>,
    },

    /// 完整轮次输入（包含 cwd、审批策略、沙箱策略等上下文）
    UserTurn {
        items: Vec<UserInput>,
        cwd: PathBuf,
        approval_policy: AskForApproval,
        approvals_reviewer: Option<ApprovalsReviewer>,
        sandbox_policy: SandboxPolicy,
        model: String,
        effort: Option<ReasoningEffortConfig>,
        summary: Option<ReasoningSummaryConfig>,
        service_tier: Option<Option<ServiceTier>>,
        final_output_json_schema: Option<Value>,
        // ...
    },

    /// 执行审批响应
    ExecApproval {
        id: String,
        approved: bool,
    },

    /// 关闭会话
    Shutdown,
}
```

### 3.3.3 Responses API 调用格式

Codex CLI 使用 OpenAI **Responses API**（而非传统的 Chat Completions API），每次请求发送完整的 JSON 负载。

**JSON 负载的三个关键字段：**

```json
{
  "instructions": "系统指令内容...",
  "tools": [...],
  "input": [...]
}
```

| 字段 | 说明 | 来源 |
|------|------|------|
| `instructions` | 系统指令（base_instructions） | 模型默认指令 + 沙箱权限说明 + 开发者指令 |
| `tools` | 可用工具列表 | 内置工具 + API 提供的工具 + MCP 工具 |
| `input` | 对话输入序列 | 按序插入的多条消息 |

**instructions 来源优先级：**

```
1. 模型默认指令 (ModelProviderInfo.base_instructions)
   ↓
2. 沙箱权限说明 (<permissions instructions>)
   ↓
3. 开发者指令 (developer_instructions)
   ↓
4. 最终合并为 base_instructions
```

**tools 三类来源：**

| 类型 | 说明 | 示例 |
|------|------|------|
| **内置工具** | Codex CLI 自带的工具 | `shell`、`apply_patch`、`view_image`、`js_repl` |
| **API 提供工具** | Responses API 内置的工具 | `web_search`、`code_interpreter` |
| **MCP 工具** | 通过 MCP 协议连接的外部工具 | `mcp__server__tool_name` |

**input 按序插入规则：**

```
input 序列:
├── [0] developer 消息 — 沙箱权限说明
│       role: "developer"
│       content: <permissions instructions>
│
├── [1] developer 消息 — 开发者指令
│       role: "developer"
│       content: developer_instructions
│
├── [2] user 消息 — 用户指令
│       role: "user"
│       content: <user_instructions>...</user_instructions>
│
├── [3] user 消息 — 环境上下文
│       role: "user"
│       content: <environment_context>...</environment_context>
│
├── [4] user 消息 — 用户实际输入
│       role: "user"
│       content: "## My request for Codex:\n{用户输入}"
│
├── [5+] 历史消息（压缩后的对话历史）
│       交替的 assistant/user 消息
│       包含工具调用和工具结果
│
└── [最后] 当前用户输入
```

### 3.3.4 SSE 流式处理

Codex CLI 通过 **Server-Sent Events (SSE)** 接收模型的流式响应。核心实现在 `codex-api/src/sse/responses.rs`（~400 行）。

**关键 SSE 事件类型：**

| 事件类型 | 说明 | 处理方式 |
|----------|------|----------|
| `response.reasoning_summary_text.delta` | 推理摘要增量文本 | 累积并显示推理过程 |
| `response.output_item.added` | 输出项添加（工具调用开始） | 创建新的工具调用上下文 |
| `response.output_text.delta` | 输出文本增量 | 实时显示给用户 |
| `response.completed` | 响应完成 | 提取最终结果，判断是否需要继续循环 |
| `response.function_call_arguments.delta` | 函数调用参数增量 | 累积函数调用参数 |
| `response.output_item.done` | 输出项完成 | 触发工具执行 |

**SSE 消费实现（简化）：**

```rust
// codex-api/src/sse/responses.rs 中的核心流处理逻辑
async fn stream_response(
    response: Response,
    tx_event: Sender<EventMsg>,
) -> Result<()> {
    let mut stream = response.bytes_stream();
    let mut buffer = String::new();

    while let Some(chunk) = stream.next().await {
        buffer.push_str(&chunk?);

        // 解析 SSE 事件
        while let Some(pos) = buffer.find("\n\n") {
            let event_text = buffer[..pos].to_string();
            buffer = buffer[pos + 2..].to_string();

            for line in event_text.lines() {
                if let Some(data) = line.strip_prefix("data: ") {
                    let event: ResponseEvent = serde_json::from_str(data)?;
                    match event {
                        ResponseEvent::ReasoningSummaryTextDelta { delta } => {
                            tx_event.send(EventMsg::ReasoningContentDelta(
                                ReasoningContentDeltaEvent { content: delta }
                            )).await?;
                        }
                        ResponseEvent::OutputItemAdded { item } => {
                            tx_event.send(EventMsg::ItemStarted(
                                ItemStartedEvent { item }
                            )).await?;
                        }
                        ResponseEvent::OutputTextDelta { delta } => {
                            tx_event.send(EventMsg::AgentMessageContentDelta(
                                AgentMessageContentDeltaEvent { delta }
                            )).await?;
                        }
                        ResponseEvent::Completed { response } => {
                            // 处理完成事件
                        }
                        _ => {}
                    }
                }
            }
        }
    }
    Ok(())
}
```

**WebSocket 传输支持（responses_websocket.rs, 832 行）：**

除了标准 SSE，Codex CLI 还支持通过 WebSocket 传输响应流。WebSocket 传输层在 `codex-api/src/endpoint/responses_websocket.rs`（832 行）中实现，主要优势包括：

- **双向通信**：支持在流式传输过程中发送中断信号
- **增量请求**：WebSocket 支持增量式请求更新（incremental request tracking），减少重复传输
- **粘性路由**：WebSocket 连接保持会话亲和性，确保请求路由到同一后端实例

```rust
// WebSocket 端点处理
// codex-api/src/endpoint/responses_websocket.rs
pub async fn handle_websocket_response(
    ws: WebSocketStream,
    request: CreateResponseRequest,
    cancellation: CancellationToken,
) -> Result<()> {
    // 建立 WebSocket 连接
    // 发送初始请求
    // 接收流式响应并通过 WebSocket 发送
    // 处理客户端中断
}
```

### 3.3.5 工具调用结果反馈

当模型发出工具调用后，Codex CLI 执行工具并将结果以 `function_call_output` 格式反馈给模型。

**function_call_output 格式：**

```json
{
  "type": "function_call_output",
  "call_id": "call_abc123",
  "output": "工具执行结果文本..."
}
```

**关键设计 — 旧提示词是新提示词的精确前缀：**

这是 Codex CLI 最核心的缓存优化策略。每次请求都携带完整的对话历史，但新请求的 `input` 序列是前一次请求的 `input` 序列的**精确前缀**加上新的工具结果：

```
请求 N 的 input:
  [msg_0, msg_1, msg_2, ..., msg_n]

请求 N+1 的 input:
  [msg_0, msg_1, msg_2, ..., msg_n, tool_result_n+1]
  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  精确前缀匹配 — API 可以复用之前计算的 KV 缓存
```

这种设计意味着：
- API 服务端可以利用前缀缓存（prefix caching），避免重复计算前面消息的注意力矩阵
- 不需要维护服务端状态（`previous_response_id`）
- 简化了错误恢复和重试逻辑

### 3.3.6 无状态请求 + 前缀缓存

**不使用 `previous_response_id` 的原因：**

Codex CLI 明确选择不使用 OpenAI API 的 `previous_response_id` 参数，原因有三：

1. **零数据保留（ZDR）**：不依赖服务端状态意味着可以实现零数据保留模式，所有对话历史仅存储在本地
2. **简化实现**：不需要处理服务端状态不一致、会话过期等问题
3. **前缀缓存优化**：精确前缀匹配在现代 LLM 推理引擎中已经非常高效（如 vLLM 的 RadixAttention），不依赖服务端状态也能获得良好的缓存命中率

**Zstd 压缩支持：**

对于超长对话历史，Codex CLI 支持使用 Zstd 压缩来减少传输数据量：

```rust
// 在构建请求时，如果历史超过阈值，使用 Zstd 压缩
fn compress_history_if_needed(items: &[ResponseItem]) -> Vec<ResponseItem> {
    if should_compress(items) {
        // 使用 Zstd 压缩旧消息
        let compressed = zstd::encode_all(items_to_bytes(items), 3)?;
        // 将压缩后的数据包装为特殊消息类型
    }
    items.to_vec()
}
```

**stream_request 函数（核心请求流程）：**

```rust
// codex.rs 中的核心请求函数（简化）
async fn stream_request(
    sess: &Session,
    turn_context: &TurnContext,
    client_session: &mut ModelClientSession,
    prompt: &Prompt,
) -> Result<()> {
    // 1. 构建完整的请求
    let request = build_request(prompt, turn_context);

    // 2. 发送请求并处理流式响应
    let response = client_session
        .stream_request(request, turn_context.cancellation_token.clone())
        .await?;

    // 3. 消费流式响应
    drain_to_completed(sess, turn_context, client_session, &prompt, response).await
}

async fn drain_to_completed(
    sess: &Session,
    turn_context: &TurnContext,
    client_session: &mut ModelClientSession,
    prompt: &Prompt,
) -> Result<()> {
    // 处理流式事件直到收到 completed 事件
    // 解析工具调用
    // 执行工具
    // 将工具结果追加到历史
    // 如果有工具调用，递归调用 stream_request
}
```

---

## 3.4 工具系统（大幅扩充）

### 3.4.1 内置工具

Codex CLI 提供以下内置工具：

| 工具名称 | 说明 | 实现位置 |
|----------|------|----------|
| **`shell`** | 在沙箱中执行 Shell 命令 | `core/src/tools/shell.rs` |
| **`apply_patch`** | 对文件应用差异补丁（核心文件编辑能力） | `apply-patch/src/` |
| **`view_image`** | 查看图片 | `core/src/tools/view_image.rs` |
| **`js_repl`** | JavaScript REPL 执行（实验性，feature-gated） | `core/src/tools/js_repl.rs` |

**shell 工具执行流程：**

```
模型发出 shell 调用
    │
    ▼
Guardian 审批检查
    │
    ├── canAutoApprove → 自动执行
    │
    └── 需要审批 → 发送 ExecApprovalRequestEvent
                      │
                      ▼
                 用户确认/拒绝
                      │
                      ▼
              在沙箱中执行命令
                      │
                      ▼
              收集 stdout/stderr
                      │
                      ▼
              截断输出（防止上下文膨胀）
                      │
                      ▼
              返回 function_call_output
```

### 3.4.2 工具 Schema 系统

工具 Schema 系统定义在 `codex-tools` crate 中，提供了多层抽象：

```rust
/// 工具规范 — 定义工具的名称、描述和参数 schema
pub struct ToolSpec {
    pub name: String,
    pub description: String,
    pub parameters: serde_json::Value,  // JSON Schema
}

/// 工具定义 — 包含工具规范和执行逻辑
pub struct ToolDefinition {
    pub spec: ToolSpec,
    pub handler: Box<dyn Fn(ToolInput) -> ToolOutput>,
}

/// 配置后的工具规范 — 经过配置系统处理后的工具
pub struct ConfiguredToolSpec {
    pub spec: ToolSpec,
    pub is_enabled: bool,
    pub requires_approval: bool,
}

/// Responses API 工具适配 — 转换为 API 格式
pub struct ResponsesApiTool {
    pub type_: String,        // "function"
    pub name: String,
    pub description: String,
    pub parameters: Value,
    pub strict: bool,
}

/// 自由格式工具 — 用于 js_repl 等非标准工具
pub struct FreeformTool {
    pub name: String,
    pub description: String,
    pub input_schema: Option<Value>,
}
```

**tool_search / tool_suggest 动态发现：**

当配置了大量 MCP 工具时，将所有工具 schema 加载到系统提示会浪费大量 token。Codex CLI 提供动态发现机制：

```rust
/// 工具搜索 — 按需发现工具的完整 schema
pub async fn tool_search(query: &str) -> Vec<ToolSpec> {
    // 1. 在已注册的工具中搜索匹配的工具
    // 2. 在 MCP 工具中搜索
    // 3. 返回匹配工具的完整 schema
}

/// 工具建议 — 基于上下文推荐可能需要的工具
pub async fn tool_suggest(context: &TurnContext) -> Vec<ToolSpec> {
    // 基于当前对话上下文，推荐可能需要的工具
}
```

### 3.4.3 apply_patch.rs 文件编辑（大幅扩充）

`apply-patch` 是 Codex CLI 的核心文件编辑能力，独立为 `codex-apply-patch` crate（parser.rs: 741 行 + lib.rs: 1672 行）。

#### Patch 格式语法

Patch 格式使用自定义的标记语言，由 Lark 语法定义：

```
start: begin_patch hunk+ end_patch

begin_patch:  "*** Begin Patch" LF
end_patch:    "*** End Patch" LF?

hunk: add_hunk | delete_hunk | update_hunk

add_hunk:    "*** Add File: " filename LF add_line+
delete_hunk: "*** Delete File: " filename LF
update_hunk: "*** Update File: " filename LF change_move? change?

change_move: "*** Move to: " filename LF
change: (change_context | change_line)+ eof_line?

change_context: ("@@" | "@@ " /(.+)/) LF
change_line: ("+" | "-" | " ") /(.+)/ LF
eof_line: "*** End of File" LF
```

**完整示例：**

```diff
*** Begin Patch
*** Add File: new_module.rs
+ pub fn new_function() -> i32 {
+     42
+ }
*** Update File: src/main.rs
@@ fn main() {
     println!("Hello");
-    old_line();
+    new_line();
 }
*** Delete File: deprecated.rs
*** End Patch
```

#### 解析器实现

解析器支持两种模式：

```rust
/// 解析模式
enum ParseMode {
    /// 严格模式 — 完全按照语法规范解析
    Strict,

    /// 宽松模式 — 兼容 GPT-4.1 的 heredoc 格式
    ///
    /// GPT-4.1 可能生成如下格式：
    /// ```json
    /// ["apply_patch", "<<'EOF'\n*** Begin Patch\n...\n*** End Patch\nEOF\n"]
    /// ```
    /// 在宽松模式下，解析器会检测并剥离 heredoc 包装
    Lenient,
}
```

**解析流程：**

```
输入: patch 文本
    │
    ▼
check_patch_boundaries_strict()
    │
    ├── 成功 → 解析 hunk
    │
    └── 失败 → check_patch_boundaries_lenient()
                    │
                    ├── 检测 heredoc 标记 (<<'EOF' ... EOF)
                    │   │
                    │   ├── 成功 → 剥离 heredoc，重新检查边界
                    │   │
                    │   └── 失败 → 返回 ParseError
                    │
                    └── 解析 hunk
```

**Hunk 数据结构：**

```rust
#[derive(Debug, PartialEq, Clone)]
pub enum Hunk {
    /// 添加新文件
    AddFile {
        path: PathBuf,
        contents: String,
    },

    /// 删除文件
    DeleteFile {
        path: PathBuf,
    },

    /// 更新文件（支持重命名）
    UpdateFile {
        path: PathBuf,
        move_path: Option<PathBuf>,     // 重命名目标路径
        chunks: Vec<UpdateFileChunk>,   // 按顺序排列的修改块
    },
}

/// 文件更新块
#[derive(Debug, PartialEq, Clone)]
pub struct UpdateFileChunk {
    /// 上下文行 — 用于定位修改位置（通常是类/方法/函数定义）
    pub change_context: Option<String>,

    /// 旧行 — 需要被替换的行
    pub old_lines: Vec<String>,

    /// 新行 — 替换后的行
    pub new_lines: Vec<String>,

    /// 是否在文件末尾
    pub is_end_of_file: bool,
}
```

#### 冲突处理策略

当 patch 无法干净地应用到文件时，系统采用多级策略处理：

1. **上下文匹配**：使用 `seek_sequence` 算法在文件中查找 `change_context` 指定的上下文行，精确定位修改位置

```rust
/// 在文件行中搜索目标序列
fn seek_sequence(
    lines: &[String],
    target: &[String],
    start_from: usize,
    eof: bool,
) -> Option<usize> {
    // 从 start_from 位置开始搜索
    // 如果 eof=true，也在文件末尾搜索
}
```

2. **多级 @@ 标记**：支持多个 `@@` 上下文标记来精确定位修改位置，减少歧义

3. **EOF 标记**：`*** End of File` 标记指示修改在文件末尾，系统会容忍尾部换行差异

4. **路径解析**：所有相对路径都基于 `cwd` 解析为绝对路径，使用 `AbsolutePathBuf` 确保路径安全

```rust
impl Hunk {
    /// 将相对路径解析为绝对路径
    pub fn resolve_path(&self, cwd: &AbsolutePathBuf) -> AbsolutePathBuf {
        let path = match self {
            Hunk::UpdateFile { path, .. } => path,
            Hunk::AddFile { .. } | Hunk::DeleteFile { .. } => self.path(),
        };
        AbsolutePathBuf::resolve_path_against_base(path, cwd)
    }
}
```

---

## 3.5 沙箱系统（大幅扩充）

沙箱是 Codex CLI 最重要的安全特性，采用**平台原生沙箱**方案，实现真正的进程级隔离。

### 3.5.1 macOS Seatbelt 沙箱

macOS 使用 Apple 原生的 **Seatbelt** (`sandbox-exec`) 沙箱系统。核心实现在 `core/src/seatbelt.rs`（623 行）。

**SBPL 策略结构：**

Seatbelt 沙箱通过 SBPL（Sandbox Profile Language）配置文件控制权限。Codex CLI 动态生成 SBPL 策略：

```
┌─────────────────────────────────────────────────────────────┐
│                    SBPL 策略结构                             │
│                                                              │
│  (version 1)                                                 │
│  (deny default)                                              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 基础策略                                              │   │
│  │  (allow process-exec)                                 │   │
│  │  (allow sysctl-read)                                  │   │
│  │  (allow file-read*)                                   │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 文件读取策略                                          │   │
│  │  (allow file-read* (subpath "/usr"))                  │   │
│  │  (allow file-read* (subpath "/bin"))                  │   │
│  │  (allow file-read* (subpath "/opt/homebrew"))         │   │
│  │  (allow file-read* (subpath "{cwd}"))                 │   │
│  │  (allow file-read* (subpath "{home}"))                │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 文件写入策略（workspace-write 模式）                   │   │
│  │  (allow file-write* (subpath "{cwd}"))                │   │
│  │  (deny file-write* (subpath "{cwd}/.git"))            │   │
│  │  (deny file-write* (subpath "{cwd}/.codex"))          │   │
│  │  (deny file-write* (subpath "{cwd}/.hg"))             │   │
│  │  (deny file-write* (subpath "{cwd}/.svn"))            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 网络策略                                              │   │
│  │  read-only 模式: (deny network*)                      │   │
│  │  workspace-write 模式: (deny network*)                │   │
│  │  danger-full-access 模式: (allow network*)            │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**create_seatbelt_command_args 代码示例：**

```rust
// codex-rs/sandboxing/src/seatbelt.rs
pub fn create_seatbelt_command_args_for_policies(
    command: Vec<String>,
    fs_policy: &FileSystemSandboxPolicy,
    network_policy: NetworkSandboxPolicy,
    sandbox_policy_cwd: &Path,
    enforce_managed_network: bool,
    network: Option<&NetworkProxy>,
) -> Vec<String> {
    let mut args = Vec::new();

    // 构建 SBPL 配置文件
    let profile = build_sbpl_profile(fs_policy, network_policy, sandbox_policy_cwd);

    args.push("-p".to_string());  // 使用内联配置文件
    args.push(profile);

    // 传递原始命令
    args.extend(command);

    args
}
```

**-D 参数传递目录路径：**

```rust
// 通过 -D 参数将目录路径注入到 SBPL 配置中
fn build_sbpl_profile(
    fs_policy: &FileSystemSandboxPolicy,
    network_policy: NetworkSandboxPolicy,
    cwd: &Path,
) -> String {
    format!(
        r#"(version 1)
        (deny default)
        (allow process-exec)
        (allow file-read*)
        (allow file-read* (subpath "/usr"))
        (allow file-read* (subpath "/bin"))
        (allow file-read* (subpath "/opt/homebrew"))
        (allow file-read* (subpath "{cwd}"))
        (allow file-read* (subpath "{home}"))
        {write_rules}
        {network_rules}
        "#,
        cwd = cwd.display(),
        home = dirs::home_dir().unwrap().display(),
        write_rules = build_write_rules(fs_policy, cwd),
        network_rules = build_network_rules(network_policy),
    )
}
```

**特殊保护目录：**

即使在工作区写入模式下，以下目录也被特殊保护：

| 目录 | 保护原因 |
|------|----------|
| `.git/` | Git 版本控制数据，防止破坏版本历史 |
| `.codex/` | Codex 自身的配置和状态数据 |
| `.hg/` | Mercurial 版本控制数据 |
| `.svn/` | Subversion 版本控制数据 |

### 3.5.2 Linux Landlock + Bubblewrap

Linux 沙箱采用**三层架构**，核心实现在 `codex-rs/linux-sandbox/src/landlock.rs`（~300 行）。

```
┌─────────────────────────────────────────────────────────────┐
│                 Linux 三层沙箱架构                            │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 第一层: Landlock (内核 5.13+)                         │   │
│  │  • 文件系统访问控制（只读/读写目录）                    │   │
│  │  • 在当前线程上应用，子进程继承                         │   │
│  │  • 适用于简单策略（read-only, workspace-write）        │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 第二层: seccomp (系统调用过滤)                         │   │
│  │  • 限制可用的系统调用                                  │   │
│  │  • 阻止危险系统调用（如 ptrace、mount）                │   │
│  │  • 与 Landlock 配合使用                               │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 第三层: Bubblewrap (用户命名空间隔离)                   │   │
│  │  • 完整的 Linux 命名空间隔离                           │   │
│  │  • 独立的 mount/IPC/network/PID 命名空间               │   │
│  │  • 适用于复杂策略（需要细粒度控制时自动路由）            │   │
│  │  • codex-linux-sandbox 独立进程                        │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  策略路由:                                                   │
│  简单策略 → Landlock 直接处理                                │
│  复杂策略 → 自动路由到 Bubblewrap                            │
└─────────────────────────────────────────────────────────────┘
```

**apply_sandbox_policy_to_current_thread 代码：**

```rust
// codex-rs/linux-sandbox/src/landlock.rs
pub fn apply_sandbox_policy_to_current_thread(
    fs_policy: &FileSystemSandboxPolicy,
) -> Result<(), SandboxErr> {
    // 1. 检查 Landlock 是否可用
    if !landlock_available() {
        return Err(SandboxErr::LandlockRestrict(
            "Landlock not available on this kernel".into()
        ));
    }

    // 2. 创建 Landlock 规则集
    let mut ruleset = Ruleset::new()
        .handle_access(Access::from_bits(
            AccessFS::READ_FILE | AccessFS::READ_DIR
        ))?;

    // 3. 添加只读目录规则
    for read_dir in &fs_policy.read_roots {
        ruleset.add_rule(PathFd::new(read_dir)?, AccessFS::READ_FILE)?;
    }

    // 4. 添加读写目录规则
    for write_dir in &fs_policy.write_roots {
        ruleset.add_rule(PathFd::new(write_dir)?,
            AccessFS::READ_FILE | AccessFS::WRITE_FILE)?;
    }

    // 5. 限制自身（应用沙箱）
    ruleset.restrict_self()?;

    Ok(())
}
```

**codex-linux-sandbox 独立进程：**

当 Landlock 无法满足复杂策略需求时，系统自动路由到 `codex-linux-sandbox` 独立进程：

```bash
# codex-linux-sandbox 通过 Bubblewrap 创建隔离环境
codex-linux-sandbox \
    --ro-bind /usr /usr \
    --ro-bind /bin /bin \
    --bind /path/to/cwd /path/to/cwd \
    --dev-bind /dev/null /dev/null \
    --unshare-net \
    --seccomp <seccomp-filter> \
    -- <command>
```

### 3.5.3 Windows Restricted Token

Windows 使用 **限制令牌（Restricted Token）** 模式实现沙箱：

```
┌─────────────────────────────────────────────────────────────┐
│                 Windows 沙箱策略                              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Restricted Token 模式                                 │   │
│  │  • 创建限制令牌，移除特权 SID                          │   │
│  │  • 限制文件系统访问权限                                │   │
│  │  • 适用于简单策略                                      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Elevated Runner 模式                                   │   │
│  │  • 提升权限的后端进程                                  │   │
│  │  • 支持更细粒度的文件系统控制                          │   │
│  │  • 通过 IPC 与主进程通信                               │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 3.5.4 沙箱策略注入系统提示词

沙箱策略不仅通过操作系统内核执行，还会注入到系统提示词中，让模型在尝试操作前就知道其约束条件。

**打包的 Markdown 模板：**

| 模板文件 | 说明 |
|----------|------|
| `workspace_write.md` | workspace-write 模式的权限说明 |
| `on_request.md` | on-request 模式的权限说明 |

**`<permissions instructions>` 格式：**

```markdown
<permissions instructions>
## File System Permissions

You have the following file system permissions:
- **Read access**: You can read files in the current working directory and its subdirectories.
- **Write access**: You can create and modify files in the current working directory and its subdirectories.
- **Protected paths**: You CANNOT write to `.git/`, `.codex/`, `.hg/`, `.svn/` directories.

## Network Permissions

- **Network access**: DISABLED. You cannot make network requests.
</permissions instructions>
```

这种设计避免了模型尝试被沙箱阻止的操作，节省 API 调用轮次。

### 3.5.5 网络访问控制

网络访问通过多层实现控制：

```
┌─────────────────────────────────────────────────────────────┐
│                 网络访问控制多层实现                           │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 第一层: Seatbelt 网络规则 (macOS)                     │   │
│  │  (deny network*) 或 (allow network*)                  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 第二层: seccomp 过滤 (Linux)                           │   │
│  │  阻止 socket/connect/bind 系统调用                     │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 第三层: NetworkProxy                                   │   │
│  │  • 应用层网络代理                                      │   │
│  │  • 审计所有网络请求                                    │   │
│  │  • 支持域名白名单/黑名单                               │   │
│  │  • 记录网络访问日志                                    │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  默认行为:                                                   │
│  read-only 模式 → 网络完全禁止                               │
│  workspace-write 模式 → 网络完全禁止                         │
│  danger-full-access 模式 → 网络完全允许                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 3.6 上下文管理（大幅扩充）

### 3.6.1 compact.rs 压缩算法

核心实现在 `codex-rs/core/src/compact.rs`（442 行），负责在对话历史过长时自动压缩早期对话。

**两种压缩模式：**

```rust
/// 控制压缩后是否需要注入初始上下文
#[derive(Clone, Copy, Debug, Eq, PartialEq)]
pub(crate) enum InitialContextInjection {
    /// 在最后一条用户消息之前注入初始上下文
    /// 用于 mid-turn 压缩（模型训练时看到的压缩摘要位置）
    BeforeLastUserMessage,

    /// 不注入初始上下文
    /// 用于 pre-turn/manual 压缩（下一个常规轮次会完整注入初始上下文）
    DoNotInject,
}
```

| 模式 | 触发时机 | 初始上下文注入 | 说明 |
|------|----------|---------------|------|
| **Pre-turn / Manual** | 用户手动触发或轮次开始前 | `DoNotInject` | 替换历史为摘要，清除 `reference_context_item`，下一轮会完整注入 |
| **Mid-turn** | 轮次进行中自动触发 | `BeforeLastUserMessage` | 在摘要前注入初始上下文，保持模型训练时的位置预期 |

**本地压缩流程（4 步）：**

```
┌──────────────────────────────────────────────────────────────┐
│                    本地压缩流程                                │
│                                                               │
│  步骤 1: 构建压缩请求                                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ history = sess.clone_history()                       │   │
│  │ history.record_items(&[initial_input_for_turn])      │   │
│  │ prompt = Prompt {                                    │   │
│  │     input: history.for_prompt(...),                  │   │
│  │     base_instructions: SUMMARIZATION_PROMPT,         │   │
│  │ }                                                   │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                   │
│                          ▼                                   │
│  步骤 2: 调用模型生成摘要                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ drain_to_completed(&sess, turn_context, &prompt)     │   │
│  │ // 流式调用 API，收集模型生成的摘要                    │   │
│  │ // 如果上下文窗口溢出，移除最旧的历史项后重试          │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                   │
│                          ▼                                   │
│  步骤 3: 构建压缩后的历史                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ summary_text = format!("{SUMMARY_PREFIX}\n{summary}")│   │
│  │ user_messages = collect_user_messages(history_items) │   │
│  │ new_history = build_compacted_history(               │   │
│  │     Vec::new(), &user_messages, &summary_text        │   │
│  │ )                                                   │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                   │
│                          ▼                                   │
│  步骤 4: 替换历史并发送事件                                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ sess.replace_compacted_history(                      │   │
│  │     new_history, reference_context_item,             │   │
│  │     compacted_item                                  │   │
│  │ )                                                   │   │
│  │ // 发送 CompactedItem 事件                           │   │
│  │ // 发送 WarningEvent（建议新开线程）                   │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

**远程压缩判断：**

```rust
/// 判断是否应使用远程压缩（云端 API）
pub(crate) fn should_use_remote_compact_task(provider: &ModelProviderInfo) -> bool {
    provider.is_openai()  // 仅 OpenAI 提供商使用远程压缩
}
```

### 3.6.2 摘要提示词模板

压缩使用专门的摘要提示词模板：

```rust
pub const SUMMARIZATION_PROMPT: &str = include_str!("../templates/compact/prompt.md");
pub const SUMMARY_PREFIX: &str = include_str!("../templates/compact/summary_prefix.md");
```

**摘要文本格式：**

```
{SUMMARY_PREFIX}

{模型生成的摘要内容}
```

摘要内容包含：
- 用户的原始请求
- 已完成的工作摘要
- 重要的决策和推理过程
- 待完成的工作（如果有）

### 3.6.3 Token 感知截断

核心实现在 `codex-rs/core/src/truncate.rs`（363 行），提供基于字节估算的 token 截断功能。

**关键常量：**

```rust
/// 每个 token 的近似字节数（用于 UTF-8 文本的粗略估算）
pub const APPROX_BYTES_PER_TOKEN: usize = 4;

/// 压缩后用户消息的最大 token 数
pub const COMPACT_USER_MESSAGE_MAX_TOKENS: usize = 20_000;
```

**TruncationPolicy 枚举：**

```rust
pub enum TruncationPolicy {
    /// 不截断
    NoTruncation,

    /// 按字节估算截断到指定 token 数
    TokenLimit {
        max_tokens: usize,
    },

    /// 按行数截断
    LineLimit {
        max_lines: usize,
    },
}
```

**truncate_with_byte_estimate 算法：**

```rust
/// 基于字节估算的截断算法
///
/// 策略：50/50 分配前缀和后缀
/// - 前缀保留开头内容（提供上下文）
/// - 后缀保留结尾内容（提供最新信息）
/// - 在 UTF-8 字符边界处安全切割
pub fn truncate_with_byte_estimate(
    text: &str,
    max_tokens: usize,
) -> String {
    let max_bytes = max_tokens * APPROX_BYTES_PER_TOKEN;

    if text.len() <= max_bytes {
        return text.to_string();
    }

    let half = max_bytes / 2;

    // 找到前缀的安全 UTF-8 切割点
    let prefix_end = find_safe_utf8_boundary(text, half);

    // 找到后缀的安全 UTF-8 切割点
    let suffix_start = find_safe_utf8_boundary_from_end(text, half);

    format!(
        "{}\n\n... [truncated {} bytes] ...\n\n{}",
        &text[..prefix_end],
        text.len() - max_bytes,
        &text[suffix_start..]
    )
}

/// 近似 token 计数
pub fn approx_token_count(text: &str) -> usize {
    (text.len() + APPROX_BYTES_PER_TOKEN - 1) / APPROX_BYTES_PER_TOKEN
}
```

**build_compacted_history_with_limit 重建逻辑：**

```rust
/// 构建压缩后的历史，确保总 token 数不超过限制
pub fn build_compacted_history_with_limit(
    user_messages: &[String],
    summary_text: &str,
    max_tokens: usize,
) -> Vec<ResponseItem> {
    let mut history = Vec::new();

    // 1. 添加摘要
    history.push(ResponseItem::from(summary_text));

    // 2. 从最新消息开始，向前添加用户消息
    let mut remaining_tokens = max_tokens
        .saturating_sub(approx_token_count(summary_text));

    for msg in user_messages.iter().rev() {
        let msg_tokens = approx_token_count(msg);
        if msg_tokens > remaining_tokens {
            // 截断消息以适应剩余预算
            let truncated = truncate_with_byte_estimate(
                msg,
                remaining_tokens
            );
            history.push(ResponseItem::from(truncated));
            break;
        }
        remaining_tokens -= msg_tokens;
        history.push(ResponseItem::from(msg.clone()));
    }

    // 3. 反转以保持时间顺序
    history.reverse();
    history
}
```

### 3.6.4 上下文窗口超时处理

当对话历史超过模型上下文窗口时，系统采用移除最旧历史项的重试策略：

```rust
// compact.rs 中的上下文窗口超时处理
Err(e @ CodexErr::ContextWindowExceeded) => {
    if turn_input_len > 1 {
        // 从开头移除最旧的历史项
        // 保留缓存（基于前缀），保持最近消息完整
        error!(
            "Context window exceeded while compacting; \
             removing oldest history item. Error: {e}"
        );
        history.remove_first_item();
        truncated_count += 1;
        retries = 0;
        continue;  // 重试
    }
    // 如果只剩一条消息仍然超限，报告错误
    sess.set_total_tokens_full(turn_context.as_ref()).await;
    let event = EventMsg::Error(e.to_error_event(None));
    sess.send_event(&turn_context, event).await;
    return Err(e);
}
```

---

## 3.7 Guardian 安全守护系统

### 3.7.1 审批策略

Guardian 系统通过 `AskForApproval` 枚举定义四级审批策略：

```rust
#[derive(Debug, Clone, Deserialize, Serialize, PartialEq, JsonSchema)]
pub enum AskForApproval {
    /// 不信任模式 — 所有命令都需要用户审批
    Untrusted,

    /// 失败时审批 — 仅当命令失败时需要审批
    OnFailure,

    /// 按需审批 — 模型认为需要时请求审批
    OnRequest,

    /// 从不审批 — 所有命令自动执行（危险）
    Never,
}
```

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| `untrusted` | 所有命令都需要用户确认 | 安全敏感环境 |
| `on-failure` | 命令失败时才需要确认 | 开发环境 |
| `on-request` | 模型自行判断是否需要确认 | 日常使用 |
| `never` | 所有命令自动执行 | CI/CD、容器环境 |

### 3.7.2 canAutoApprove 判断逻辑

Guardian 的核心判断函数 `canAutoApprove` 维护一个已知安全命令的白名单：

```rust
fn is_known_safe_command(command: &str) -> bool {
    // 完全匹配的安全命令
    const SAFE_COMMANDS: &[&str] = &[
        "cat", "cd", "echo", "grep", "ls", "pwd", "wc",
    ];

    // Git 安全子命令
    const SAFE_GIT_COMMANDS: &[&str] = &[
        "git status", "git log", "git diff", "git show",
        "git branch", "git stash list",
    ];

    // 受限的 find 命令（仅允许 -name、-type 参数）
    // 受限的 ripgrep 命令（仅允许基本搜索参数）

    // 复合命令解析（bash -lc "..."）
    if command.starts_with("bash -lc") {
        // 解析内层命令并递归检查
    }

    // ... 白名单匹配逻辑
}
```

**复合命令解析：**

```rust
/// 解析复合命令，提取实际执行的命令
fn parse_command(command: &str) -> ParsedCommand {
    // 处理 bash -lc "..." 格式
    if command.starts_with("bash -lc") {
        if let Some(inner) = extract_quoted_arg(command) {
            return parse_command(inner);
        }
    }

    // 处理管道命令 — 取第一个命令
    if let Some(first) = command.split('|').next() {
        return ParsedCommand {
            primary: first.trim().to_string(),
            has_pipe: true,
        };
    }

    ParsedCommand {
        primary: command.to_string(),
        has_pipe: false,
    }
}
```

### 3.7.3 用户确认 UI

当命令需要用户审批时，TUI 层通过 `BottomPane` 的 `view_stack` 显示审批覆盖层：

```
┌──────────────────────────────────────────────────────────────┐
│  TUI 主界面                                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 聊天消息区域                                            │  │
│  │ ...                                                    │  │
│  └────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ BottomPane (view_stack)                                │  │
│  │ ┌──────────────────────────────────────────────────┐  │  │
│  │ │ ApprovalOverlay                                  │  │  │
│  │ │                                                   │  │  │
│  │ │  ⚠ 命令需要审批                                   │  │  │
│  │ │                                                   │  │  │
│  │ │  命令: rm -rf /tmp/build                          │  │  │
│  │ │  工作目录: /home/user/project                      │  │  │
│  │ │                                                   │  │  │
│  │ │  [y] 允许  [n] 拒绝  [a] 始终允许此类命令          │  │  │
│  │ └──────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

用户确认后，通过 `Op::ExecApproval` 发送审批决策：

```rust
Op::ExecApproval {
    id: "approval_request_id".to_string(),
    approved: true,  // 或 false
}
```

---

## 3.8 多 Agent 系统

Codex CLI 支持完整的多 Agent 编排系统，每个 Agent 运行在独立的沙箱容器中。

**核心原语：**

| 原语 | 说明 |
|------|------|
| `spawn` | 创建新的 Agent 实例 |
| `resume` | 恢复已暂停的 Agent |
| `wait` | 等待 Agent 完成 |
| `close` | 终止 Agent |
| `send-message` | 向 Agent 发送消息 |
| `list` | 列出所有 Agent |
| `assign-task` | 向 Agent 分配任务 |

**批量 Agent 任务（spawn_agents_on_csv）：**

```rust
/// 从 CSV 文件批量创建 Agent 任务
/// 每个 CSV 行定义一个独立的 Agent 任务
/// 所有 Agent 并行运行，各自拥有独立的沙箱环境
async fn spawn_agents_on_csv(
    csv_path: &Path,
    task_template: &str,
    sandbox_policy: SandboxPolicy,
) -> Vec<AgentHandle> {
    let tasks = parse_csv(csv_path).await?;
    let mut handles = Vec::new();

    for task in tasks {
        let handle = spawn_agent(
            task.prompt,
            task.cwd,
            sandbox_policy.clone(),
        ).await?;
        handles.push(handle);
    }

    handles
}
```

**Guardian 守护者模式：**

Guardian 作为安全中间层，决定哪些操作可以自动执行：

```
┌──────────────────────────────────────────────────────────────┐
│                 Guardian 守护者模式                            │
│                                                               │
│  Agent 请求                                                   │
│      │                                                        │
│      ▼                                                        │
│  ┌──────────────┐                                            │
│  │ Guardian     │                                            │
│  │ 评估请求      │                                            │
│  │              │                                            │
│  │ 1. 检查命令白名单                                           │
│  │ 2. 评估风险等级                                             │
│  │ 3. 检查审批策略                                             │
│  └──────┬───────┘                                            │
│         │                                                     │
│    ┌────┴────┐                                               │
│    ▼         ▼                                                │
│  自动执行   请求用户审批                                       │
│    │         │                                                │
│    ▼         ▼                                                │
│  沙箱执行   ApprovalOverlay                                   │
│              │                                                │
│              ▼                                                │
│         用户确认/拒绝                                          │
└──────────────────────────────────────────────────────────────┘
```

---

## 3.9 配置系统

### 3.9.1 配置文件发现与加载

Codex CLI 使用 **5 级优先级** 配置系统：

```
优先级从高到低:
┌──────────────────────────────────────────────────────────────┐
│  1. 环境变量 (CODEX_*)                                       │
│     CODEX_MODEL=gpt-4o                                       │
│     CODEX_SANDBOX=workspace-write                             │
│     OPENAI_API_KEY=sk-...                                    │
│                                                               │
│  2. CLI 标志 (--model, --sandbox, --approval-policy)          │
│     codex --model gpt-4o --sandbox workspace-write            │
│                                                               │
│  3. Profile (codex --profile <name>)                         │
│     ~/.codex/profiles/<name>.toml                             │
│                                                               │
│  4. 全局 config.toml                                          │
│     ~/.codex/config.toml                                      │
│                                                               │
│  5. 内置默认值                                                │
│     model = "o4-mini"                                         │
│     sandbox_mode = "read-only"                                │
│     approval_policy = "on-request"                            │
└──────────────────────────────────────────────────────────────┘
```

**ConfigLayerStack 合并：**

```rust
/// 配置层栈 — 按优先级从低到高合并
pub struct ConfigLayerStack {
    layers: Vec<ConfigLayer>,
}

pub struct ConfigLayer {
    pub name: ConfigLayerSource,
    pub config: TomlValue,
    pub is_disabled: bool,
}

pub enum ConfigLayerSource {
    /// 内置默认值
    Builtin,
    /// 全局配置文件
    Global { path: PathBuf },
    /// 项目配置文件
    Project { path: PathBuf },
    /// 环境变量
    Environment,
    /// CLI 标志
    CliFlags,
    /// Profile
    Profile { name: String },
}
```

### 3.9.2 AGENTS.md / codex.md 发现机制

核心实现在 `codex-rs/core/src/project_doc.rs`（315 行）。

**发现算法（4 步）：**

```
步骤 1: 确定项目根目录
┌──────────────────────────────────────────────────────────────┐
│  从当前工作目录向上遍历，查找 project_root_markers            │
│  默认标记: [".git"]                                          │
│  可配置: project_root_markers = [".git", "package.json"]     │
│                                                               │
│  /home/user/project/src/  ← cwd                              │
│  /home/user/project/      ← 找到 .git，确定为项目根          │
│  /home/user/                                                   │
│  /home/                                                       │
│  /                                                            │
└──────────────────────────────────────────────────────────────┘

步骤 2: 从项目根到 cwd 收集 AGENTS.md
┌──────────────────────────────────────────────────────────────┐
│  搜索路径（从根到 cwd）:                                      │
│  /home/user/project/AGENTS.md          ← 项目级指令          │
│  /home/user/project/src/AGENTS.md      ← 子目录指令          │
│                                                               │
│  候选文件名（按优先级）:                                       │
│  1. AGENTS.override.md  (本地覆盖，最高优先级)                │
│  2. AGENTS.md           (标准文件名)                          │
│  3. codex.md            (备用文件名，可配置)                   │
└──────────────────────────────────────────────────────────────┘

步骤 3: 读取并拼接内容
┌──────────────────────────────────────────────────────────────┐
│  大小限制: 32 KiB (project_doc_max_bytes)                     │
│  如果总大小超过限制，从最早发现的文件开始截断                   │
│  多个文件之间用 "\n\n" 分隔拼接                                │
└──────────────────────────────────────────────────────────────┘

步骤 4: 组装用户指令
┌──────────────────────────────────────────────────────────────┐
│  最终指令 = Config::instructions                              │
│            + "\n\n--- project-doc ---\n\n"                    │
│            + AGENTS.md 内容                                    │
│            + JS REPL 指令（如果启用）                           │
│            + 子 Agent 指令（如果启用）                          │
└──────────────────────────────────────────────────────────────┘
```

### 3.9.3 配置文件格式

完整的 TOML 配置文件示例：

```toml
# ~/.codex/config.toml

# 模型配置
model = "o4-mini"
model_provider = "openai"

# 沙箱模式
sandbox_mode = "workspace-write"

# 审批策略
approval_policy = "on-request"

# 模型提供商配置
[model_provider]
name = "openai"

# MCP 服务器配置
[mcp_servers]
my-server = { command = "npx", args = ["-y", "@my/mcp-server"] }
another-server = { command = "python", args = ["-m", "my_mcp"] }

# 项目文档配置
project_doc_max_bytes = 32768
project_doc_fallback_filenames = ["codex.md", "CODEX.md"]
project_root_markers = [".git", "package.json", "Cargo.toml"]

# 需求配置
[requirements]
# 定义项目特定的约束条件

# OpenTelemetry 配置
[otel]
enabled = true
endpoint = "http://localhost:4318"
```

### 3.9.4 模型路由

**ModelProviderInfo 注册表：**

```rust
/// 模型提供商信息
pub struct ModelProviderInfo {
    /// 提供商标识符
    pub id: String,

    /// 提供商名称
    pub name: String,

    /// 基础指令模板
    pub base_instructions: BaseInstructions,

    /// 支持的模型列表
    pub models: Vec<ModelInfo>,

    /// 是否为 OpenAI 提供商
    pub is_openai: bool,

    /// 流式请求最大重试次数
    pub stream_max_retries: usize,
}
```

**认证方式路由：**

| 认证方式 | 说明 | 配置 |
|----------|------|------|
| **ChatGPT OAuth** | 通过浏览器登录 ChatGPT 账户 | `codex login` |
| **API Key** | 直接使用 OpenAI API Key | `codex login --api-key` 或 `OPENAI_API_KEY` |
| **本地模型** | Ollama / LM Studio | `model_provider = "ollama"` |
| **Azure** | Azure OpenAI Service | `model_provider = "azure"` |

---

## 3.10 状态与会话管理

### 3.10.1 双层存储架构

Codex CLI 使用**双层存储架构**持久化会话状态：

```
┌──────────────────────────────────────────────────────────────┐
│                 双层存储架构                                   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 第一层: JSONL Rollout 文件                            │   │
│  │  ~/.codex/sessions/YYYY/MM/DD/                       │   │
│  │  rollout-2025-05-07T17-24-21-{uuid}.jsonl            │   │
│  │                                                       │   │
│  │  • 完整的会话事件流                                   │   │
│  │  • 可用 jq/fx 等工具直接查看                          │   │
│  │  • 支持会话恢复和分叉                                 │   │
│  │  • 人类可读的 JSON Lines 格式                         │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 第二层: SQLite 数据库                                 │   │
│  │  ~/.codex/state.db                                    │   │
│  │                                                       │   │
│  │  • 线程元数据索引                                     │   │
│  │  • 快速搜索和过滤                                     │   │
│  │  • 线程列表分页                                       │   │
│  │  • 从 Rollout 文件 read-repair                        │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### 3.10.2 Rollout 文件格式

**RolloutLine 格式：**

每行是一个 JSON 对象，包含时间戳和事件类型：

```json
{"timestamp":"2025-05-07T17:24:21.123Z","item":{"SessionMeta":{...}}}
{"timestamp":"2025-05-07T17:24:22.456Z","item":{"ResponseItem":{...}}}
{"timestamp":"2025-05-07T17:24:23.789Z","item":{"EventMsg":{...}}}
{"timestamp":"2025-05-07T17:24:24.012Z","item":{"TurnContext":{...}}}
{"timestamp":"2025-05-07T17:24:25.345Z","item":{"Compacted":{...}}}
```

**RolloutItem 类型表：**

| 类型 | 说明 | 包含内容 |
|------|------|----------|
| `ResponseItem` | 模型响应项 | 文本消息、工具调用、工具结果 |
| `EventMsg` | 事件消息 | 执行命令开始/结束、审批请求等 |
| `TurnContext` | 轮次上下文 | 模型信息、沙箱策略、审批策略等 |
| `Compacted` | 压缩记录 | 压缩摘要、替换历史 |
| `SessionMeta` | 会话元数据 | 会话 ID、时间戳、cwd、Git 信息等 |

### 3.10.3 RolloutRecorder

核心实现在 `codex-rs/rollout/src/recorder.rs`（1111 行）。

**异步写入架构：**

```rust
/// Rollout 记录器 — 异步写入会话事件
#[derive(Clone)]
pub struct RolloutRecorder {
    /// 命令发送通道（有界队列，容量 256）
    tx: Sender<RolloutCmd>,

    /// 后台写入任务状态
    writer_task: Arc<RolloutWriterTask>,

    /// Rollout 文件路径
    pub(crate) rollout_path: PathBuf,

    /// SQLite 数据库句柄
    state_db: Option<StateDbHandle>,

    /// 事件持久化模式
    event_persistence_mode: EventPersistenceMode,
}

/// 写入命令
enum RolloutCmd {
    /// 添加事件项
    AddItems(Vec<RolloutItem>),

    /// 持久化（创建文件并写入所有缓冲项）
    Persist { ack: oneshot::Sender<std::io::Result<()>> },

    /// 刷新（将缓冲项写入已打开的文件）
    Flush { ack: oneshot::Sender<std::io::Result<()>> },

    /// 关闭（写入所有缓冲项后停止）
    Shutdown { ack: oneshot::Sender<std::io::Result<()>> },
}
```

**线程发现（按 UUID/按名称/已归档）：**

```rust
impl RolloutRecorder {
    /// 列出线程（支持分页、排序、过滤）
    pub async fn list_threads(
        config: &impl RolloutConfigView,
        page_size: usize,
        cursor: Option<&Cursor>,
        sort_key: ThreadSortKey,
        allowed_sources: &[SessionSource],
        model_providers: Option<&[String]>,
        default_provider: &str,
        search_term: Option<&str>,
    ) -> std::io::Result<ThreadsPage>

    /// 列出已归档线程
    pub async fn list_archived_threads(...) -> std::io::Result<ThreadsPage>

    /// 查找最新的线程路径（可选按 cwd 过滤）
    pub async fn find_latest_thread_path(...) -> std::io::Result<Option<PathBuf>>
}
```

### 3.10.4 会话恢复与分叉

**恢复流程：**

```
┌──────────────────────────────────────────────────────────────┐
│                 会话恢复流程                                   │
│                                                               │
│  1. 加载 Rollout 文件                                         │
│     load_rollout_items(path)                                  │
│     ├── 读取 JSONL 文件                                       │
│     ├── 逐行解析 RolloutLine                                  │
│     ├── 提取 SessionMeta 获取 thread_id                       │
│     └── 收集所有 RolloutItem                                  │
│                                                               │
│  2. 重放 RolloutItem 流                                       │
│     ├── SessionMeta → 恢复会话元数据                          │
│     ├── ResponseItem → 重建消息历史                           │
│     ├── TurnContext → 恢复轮次上下文                           │
│     ├── Compacted → 恢复压缩历史                              │
│     └── EventMsg → 重建事件状态                               │
│                                                               │
│  3. 创建新的 Codex 实例                                       │
│     ├── 使用恢复的历史初始化 Session                          │
│     ├── RolloutRecorder 以 Resume 模式打开（追加写入）        │
│     └── 继续正常的 SQ/EQ 事件循环                             │
└──────────────────────────────────────────────────────────────┘
```

**分叉流程：**

```
┌──────────────────────────────────────────────────────────────┐
│                 会话分叉流程                                   │
│                                                               │
│  1. 加载原始 Rollout 文件                                     │
│     load_rollout_items(original_path)                         │
│                                                               │
│  2. 创建新的线程 ID                                           │
│     new_thread_id = Uuid::new_v4()                            │
│                                                               │
│  3. 继承配置                                                 │
│     ├── 模型配置                                             │
│     ├── 沙箱策略                                             │
│     ├── 审批策略                                             │
│     └── 工具配置                                             │
│                                                               │
│  4. 创建新的 Rollout 文件                                     │
│     RolloutRecorder::new(                                    │
│         RolloutRecorderParams::Create {                      │
│             conversation_id: new_thread_id,                  │
│             forked_from_id: Some(original_thread_id),        │
│             source: SessionSource::Fork,                     │
│             ...                                              │
│         }                                                    │
│     )                                                        │
│                                                               │
│  5. 不修改原始 Rollout 文件                                   │
│     原始会话保持不变，可以继续使用                             │
└──────────────────────────────────────────────────────────────┘
```

**TUI 恢复选择器：**

```
┌──────────────────────────────────────────────────────────────┐
│  会话恢复选择器                                               │
│                                                               │
│  最近会话 (1/3):                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ #1  [2025-05-07 17:24]  "重构认证模块"                  │  │
│  │     /home/user/project  main  o4-mini                  │  │
│  │                                                        │  │
│  │ #2  [2025-05-07 15:10]  "修复 CI 管道"                  │  │
│  │     /home/user/project  fix-ci  o4-mini                │  │
│  │                                                        │  │
│  │ #3  [2025-05-06 09:30]  "添加单元测试"                  │  │
│  │     /home/user/project  main  gpt-4o                   │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
│  分页: 每页 25 条 | 过滤: 支持搜索 | 排序: 按时间/按名称     │
│                                                               │
│  [↑↓] 选择  [Enter] 恢复  [/] 搜索  [q] 退出                 │
└──────────────────────────────────────────────────────────────┘
```

### 3.10.5 记忆系统

```
~/.codex/
├── sessions/                  # 会话 Rollout 文件
│   ├── 2025/
│   │   ├── 05/
│   │   │   ├── 07/
│   │   │   │   └── rollout-2025-05-07T17-24-21-{uuid}.jsonl
│   │   │   └── ...
│   │   └── ...
│   └── archived/              # 已归档会话
│       └── ...
├── state.db                   # SQLite 状态数据库
├── config.toml                # 全局配置
├── profiles/                  # 配置 Profile
│   └── dev.toml
└── memories/                  # 跨会话记忆
    └── ...
```

记忆系统通过 `memories` 模块实现跨会话知识保持。当 `generate_memories` 配置启用时，系统会在会话结束时自动提取关键信息并持久化，在后续会话中作为上下文注入。

---

## 3.11 MCP 集成（大幅扩充）

### 3.11.1 codex-rmcp-client

MCP 客户端通过 `codex-rmcp-client` crate 实现，核心是 `McpConnectionManager`：

```rust
/// MCP 连接管理器 — 管理所有 MCP 服务器连接
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

**config.toml 配置方式：**

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

### 3.11.2 MCP 工具暴露控制

MCP 工具通过 `McpToolSnapshot` 暴露给模型：

```rust
/// MCP 工具快照 — 某个时刻的 MCP 工具列表
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

**重要设计 — MCP 工具不受沙箱保护：**

MCP 工具在 MCP 服务器进程中执行，**不受 Codex CLI 沙箱策略的约束**。这意味着：

- MCP 工具可以访问文件系统的任意位置
- MCP 工具可以发起网络请求
- 安全性依赖于 MCP 服务器自身的安全实现

这是有意的设计选择，因为 MCP 工具通常需要访问外部资源（如数据库、API），沙箱限制会使其无法正常工作。

### 3.11.3 MCP Server 模式

Codex CLI 自身也可以作为 MCP 服务器运行：

```bash
# 启动 Codex CLI 作为 MCP 服务器
codex mcp-server
```

**RmcpServer 实现：**

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

/// run_codex 工具 — 允许外部 MCP 客户端通过 Codex 执行任务
pub struct RunCodexTool {
    pub prompt: String,
    pub cwd: Option<String>,
    pub model: Option<String>,
}
```

---

## 3.12 错误处理与重试

### 3.12.1 错误类型体系

核心实现在 `codex-rs/core/src/error.rs`（659 行），定义了完整的错误类型枚举：

```rust
#[derive(Debug, Clone, thiserror::Error)]
pub enum CodexErr {
    /// 轮次被中止（用户中断）
    #[error("Turn aborted")]
    TurnAborted,

    /// 流式响应错误
    #[error("Stream error: {0}")]
    Stream(String),

    /// 上下文窗口超限
    #[error("Context window exceeded")]
    ContextWindowExceeded,

    /// 请求超时
    #[error("Request timeout")]
    Timeout,

    /// 操作被中断
    #[error("Operation interrupted")]
    Interrupted,

    /// 意外的 HTTP 状态码
    #[error("Unexpected status: {0}")]
    UnexpectedStatus(u16),

    /// 无效请求
    #[error("Invalid request: {0}")]
    InvalidRequest(String),

    /// 使用限制达到
    #[error("Usage limit reached")]
    UsageLimitReached,

    /// 服务器过载
    #[error("Server overloaded")]
    ServerOverloaded,

    /// 配额超限
    #[error("Quota exceeded")]
    QuotaExceeded,

    /// 重试次数超限
    #[error("Retry limit reached")]
    RetryLimit,

    /// 内部服务器错误
    #[error("Internal server error: {0}")]
    InternalServerError(String),

    /// 沙箱错误
    #[error("Sandbox error: {0}")]
    Sandbox(#[from] SandboxErr),

    // ... 更多变体
}
```

### 3.12.2 可重试错误判断

```rust
impl CodexErr {
    /// 判断错误是否可重试
    pub fn is_retryable(&self) -> bool {
        match self {
            CodexErr::Stream(_) => true,
            CodexErr::Timeout => true,
            CodexErr::ServerOverloaded => true,
            CodexErr::InternalServerError(_) => true,
            CodexErr::UnexpectedStatus(429) => true,  // Rate limit
            CodexErr::UnexpectedStatus(502) => true,  // Bad Gateway
            CodexErr::UnexpectedStatus(503) => true,  // Service Unavailable
            _ => false,
        }
    }
}
```

### 3.12.3 重试策略

**指数退避 backoff() 函数：**

```rust
/// 指数退避计算
/// 重试 1: ~1s, 重试 2: ~2s, 重试 3: ~4s, ...
pub fn backoff(retry_count: usize) -> Duration {
    let base_ms = 1000u64;
    let max_ms = 30_000u64;  // 最大 30 秒
    let delay_ms = base_ms * 2u64.saturating_pow(retry_count as u32 - 1);
    Duration::from_millis(delay_ms.min(max_ms))
}
```

**Stream 错误特殊处理：**

```rust
// Stream 错误返回 Option<Duration> 表示是否应该重试以及等待时间
fn handle_stream_error(err: &CodexErr) -> Option<Duration> {
    match err {
        CodexErr::Stream(msg) if msg.contains("connection reset") => {
            Some(Duration::from_secs(1))  // 立即重试
        }
        CodexErr::Stream(msg) if msg.contains("timeout") => {
            Some(Duration::from_secs(2))  // 等待后重试
        }
        CodexErr::ServerOverloaded => {
            Some(backoff(1))  // 指数退避
        }
        _ => None,  // 不可重试
    }
}
```

**ContextWindowExceeded 移除最旧历史项重试：**

```rust
// 在压缩过程中，如果上下文窗口仍然超限
Err(e @ CodexErr::ContextWindowExceeded) => {
    if turn_input_len > 1 {
        // 移除最旧的历史项（保留前缀缓存）
        history.remove_first_item();
        truncated_count += 1;
        continue;  // 重试
    }
    // 无法继续缩减，报告错误
}
```

**使用限制错误的升级建议：**

```rust
CodexErr::UsageLimitReached => {
    // 发送升级建议事件
    sess.send_event(&turn_context, EventMsg::Error(ErrorEvent {
        message: "API usage limit reached. Consider upgrading your plan or waiting for the limit to reset.".to_string(),
        codex_error_info: Some(CodexErrorInfo::UsageLimitReached),
    })).await;
}
```

### 3.12.4 沙箱执行失败处理

```rust
#[derive(Debug, Clone, thiserror::Error)]
pub enum SandboxErr {
    /// 操作被沙箱拒绝
    #[error("Operation denied by sandbox: {0}")]
    Denied(String),

    /// seccomp 过滤器安装失败
    #[error("Failed to install seccomp filter: {0}")]
    SeccompInstall(String),

    /// 命令执行超时
    #[error("Command timed out after {0:?}")]
    Timeout(Duration),

    /// 命令被信号终止
    #[error("Command terminated by signal: {0}")]
    Signal(i32),

    /// Landlock 限制应用失败
    #[error("Failed to apply Landlock restrictions: {0}")]
    LandlockRestrict(String),
}
```

---

## 3.13 构建系统

| 工具 | 用途 |
|------|------|
| **Bazel 9** | 主力构建系统，CI/CD，跨平台编译 |
| **Cargo** | Rust 包管理和开发构建 |
| **Nix** | 可复现构建（flake.nix） |
| **just** | 任务运行器 |
| **pnpm** | npm 生态包管理（monorepo） |

**双构建系统策略**：Bazel 用于生产级 CI/CD 和跨平台编译，Cargo 用于日常开发。

```
┌──────────────────────────────────────────────────────────────┐
│                 双构建系统策略                                 │
│                                                               │
│  开发环境:                                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ cargo build                                           │   │
│  │ cargo test                                            │   │
│  │ cargo run --bin codex -- tui                          │   │
│  │ cargo clippy                                          │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  CI/CD:                                                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ bazel build //codex-rs:cli                            │   │
│  │ bazel test //codex-rs/...                             │   │
│  │ bazel run //codex-rs:cli -- --release                 │   │
│  │ 跨平台: Linux x86_64, macOS ARM64, Windows x86_64     │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  可复现构建:                                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ nix build                                             │   │
│  │ nix develop                                           │   │
│  │ 确保所有依赖版本锁定                                   │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## 3.14 遥测与可观测性

### 3.14.1 内置 OpenTelemetry SDK

Codex CLI 内置了 OpenTelemetry SDK，通过 `codex-otel` crate 实现：

```toml
# config.toml 中的 OTel 配置
[otel]
enabled = true
endpoint = "http://localhost:4318"  # OTLP exporter endpoint
```

### 3.14.2 5 种核心指标

| 指标名称 | 类型 | 说明 |
|----------|------|------|
| `feature.state` | Gauge | 特性标志状态 |
| `approval.requested` | Counter | 审批请求次数 |
| `tool.call` | Counter | 工具调用次数 |
| `conversation.turn.count` | Counter | 对话轮次计数 |
| `shell_snapshot` | Histogram | Shell 命令执行快照 |

### 3.14.3 AnalyticsEventsClient

```rust
/// 分析事件客户端
pub struct AnalyticsEventsClient {
    /// 事件发送端点
    endpoint: String,

    /// 安装 ID
    installation_id: String,
}

impl AnalyticsEventsClient {
    /// 发送分析事件
    pub async fn track_event(&self, event: AnalyticsEvent) {
        // POST /codex/analytics-events/events
    }

    /// 跟踪压缩事件
    pub async fn track_compaction(&self, event: CodexCompactionEvent) {
        // 记录压缩触发原因、持续时间、token 变化等
    }
}
```

**插件遥测事件：**

```rust
// 插件安装/禁用事件
AnalyticsEvent::PluginInstalled { plugin_id, version }
AnalyticsEvent::PluginDisabled { plugin_id, reason }
```

### 3.14.4 隐私保护

- **默认不记录 prompt**：系统提示和用户输入不会被发送到遥测端点
- **LLM_CLI_TELEMETRY_DISABLED**：设置此环境变量可完全禁用遥测
- **本地优先**：所有会话数据存储在本地 `~/.codex/` 目录

---

## 3.15 认证与授权

### 3.15.1 ChatGPT 账户登录

Codex CLI 支持通过 ChatGPT 账户登录，提供三种认证流程：

**浏览器环境（localhost:1455）：**

```
┌──────────────────────────────────────────────────────────────┐
│  ChatGPT OAuth 流程（浏览器环境）                             │
│                                                               │
│  1. codex login                                              │
│     │                                                        │
│     ▼                                                        │
│  2. 启动本地 HTTP 服务器 (localhost:1455)                     │
│     │                                                        │
│     ▼                                                        │
│  3. 打开默认浏览器，跳转到 ChatGPT 授权页面                    │
│     │                                                        │
│     ▼                                                        │
│  4. 用户在浏览器中登录并授权                                   │
│     │                                                        │
│     ▼                                                        │
│  5. ChatGPT 重定向到 localhost:1455/callback?code=...        │
│     │                                                        │
│     ▼                                                        │
│  6. 本地服务器接收授权码，交换 access_token                    │
│     │                                                        │
│     ▼                                                        │
│  7. 存储 token 到 keyring                                     │
└──────────────────────────────────────────────────────────────┘
```

**无浏览器环境（设备码认证流程 3 步）：**

```
1. codex login
   → 显示设备码: XXXX-XXXX
   → 显示验证 URL: https://chatgpt.com/device

2. 用户在其他设备上打开 URL，输入设备码

3. codex login 轮询等待授权完成
   → 授权成功，存储 token
```

**远程服务器（SSH 端口转发）：**

```bash
# 在远程服务器上
codex login

# 在本地机器上建立端口转发
ssh -L 1455:localhost:1455 user@remote-server

# 然后在本地浏览器中完成授权
```

### 3.15.2 API Key 登录

```bash
# 方式 1: 通过命令行
codex login --api-key sk-...

# 方式 2: 通过环境变量
export OPENAI_API_KEY=sk-...
```

### 3.15.3 keyring-store 密钥管理

密钥管理通过 `codex-keyring-store` crate 实现，支持 4 个平台：

| 平台 | 后端 | 说明 |
|------|------|------|
| **macOS** | Keychain | 系统密钥链 |
| **Windows** | Credential Manager | Windows 凭据管理器 |
| **Linux** | DBus + keyutils | 通过 DBus 协议访问系统密钥环 |
| **FreeBSD** | DBus | 通过 DBus 协议 |

**三种存储模式：**

```rust
pub enum OAuthCredentialsStoreMode {
    /// 自动选择（优先 Keyring，回退到文件）
    Auto,

    /// 仅使用 Keyring
    Keyring,

    /// 仅使用文件存储
    File,
}
```

**安全设计：**

```rust
/// 密钥服务名称
const KEYRING_SERVICE: &str = "codex-cli";

/// 使用 SHA-256 哈希作为存储键
fn hash_key(key: &str) -> String {
    use sha2::{Sha256, Digest};
    let mut hasher = Sha256::new();
    hasher.update(key.as_bytes());
    format!("{:x}", hasher.finalize())
}

/// 刷新令牌时间偏移（30 秒）
const REFRESH_SKEW_MILLIS: u64 = 30_000;
```

---

## 3.16 IDE 集成

### 3.16.1 app-server 协议

Codex CLI 通过 `app-server` 协议与 IDE 集成，使用 **JSON-RPC 2.0** 双向通信。

**传输层：**

| 传输方式 | 说明 | 状态 |
|----------|------|------|
| **stdio** | 通过标准输入/输出通信（默认） | 稳定 |
| **websocket** | 通过 WebSocket 通信 | 实验性 |

**核心原语：**

| 原语 | 说明 |
|------|------|
| **Thread** | 对话线程（对应一个会话） |
| **Turn** | 对话轮次（一次用户输入到模型完成响应） |
| **Item** | 轮次中的项目（消息、工具调用、工具结果） |

**关键 API 列表：**

| API | 方法 | 说明 |
|-----|------|------|
| `thread/start` | POST | 创建新对话线程 |
| `thread/resume` | POST | 恢复已有线程 |
| `thread/fork` | POST | 分叉线程 |
| `turn/start` | POST | 开始新轮次 |
| `turn/steer` | POST | 引导当前轮次 |
| `turn/interrupt` | POST | 中断当前轮次 |
| `plugin/*` | POST | 插件管理 |
| `config/*` | POST | 配置管理 |
| `review/start` | POST | 开始代码审查 |
| `feedback/upload` | POST | 上传用户反馈 |

**初始化握手：**

```json
// 客户端 → 服务器
{
  "jsonrpc": "2.0",
  "method": "initialize",
  "params": {
    "clientInfo": { "name": "vscode-codex", "version": "1.0.0" },
    "capabilities": {}
  },
  "id": 1
}

// 服务器 → 客户端
{
  "jsonrpc": "2.0",
  "result": {
    "serverInfo": { "name": "codex-app-server", "version": "0.1.0" },
    "capabilities": {}
  },
  "id": 1
}

// 客户端 → 服务器（确认初始化完成）
{
  "jsonrpc": "2.0",
  "method": "initialized",
  "params": {}
}
```

**背压处理：**

当客户端无法及时处理事件时，服务器通过有界队列实现背压：

```rust
/// 有界事件队列（防止内存溢出）
const EVENT_QUEUE_CAPACITY: usize = 1024;

/// 背压错误码
const BACKPRESSURE_ERROR_CODE: i32 = -32001;

// 当队列满时，返回背压错误
if event_queue.is_full() {
    return Err(JsonRpcError {
        code: BACKPRESSURE_ERROR_CODE,
        message: "Event queue full, client is too slow".to_string(),
        data: None,
    });
}
```

---

## 3.17 实时通信

### 3.17.1 WebRTC 实现

Codex CLI 的实时通信从 WebSocket 迁移到 **WebRTC**，以获得更好的实时性能和更低的延迟。

**从 WebSocket 迁移到 WebRTC 的动机：**

| 维度 | WebSocket | WebRTC |
|------|-----------|--------|
| **延迟** | ~50-100ms | ~10-30ms |
| **音频质量** | 一般 | 优秀（opus 编解码） |
| **NAT 穿透** | 需要代理 | 内置 ICE/STUN/TURN |
| **数据通道** | 文本为主 | 支持二进制 |
| **浏览器兼容** | 通用 | 通用 |

**核心技术栈：**

```
┌──────────────────────────────────────────────────────────────┐
│                 WebRTC 实时通信架构                            │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 音频编解码: opus-rs                                   │   │
│  │  • 低延迟音频编解码                                   │   │
│  │  • 自适应比特率                                       │   │
│  │  • 适用于语音交互场景                                  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Data Channel: 控制消息                                │   │
│  │  • 文本消息传输                                       │   │
│  │  • 会话控制命令                                       │   │
│  │  • 可靠有序传输                                       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ RTP Track: 音频流                                     │   │
│  │  • 实时音频传输                                       │   │
│  │  • 适用于语音对话场景                                  │   │
│  │  • 低延迟优先                                         │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ SDP 协商                                              │   │
│  │  1. 客户端创建 offer                                  │   │
│  │  2. 发送 offer 到服务器                               │   │
│  │  3. 服务器生成 answer                                 │   │
│  │  4. 客户端设置 answer，建立连接                        │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

**SDP offer/answer 协商流程：**

```json
// 客户端 → 服务器: 开始实时对话
{
  "type": "user_turn",
  "items": [...],
  "conversation_start": {
    "transport": {
      "type": "webrtc",
      "sdp": "v=0\r\no=- 123456 2 IN IP4 127.0.0.1\r\n..."
    }
  }
}

// 服务器 → 客户端: 返回 SDP answer
{
  "type": "realtime_conversation_start",
  "sdp_answer": "v=0\r\no=- 654321 2 IN IP4 127.0.0.1\r\n..."
}
```

**4-PR 迁移栈（#16805-#16807, #16769）：**

WebRTC 迁移涉及 4 个 Pull Request 的渐进式实现：

| PR | 说明 |
|-----|------|
| #16805 | WebRTC 基础架构和 SDP 协商 |
| #16806 | opus-rs 音频编解码集成 |
| #16807 | Data Channel 控制消息协议 |
| #16769 | RTP track 音频流传输 |

**实时 API 端点：**

```
ConversationStartTransport::Websocket  → WebSocket 传输
ConversationStartTransport::Webrtc { sdp } → WebRTC 传输

实时事件类型:
├── SessionUpdated          # 会话配置更新
├── InputAudioSpeechStarted # 用户开始说话
├── InputTranscriptDelta    # 用户语音转写增量
├── OutputTranscriptDelta   # 模型语音转写增量
├── AudioOut                # 模型音频输出帧
├── ResponseCreated         # 响应创建
├── ResponseCancelled       # 响应取消
├── ResponseDone            # 响应完成
├── ConversationItemAdded   # 对话项添加
├── ConversationItemDone    # 对话项完成
├── HandoffRequested        # 交接请求
└── Error                   # 错误
```

**支持的语音列表：**

```rust
pub enum RealtimeVoice {
    // V1 语音
    Juniper, Maple, Spruce, Ember, Vale, Breeze, Arbor, Sol, Cove,
    // V2 语音
    Alloy, Ash, Ballad, Coral, Echo, Sage, Shimmer, Verse,
    Marin, Cedar,
}
```

默认语音：V1 为 `Cove`，V2 为 `Marin`。
---
# 对比分析

## 4.1 架构哲学差异

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **核心理念** | 深度嵌入开发者终端环境，通过多层权限系统实现"强大但受监督"的 AI 助手 | 在本地机器上安全高效运行，通过 OS 级沙箱隔离实现"零信任"安全模型 |
| **设计哲学** | 信任权限系统 — AI 应该功能强大，但每一步都经过多层安全检查 | 不信任 AI — 操作系统内核本身阻止危险操作 |
| **模型支持** | 仅限 Claude 系列（Anthropic API、Bedrock、Vertex AI、Foundry） | OpenAI 模型（支持多提供商，包括本地 Ollama/LM Studio） |
| **开源策略** | 闭源（曾有泄露版本被分析） | Apache 2.0 开源 |

**深入分析：**

Claude Code 和 Codex CLI 的架构哲学差异源于两家公司对 AI 安全的不同根本假设。Claude Code 采取的是**"信任但验证"**（Trust but Verify）的策略——它假设模型本身是高度对齐的（通过 RLHF 和 Constitutional AI 训练），因此可以赋予模型强大的能力，但通过多层软件级权限管道来防止潜在的误用。这种哲学使得 Claude Code 在功能上更加丰富和灵活，能够支持 40+ 工具、多 Agent 团队协作、后台梦境任务等高级特性。代价是权限系统本身成为了一个庞大的攻击面——精心构造的提示注入可能绕过多层权限检查。

Codex CLI 则采取了**"零信任"**（Zero Trust）策略——它假设 AI 模型可能做出任何操作，因此将安全边界交由操作系统内核来强制执行。这种哲学的根源在于 OpenAI 对企业级部署的考量：在企业环境中，安全必须由不可绕过的机制来保证，而非依赖软件层的规则匹配。沙箱方案的优势在于即使模型被成功注入恶意指令，操作系统内核也会阻止其执行危险操作（如删除用户文件、访问敏感目录）。但代价是功能灵活性受限——所有工具调用必须串行执行（因为沙箱约束），且某些需要系统级权限的操作无法实现。

从工程角度看，两种哲学代表了 AI 编码助手安全设计的两个极端：
- **Claude Code** 更像一个"有经验的助手"——能力强但需要监督，适合信任 AI 能力的开发者
- **Codex CLI** 更像一个"受限的实习生"——能力有限但不会闯祸，适合安全优先的企业环境

## 4.2 Agent Loop 差异

Agent Loop 是 AI 编码助手的核心引擎，两者的实现方式截然不同，反映了各自的设计哲学和技术栈选择。

### 4.2.1 循环模型对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **循环范式** | 流式异步生成器（AsyncGenerator）双层循环 | 提交-事件模式（Submission Queue / Event Queue） |
| **核心实现** | `query.ts` — `async function* query()` 生成器 | `codex.rs` — 事件循环 + `agent_loop` 模块 |
| **驱动方式** | 生成器 yield 事件流，UI 消费事件 | Submission Queue 推送请求，Event Queue 消费响应 |
| **终止条件** | `stop_reason === "end_turn"` 或 max-output-tokens 恢复 | API 返回完成信号或上下文溢出 |

**Claude Code：流式异步生成器双层循环**

Claude Code 的 Agent 循环采用 TypeScript 的 `async function*` 生成器模式，这是整个系统最核心的设计决策之一。`query()` 函数是一个异步生成器，它持续 `yield` 事件（`StreamEvent | Message`），外层的 `queryLoop` 消费这些事件并更新 UI。

```
queryLoop (外层循环 — 管理会话级状态)
  │
  └─▶ query() (内层循环 — async generator)
        │
        ├─▶ queryModelWithStreaming() — 调用 API
        │     ├─ 构建系统提示（工具描述 + CLAUDE.md + 上下文）
        │     ├─ 发送请求到 Anthropic API
        │     └─ 解析 SSE 流 → yield StreamEvent
        │
        ├─▶ 检查 stop_reason
        │     ├─ "end_turn" → 完成
        │     ├─ "tool_use" → 进入工具执行
        │     └─ "max_tokens" → 恢复机制（最多3次重试）
        │
        └─▶ runTools() — 工具执行编排
              ├─ 权限检查（多层管道）
              ├─ 并发/串行分批执行
              └─ yield 工具结果
```

这种双层循环设计的优势在于：
1. **关注点分离**：外层 `queryLoop` 管理会话级状态（消息历史、文件缓存、用量追踪），内层 `query()` 专注于单轮对话的工具调用循环
2. **流式交互**：生成器的 `yield` 机制天然支持流式输出，模型开始生成文本的瞬间用户就能看到内容
3. **可组合性**：生成器可以被多个消费者同时监听（UI 渲染、日志记录、遥测上报）

**Codex CLI：提交-事件模式**

Codex CLI 的 Agent 循环基于 Rust 的异步通道（Channel）实现，采用经典的 Submission Queue / Event Queue 模式：

```
Submission Queue (用户输入 + 工具结果)
  │
  ▼
Agent Loop (core/agent/)
  │
  ├─▶ 构建请求（系统指令 + 完整对话历史）
  ├─▶ 调用 OpenAI Responses API（SSE 流式）
  ├─▶ 解析事件流
  │     ├─ 文本输出 → yield 到 Event Queue
  │     ├─ 工具调用 → 进入工具执行流程
  │     └─ 完成信号 → 退出循环
  │
  └─▶ 工具执行（严格串行）
        ├─ Guardian 审批
        ├─ 沙箱执行
        └─ 结果追加到对话历史 → 重新提交
```

这种设计的优势在于：
1. **无状态性**：每个请求携带完整的对话历史，服务端无需维护状态
2. **Rust 所有权模型**：通道传递所有权，避免数据竞争
3. **可测试性**：可以轻松模拟 Submission/Event 进行单元测试

### 4.2.2 状态管理对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **状态模型** | 有状态（Stateful） | 无状态（Stateless） |
| **状态持有者** | `QueryEngine` 实例 | 无（每次请求自包含） |
| **消息历史** | 内存中累积，按需压缩 | 每次请求携带完整历史 |
| **文件缓存** | LRU 缓存（100 文件 / 25MB） | 文件快照系统 |
| **Abort 控制** | `AbortController` 实例 | `CancellationToken` |
| **用量追踪** | `CostTracker` 实例 | 内置 token 计数器 |

**Claude Code 的有状态设计：**

Claude Code 的 `QueryEngine` 是一个有状态的单例（每个会话一个实例），它在整个会话期间持有：
- **消息历史**（`messages[]`）：所有对话消息的累积数组，包括用户消息、助手消息、工具调用和工具结果
- **文件缓存**（LRU Cache）：最近读取的文件内容，避免重复读取
- **AbortController**：用于取消正在进行的 API 请求和工具执行
- **用量追踪**：Token 消耗、费用计算、速率限制监控

这种有状态设计的优势是**内存效率高**——不需要每次请求都序列化完整历史，且可以维护跨轮次的文件缓存。但代价是**复杂性高**——状态管理成为了一个核心挑战，需要处理并发访问、状态一致性、内存泄漏等问题。

**Codex CLI 的无状态设计：**

Codex CLI 采取了完全不同的策略——每个 API 请求都是完全自包含的，携带完整的对话历史。这意味着：
- **服务端无状态**：API 服务不需要维护会话状态
- **可恢复性**：如果进程崩溃，可以从保存的历史中恢复
- **可扩展性**：理论上可以轻松实现分布式部署

但代价是**网络开销大**——随着对话增长，每次请求的 payload 越来越大。Codex CLI 通过提示词前缀缓存来缓解这个问题：由于每次请求都是前一次的精确前缀（仅追加新消息），API 缓存可以命中大部分前缀内容，避免重复计算。

### 4.2.3 流式处理对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **流式协议** | HTTP SSE（Anthropic Messages API） | HTTP SSE（OpenAI Responses API） |
| **流式解析** | 自定义 SSE 解析器 | reqwest + 自定义事件解析 |
| **关键优化** | `StreamingToolExecutor` — 模型生成时就开始执行工具 | 工具调用严格串行，等待完整响应 |
| **事件类型** | `text_delta`, `thinking_delta`, `tool_use`, `content_block` | `response.output_text`, `response.function_call`, `response.completed` |
| **背压控制** | 生成器天然背压 | 通道容量限制 |

**Claude Code 的 StreamingToolExecutor：**

这是 Claude Code 最独特的优化之一。传统的 Agent 循环是"等待完整响应 → 解析工具调用 → 执行工具 → 发送结果"的串行流程。Claude Code 的 `StreamingToolExecutor` 打破了这个限制——它在模型仍在生成响应时，就开始执行已经完成的工具调用块。

```
时间线：
  t0: 模型开始生成 → "我来帮你分析这个文件"
  t1: 模型生成 tool_use 块 1 (FileRead) → [完整]
  t2: 模型继续生成 → "同时我也需要查看..."
  t3: 模型生成 tool_use 块 2 (Grep) → [完整]
  t4: 模型继续生成 → "基于以上信息..."

  传统模式: t0──────t4 等待完整响应 ──→ 执行 FileRead ──→ 执行 Grep
  Claude Code: t1 立即执行 FileRead ──→ t3 立即执行 Grep ──→ t4 响应完成
```

这种"边生成边执行"的模式显著减少了端到端延迟，特别是在模型需要调用多个只读工具（如文件读取、搜索）的场景下。由于只读工具之间没有数据依赖，可以安全地并行执行。

**Codex CLI 的串行流式处理：**

Codex CLI 采用更保守的串行策略——等待模型完整生成所有工具调用后，再逐个执行。这是由沙箱约束决定的：每个工具调用都在独立的沙箱进程中执行，且沙箱策略可能因工具而异（如 shell 命令和网络请求的沙箱策略不同），因此无法简单地并行化。

### 4.2.4 工具并发对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **并发模型** | 按安全性分批（并行 + 串行） | 全部串行 |
| **并行上限** | 最多 10 个并发工具 | N/A（串行） |
| **分类依据** | `isConcurrencySafe(input)` 方法 | N/A |
| **错误传播** | `siblingAbortController` 取消同批工具 | `CancellationToken` 传播 |
| **串行化原因** | 写操作需要保证顺序 | 沙箱进程隔离约束 |

**Claude Code 的并发编排策略：**

Claude Code 的工具并发执行是一个精心设计的系统，核心逻辑如下：

1. **分类阶段**：模型返回多个工具调用后，系统遍历每个调用，调用 `tool.isConcurrencySafe(input)` 判断是否可以并行执行
   - 只读工具（如 `FileRead`、`Grep`、`Glob`）通常返回 `true`
   - 写操作工具（如 `FileEdit`、`Bash`）通常返回 `false`

2. **并行批次**：所有标记为安全的工具放入并行批次，使用 `Promise.all` 并发执行，上限 10 个

3. **串行批次**：非安全工具在前一批完成后逐个执行

4. **兄弟错误处理**：并行批次中，如果某个工具执行失败（如文件不存在），`siblingAbortController.abort()` 会立即取消同批次中仍在执行的其他工具，避免浪费资源

```
工具调用: [FileRead(a.ts), FileRead(b.ts), FileEdit(c.ts), Grep(pattern), Bash(cmd)]

分类结果:
  并行批: [FileRead(a.ts), FileRead(b.ts), Grep(pattern)]  ← 3个只读工具
  串行批: [FileEdit(c.ts), Bash(cmd)]                      ← 2个写操作

执行时间线:
  t0: FileRead(a.ts) ─┐
  t0: FileRead(b.ts) ─┤ 并行执行
  t0: Grep(pattern)  ─┘
  t1: FileEdit(c.ts)  ← 串行执行
  t2: Bash(cmd)       ← 串行执行
```

**Codex CLI 的串行约束：**

Codex CLI 的全部串行策略是由沙箱架构决定的。每个工具调用都在独立的沙箱进程中执行，沙箱的创建和销毁有固定开销，且不同工具可能需要不同的沙箱策略（如 shell 命令需要更严格的沙箱配置）。此外，串行执行简化了错误处理和状态管理——每个工具的结果在下一个工具执行前就已经确定。

### 4.2.5 缓存优化对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **缓存策略** | Fork 子 Agent 共享前缀 + 动态边界分割 | 精确前缀匹配 + Zstd 压缩 |
| **缓存层级** | API 提示词缓存（Anthropic 侧） | API 提示词缓存（OpenAI 侧） |
| **前缀共享** | Fork 机制：子 Agent 共享父对话历史前缀 | 自然形成：每次请求是前一次的精确前缀 |
| **边界优化** | `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 标记 | 无特殊标记 |
| **压缩** | 无（依赖 API 侧缓存） | Zstd 压缩对话历史 |

**Claude Code 的 Fork 缓存优化：**

当 Claude Code 需要生成多个子 Agent（如协调器模式下的多个 Worker）时，它使用 Fork 机制来最大化 API 提示词缓存命中率：

```
父 Agent 上下文: [系统提示 + 对话历史 + 任务描述]
  │
  ├─ Fork → 子 Agent A: [父上下文 + "请分析模块A"]
  ├─ Fork → 子 Agent B: [父上下文 + "请分析模块B"]
  └─ Fork → 子 Agent C: [父上下文 + "请分析模块C"]

API 缓存视角:
  前缀 [系统提示 + 对话历史 + 任务描述] 被缓存一次
  3 个子 Agent 请求只需计算各自不同的后缀部分
  缓存命中率 ≈ (公共前缀长度) / (总长度) → 通常 >80%
```

此外，Claude Code 通过 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 标记将系统提示分为静态部分（全局指令、工具描述）和动态部分（会话状态、Git 信息），确保静态部分始终被缓存。

**Codex CLI 的前缀缓存：**

Codex CLI 的无状态设计天然适合前缀缓存——由于每次请求都是前一次的精确前缀（仅追加新的对话轮次），OpenAI API 的缓存机制可以高效地命中大部分内容。Codex CLI 还使用 Zstd 压缩对话历史，进一步减少网络传输开销。

### 4.2.6 错误恢复对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **max-tokens 恢复** | 最多 3 次重试，追加续写提示 | N/A |
| **上下文溢出** | `reactiveCompact()` — 压缩旧消息 | `ContextWindowExceeded` — 移除最旧历史项 |
| **API 错误** | `withRetry()` — 指数退避 + 抖动 | 指数退避 + API `Duration` 头 |
| **流中断** | 重新发起请求 | 重新发起请求 |
| **工具失败** | `siblingAbortController` + 错误消息反馈 | 错误消息反馈给模型 |

**Claude Code 的 max-output-tokens 恢复机制：**

当 Claude 模型的输出达到 `max-output-tokens` 限制时（通常意味着模型还没说完就被截断了），Claude Code 会自动恢复：

```
1. 检测到 stop_reason === "max_tokens"
2. 追加续写提示: "[Please continue from where you left off]"
3. 重新调用 API（携带完整历史 + 续写提示）
4. 最多重试 3 次
5. 如果仍然失败，将已有内容作为最终结果返回
```

这个机制确保了模型在处理复杂任务时不会因为输出长度限制而丢失信息。

**Codex CLI 的 ContextWindowExceeded 处理：**

当对话历史超出模型的上下文窗口时，Codex CLI 采取更直接的方式——移除最旧的历史项，直到对话历史适合上下文窗口。这种方式简单有效，但可能丢失重要的早期上下文。Codex CLI 还支持远程压缩（`compact_remote.rs`），利用云端 API 对对话历史进行摘要压缩。

## 4.3 工具系统差异

### 4.3.1 工具数量与分类对比

| 分类 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **核心工具** | BashTool, FileReadTool, FileEditTool, FileWriteTool, GlobTool, GrepTool, AgentTool, SkillTool | shell, apply_patch, view_image |
| **任务管理** | TaskCreate, TaskGet, TaskUpdate, TaskList, TaskOutput, TaskStop | tasks (spawn, resume, wait, close, send-message, list, assign-task) |
| **多 Agent** | AgentTool, TeamCreate, TeamDelete, SendMessageTool | spawn_agents_on_csv |
| **Web 工具** | WebFetchTool, WebSearchTool, WebBrowserTool (feature-gated) | N/A |
| **搜索发现** | ToolSearchTool | tool_search, tool_suggest |
| **计划模式** | EnterPlanMode, ExitPlanMode | N/A |
| **消息通信** | SendMessageTool, PushNotification (feature-gated) | send-message |
| **监控调度** | MonitorTool (feature-gated), CronCreate/Delete/List (feature-gated) | N/A |
| **工作流** | WorkflowTool (feature-gated) | N/A |
| **实验性** | JS REPL (feature-gated) | JS REPL (feature-gated) |
| **MCP 桥接** | MCPTool (动态发现) | MCP 工具适配器 |
| **总计** | 40+ | 25+ |

Claude Code 的工具数量几乎是 Codex CLI 的两倍，这反映了两者不同的设计理念：
- **Claude Code** 倾向于提供丰富的内置工具，覆盖尽可能多的使用场景，减少对 MCP 外部工具的依赖
- **Codex CLI** 倾向于提供最小化的核心工具集，通过 MCP 协议扩展功能

### 4.3.2 文件编辑方式深度对比

| 维度 | Claude Code (search-and-replace) | Codex CLI (unified diff patch) |
|------|----------------------------------|-------------------------------|
| **编辑格式** | `old_str` + `new_str` 精确替换 | 标准 unified diff 格式 |
| **上下文要求** | 需要精确匹配旧文本 | 需要提供上下文行 |
| **可审查性** | 替换前后对比清晰 | 标准 diff 格式，可用 `diff`/`patch` 工具审查 |
| **多位置编辑** | 需要多次调用 | 单次 patch 可包含多个 hunk |
| **错误容忍** | 匹配失败时提供模糊匹配建议 | patch 应用失败时报告冲突 |
| **模型负担** | 需要精确复制旧文本 | 需要生成正确的 diff 语法 |
| **语义意图** | "将这段文本替换为那段文本" | "对这个文件应用这些变更" |

**Claude Code 的 search-and-replace 方式：**

```typescript
// Claude Code 的文件编辑调用格式
{
  "tool": "file_edit",
  "input": {
    "path": "/src/components/App.tsx",
    "old_str": "const greeting = 'Hello';",
    "new_str": "const greeting = 'Bonjour';"
  }
}
```

**优势：**
- **直观易懂**：替换语义清晰，模型和人类都能轻松理解
- **精确控制**：只替换匹配的文本，不会意外修改其他部分
- **低认知负担**：模型不需要学习 diff 语法
- **模糊匹配**：当精确匹配失败时，系统会尝试找到最接近的匹配位置

**劣势：**
- **冗余**：需要完整复制旧文本，浪费 token
- **大范围修改低效**：如果需要修改文件中多个位置，需要多次调用
- **脆弱性**：如果旧文本有微小差异（如空格、换行），匹配可能失败

**Codex CLI 的 unified diff patch 方式：**

```diff
--- a/src/components/App.tsx
+++ b/src/components/App.tsx
@@ -10,7 +10,7 @@
 function App() {
+  const [count, setCount] = useState(0);
   return (
     <div>
-      <h1>Hello</h1>
+      <h1>Hello {count}</h1>
+      <button onClick={() => setCount(c => c+1)}>+</button>
     </div>
   );
 }
```

**优势：**
- **标准格式**：unified diff 是软件工程的通用标准，所有开发者都熟悉
- **可审查性**：可以用标准 `diff`/`patch` 工具审查和应用
- **多位置编辑**：单个 patch 可以包含多个 hunk，一次修改文件中多个位置
- **代码审查友好**：diff 格式天然适合代码审查场景
- **语义设计选择**：强制模型推理上下文行，鼓励精确编辑而非整体重写

**劣势：**
- **语法复杂**：模型需要正确生成 diff 语法（行号、`@@` 标记、`+/-` 前缀）
- **行号敏感性**：如果文件在生成 diff 后被修改，行号可能偏移导致应用失败
- **冲突处理**：多个 hunk 之间可能存在依赖关系，应用顺序影响结果

### 4.3.3 延迟加载对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **触发方式** | `ToolSearchTool` — 模型主动查询 | `tool_search` / `tool_suggest` — API 内置 |
| **加载粒度** | 单个工具的完整 schema | 工具列表 + schema |
| **系统提示影响** | 仅包含工具名称列表（无 schema） | 同样仅包含名称 |
| **缓存策略** | 加载后缓存在当前会话 | API 侧缓存 |
| **适用场景** | 100+ MCP 工具的大规模场景 | 动态工具发现 |

**Claude Code 的 ToolSearchTool 机制：**

当 Claude Code 连接了大量 MCP 服务器时，工具 schema 的总量可能非常庞大。如果将所有 schema 都包含在系统提示中，会浪费大量 token 并可能降低模型性能。ToolSearchTool 的解决方案：

```
系统提示中:
  "可用工具: mcp__slack__send_message, mcp__github__create_issue, ..."

模型思考: "我需要发送 Slack 消息"
  → 调用 ToolSearch({ query: "select:mcp__slack__send_message" })
  → 返回: { name: "mcp__slack__send_message", schema: {...} }
  → 模型现在有了完整的 schema，可以正确调用该工具
```

**Codex CLI 的 tool_search / tool_suggest：**

Codex CLI 利用 OpenAI Responses API 的内置 `tool_search` 和 `tool_suggest` 功能，这是 API 层面提供的动态工具发现机制，不需要客户端额外实现。

### 4.3.4 工具接口定义对比

| 维度 | Claude Code (TypeScript) | Codex CLI (Rust) |
|------|--------------------------|------------------|
| **接口定义** | `interface Tool` + Zod schema | `trait Tool` + serde 序列化 |
| **输入验证** | Zod v4 运行时验证 | serde 编译时 + 运行时验证 |
| **类型安全** | TypeScript 编译时 + Zod 运行时 | Rust 编译时保证 |
| **Schema 生成** | 从 Zod schema 自动生成 JSON Schema | 从 Rust 结构体 derive 生成 |
| **扩展方式** | 实现接口 + 注册到工具池 | 实现 trait + 注册到工具注册表 |

**Claude Code 的 TypeScript 接口：**

```typescript
interface Tool {
  name: string
  description: string
  inputJSONSchema: JSONSchema           // Zod 验证输入
  call(input, context): Promise<Result> // 核心执行
  validateInput?(input): ValidationResult
  checkPermissions?(input, context): PermissionResult
  isConcurrencySafe(input): boolean     // 能否并行运行？
  isReadOnly?: boolean                  // 无副作用？
  isEnabled?(): boolean                 // 特性门控？
  shouldDefer?: boolean                 // 懒加载 schema？
}
```

**Codex CLI 的 Rust trait：**

```rust
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value;
    async fn execute(&self, input: serde_json::Value, ctx: &ToolContext)
        -> Result<ToolOutput, ToolError>;
    fn is_read_only(&self) -> bool { false }
}
```

两者的关键差异在于：
- **Claude Code** 的接口更加丰富，包含权限检查、并发安全性判断、延迟加载等元数据
- **Codex CLI** 的 trait 更加精简，安全和并发问题由外部系统（沙箱、Guardian）处理

## 4.4 上下文管理差异

### 4.4.1 压缩策略对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **压缩层级** | 四层渐进式（微压缩 → 自动压缩 → 会话记忆 → 反应式） | 两层（本地压缩 + 远程压缩）+ Token 截断 |
| **触发阈值** | 80% / 85% / 90% / 98% token 使用率 | 上下文窗口溢出时触发 |
| **压缩方式** | LLM 摘要（发送旧消息给模型进行摘要） | LLM 摘要（本地或远程） |
| **信息保留** | 提取关键信息到持久记忆文件 | 移除最旧历史项 |
| **渐进性** | 高（四层渐进，每层处理不同严重程度） | 低（主要依赖溢出触发） |

**Claude Code 的四层渐进式压缩：**

Claude Code 的压缩系统是其上下文管理最精妙的设计之一，四个层级对应不同的 token 使用率阈值：

```
Token 使用率: 0% ──────── 80% ──── 85% ──── 90% ──── 98% ──── 100%

层级 1: 微压缩 (Micro-Compact) — 阈值 80%
  • 清除旧工具结果（替换为 "[Old tool result content cleared]"）
  • 针对 FileRead、Bash、Grep 等工具
  • 基于时间阈值（旧于 N 轮的结果被清除）
  • 影响：减少约 10-20% 的 token 使用

层级 2: 自动压缩 (Auto-Compact) — 阈值 ~167K tokens (约 85%)
  • 发送旧消息给模型进行摘要
  • 替换为压缩边界标记
  • 保留最近 N 轮对话完整不变
  • 影响：通常减少 50-70% 的 token 使用

层级 3: 会话记忆压缩 (Session Memory Compact) — 阈值 90%
  • 提取关键信息到持久会话记忆
  • 保持 10K-40K tokens 的记忆窗口
  • 记忆文件按类型分类（用户角色、反馈、项目上下文等）
  • 影响：跨会话保留关键信息

层级 4: 反应式压缩 (Reactive Compact) — 阈值 98% 或 API 错误
  • 由 API 的 `prompt_too_long` 错误触发
  • 截断最旧的消息组
  • 最后手段，可能导致信息丢失
```

**Codex CLI 的压缩策略：**

Codex CLI 的压缩相对简单直接：

- **本地压缩**（`compact.rs`）：当对话过长时，使用本地模型对早期对话进行摘要
- **远程压缩**（`compact_remote.rs`）：利用云端 API 进行更高质量的压缩
- **Token 截断**：通过输出截断工具防止工具结果占据过多上下文
- **历史项移除**：当 `ContextWindowExceeded` 错误发生时，移除最旧的历史项

### 4.4.2 Token 计算方式对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **计算公式** | `字符数 / 4 * 4 / 3` | `(len + 3) / 4`（向上取整） |
| **精度** | 已知 10-30% 误差 | 类似精度 |
| **用途** | Token 预算追踪、压缩触发判断 | 上下文窗口管理 |
| **备注** | 保守估计，倾向于高估 | 简单估算 |

Claude Code 的 token 计算公式 `字符数 / 4 * 4 / 3` 本质上是 `字符数 / 3`，这是一个保守的估计值（实际 Claude 模型的 tokenizer 大约是 3-4 个字符 per token）。已知这个估算有 10-30% 的误差，但作为预算追踪的近似值已经足够——系统宁愿在 80% 使用率时就开始压缩，也不愿等到真正溢出。

Codex CLI 的 `(len + 3) / 4` 更加简洁，假设平均 4 个字符对应 1 个 token，这是英语文本的常见比例。对于包含大量代码（通常 token 效率更高）的对话，这个估算可能偏低。

### 4.4.3 持久记忆对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **记忆格式** | `MEMORY.md` + 分类文件 | `AGENTS.md` + rollout 持久化 |
| **存储位置** | `~/.claude/projects/<slug>/memory/` | 项目根目录 `AGENTS.md` |
| **记忆分类** | 4 类（用户角色、反馈、项目、参考） | 无分类 |
| **自动更新** | Dream Task 后台整合 | 无自动更新 |
| **跨会话** | 支持（持久化到磁盘） | 支持（文件系统） |
| **容量限制** | MEMORY.md 最多 200 行 | 无明确限制 |

**Claude Code 的记忆系统：**

Claude Code 的持久记忆系统是业界最完善的 AI 助手记忆方案之一：

```
~/.claude/projects/<project-slug>/memory/
├── MEMORY.md           # 索引文件 (最多 200 行，自动维护)
├── user_role.md        # 用户类型：角色、偏好、工作风格
├── feedback_testing.md # 反馈类型：要重复/避免的行为模式
├── project_auth.md     # 项目类型：持续工作上下文、架构决策
└── reference_docs.md   # 参考类型：外部系统指针、API 文档链接
```

**Dream Task（梦境任务）** 是 Claude Code 独有的后台记忆整合机制——在用户空闲时，系统自动启动一个后台 Agent，回顾最近的会话并更新记忆文件。这实现了真正的"跨会话学习"，AI 助手会随着使用时间的增长而越来越了解用户和项目。

**Codex CLI 的记忆系统：**

Codex CLI 的记忆系统更加简洁，主要通过 `AGENTS.md` 文件实现：
- 项目级指令文件，向上遍历目录树查找
- 类似于 Claude Code 的 `CLAUDE.md`，但功能更简单
- rollout 持久化：将 Agent 的执行结果持久化到磁盘，支持恢复

### 4.4.4 提示词缓存对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **缓存机制** | 动态边界分割 + Fork 前缀共享 | 精确前缀匹配 + Zstd 压缩 |
| **边界标记** | `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` | 无特殊标记 |
| **Fork 优化** | 子 Agent 共享父对话历史前缀 | N/A（无 Fork 机制） |
| **压缩** | 无额外压缩 | Zstd 压缩对话历史 |

**Claude Code 的动态边界分割：**

Claude Code 在系统提示中插入 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 标记，将系统提示分为两部分：
- **静态部分**（标记之前）：全局指令、工具描述、MCP 服务器配置等，这些内容在会话中不会变化
- **动态部分**（标记之后）：当前 Git 状态、日期、会话状态等，这些内容每次请求都可能不同

API 缓存可以高效地命中静态部分（通常占系统提示的 60-80%），仅重新计算动态部分。

## 4.5 安全模型差异

安全模型是 Claude Code 和 Codex CLI **最根本的架构区别**，也是选择哪个工具时最重要的考量因素。

### 4.5.1 安全基础对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **安全基础** | 软件层多层权限管道 | OS 内核级沙箱 |
| **执行层** | 用户空间进程 | 受限沙箱进程 |
| **强制力** | 软件规则（可被绕过） | 内核策略（不可绕过） |
| **设计假设** | AI 大部分时间是安全的 | AI 任何时候都可能不安全 |
| **安全保证** | "概率性安全" — 大部分攻击会被阻止 | "确定性安全" — 内核保证的边界 |

**Claude Code 的软件层安全：**

Claude Code 的安全系统完全构建在软件层面，通过多层权限管道来控制 AI 的行为。这种方案的优势是灵活性极高——可以为每个工具、每个操作配置精细的权限规则。但核心局限在于：**软件层的规则本质上是可以被绕过的**。如果攻击者能够构造出足够巧妙的提示注入，让 AI 请求一个看似良性但实际危险的操作（如修改 `.bashrc` 添加后门），权限系统可能无法识别。

Claude Code 的 7 层安全机制（危险文件保护、危险命令检测、Bypass 终止开关、Auto 断路器、拒绝追踪、技能范围收窄、MCP Shell 阻止）提供了纵深防御，但每一层都是软件实现，理论上都可以被绕过。

**Codex CLI 的 OS 内核级安全：**

Codex CLI 的安全基础是操作系统内核提供的沙箱机制：
- **macOS**：Apple Seatbelt (`sandbox-exec`) — 内核级文件系统和网络访问控制
- **Linux**：Landlock（内核级文件系统隔离）+ Bubblewrap（用户命名空间隔离）
- **Windows**：Restricted Token（令牌限制）+ Elevated Runner（提升权限后端）

这些机制由操作系统内核强制执行，**AI 模型无法绕过**。即使模型被注入了恶意指令，试图执行 `rm -rf /`，内核也会阻止该操作。这是"确定性安全"——安全保证不依赖于软件规则的正确性，而是依赖于操作系统内核的安全属性。

### 4.5.2 隔离级别对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **进程隔离** | 无（所有工具在同一进程中执行） | 有（每个工具在独立沙箱进程中） |
| **文件系统隔离** | 无（依赖权限规则） | 有（沙箱限制可访问的目录） |
| **网络隔离** | 无（依赖工具层控制） | 有（沙箱默认阻止网络访问） |
| **资源限制** | 无 | 有（CPU、内存、磁盘配额） |
| **沙箱逃逸风险** | N/A（无沙箱） | 极低（需要内核漏洞） |

这是两者安全差异的核心所在。Claude Code 的所有工具调用都在同一个进程中执行，没有进程级隔离。这意味着如果某个工具被攻破（如通过恶意 MCP 服务器），攻击者可能获得与 Claude Code 进程相同的权限。

Codex CLI 的每个工具调用都在独立的沙箱进程中执行，即使某个工具被攻破，攻击者也只能访问沙箱允许的资源。

### 4.5.3 决策机制对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **决策链** | 权限模式 → 规则匹配 → Hook → LLM 分类器 → 用户确认 | 沙箱策略（OS 内核执行）+ Guardian 审批 |
| **自动批准** | Auto 模式 + LLM 分类器 | Guardian `canAutoApprove` 策略 |
| **用户参与** | 5 种权限模式（default/acceptEdits/plan/bypass/auto） | 用户确认/拒绝 |
| **远程控制** | GrowthBook 远程特性门 + Statsig 实时开关 | 无远程控制 |
| **审计能力** | 拒绝追踪（>3 连续或 >20 总计） | N/A |

**Claude Code 的多层决策链：**

Claude Code 的安全决策经过最多 5 层处理：
1. **权限模式**：首先检查全局权限模式（bypass/dontAsk/default/acceptEdits/plan/auto）
2. **规则匹配**：应用 Allow/Deny/Ask 规则（基于工具名称、路径模式等）
3. **Hook 执行**：运行 PreToolUse Hook（可能是 Shell 命令、LLM 评估、HTTP 请求等）
4. **LLM 分类器**（Auto 模式）：使用 Haiku 模型评估操作安全性
5. **用户确认**：最终由用户决定是否允许

这种多层设计提供了极大的灵活性，但也引入了复杂性——每一层都有自己的规则和边缘情况，整体行为难以预测。

**Codex CLI 的 Guardian 模式：**

Codex CLI 的安全决策更加简洁：
1. **沙箱策略**：由 OS 内核强制执行，不可绕过
2. **Guardian 审批**：对于需要额外审批的操作（如写入文件），Guardian 模块决定是否自动批准或需要用户确认

Guardian 的 `canAutoApprove` 策略基于操作类型和上下文（如只读操作通常自动批准，写入操作需要确认）。

### 4.5.4 攻击面分析

| 攻击向量 | Claude Code 风险 | Codex CLI 风险 |
|----------|------------------|----------------|
| **提示注入** | 高风险 — 精心构造的提示可能绕过权限管道 | 中风险 — 沙箱限制实际损害 |
| **恶意 MCP 服务器** | 高风险 — MCP 工具不受权限管道完全保护 | 高风险 — MCP 工具不受沙箱保护 |
| **危险命令执行** | 中风险 — 危险命令检测 + 用户确认 | 低风险 — 沙箱阻止 |
| **数据泄露（网络）** | 高风险 — 工具可以发起网络请求 | 中风险 — 已批准的网络调用仍可能泄露 |
| **文件系统攻击** | 高风险 — 无进程隔离 | 低风险 — 沙箱隔离 |
| **供应链攻击** | 中风险 — 依赖 Anthropic API 安全性 | 中风险 — 依赖 OpenAI API 安全性 |

**关键发现：**

1. **MCP 工具是共同的安全盲点**：两个系统都支持 MCP 协议扩展工具，但 MCP 工具通常不受核心安全机制的保护。Claude Code 的权限管道可能不覆盖 MCP 工具的所有操作；Codex CLI 的沙箱不保护 MCP 工具的执行。

2. **网络访问是最大的数据泄露风险**：即使 Codex CLI 的沙箱阻止了未授权的网络访问，一旦用户批准了某个网络调用（如 API 请求），该调用可能被用来泄露数据到攻击者控制的服务器。

3. **Claude Code 的 LLM 分类器是双刃剑**：使用 LLM 来判断操作安全性是一个创新的想法，但引入了新的攻击面——攻击者可能尝试欺骗分类器本身。

### 4.5.5 安全优势与局限对比表

| 维度 | Claude Code 优势 | Claude Code 局限 | Codex CLI 优势 | Codex CLI 局限 |
|------|------------------|------------------|----------------|----------------|
| **安全性** | 多层纵深防御 | 软件层可被绕过 | 内核级保证 | MCP 工具不受保护 |
| **灵活性** | 精细权限控制 | 配置复杂 | 简单策略模型 | 粒度较粗 |
| **功能** | 支持更多危险操作 | 更大的攻击面 | 安全边界清晰 | 功能受限 |
| **企业适用** | 远程策略控制 | 需要信任 Anthropic | 零信任模型 | 需要信任 OpenAI |
| **用户体验** | Auto 模式减少打扰 | bypass 模式危险 | 沙箱透明 | 审批可能频繁 |

## 4.6 多 Agent 系统差异

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **架构层级** | 三级（子 Agent / 协调器 / 团队） | 完整多 Agent + 守护者模式 |
| **子 Agent 生成** | `AgentTool` — 模型主动生成 | `spawn` 原语 — 用户或 Agent 请求 |
| **隔离机制** | 隔离文件缓存 + 独立 AbortController | 独立沙箱容器 |
| **通信方式** | `SendMessageTool` 路由消息 | `send-message` 原语 |
| **工具过滤** | 按子 Agent 定义过滤工具池 | 继承父 Agent 工具 |
| **批量任务** | 不支持 | `spawn_agents_on_csv` — CSV 驱动批量 |
| **持久化** | 团队文件持久化到 `~/.claude/teams/` | 无持久化 |
| **结果聚合** | 协调器通过 XML 协议聚合 | 守护者模式管理 |
| **Fork 优化** | 共享前缀最大化缓存命中 | N/A |

**深入分析：**

Claude Code 的多 Agent 系统更加成熟和完整，提供了三个递进的层级：
1. **子 Agent**：最基础的单元，由主 Agent 通过 `AgentTool` 动态生成，适合简单的任务委派
2. **协调器模式**：系统提示重写为编排模式，通过 `AgentTool` 生成受限工具的 Worker，适合并行任务处理
3. **团队模式**：持久化的多 Agent 团队，支持消息路由和共享 scratchpad，适合长期协作项目

Claude Code 的 Fork 机制在多 Agent 场景下尤为重要——当从同一上下文生成多个子 Agent 时，它们共享相同的对话历史前缀，API 缓存可以高效地复用这部分内容。

Codex CLI 的多 Agent 系统更加务实，核心特点是每个 Agent 运行在独立的沙箱容器中，实现了真正的进程级隔离。`spawn_agents_on_csv` 是一个独特功能——可以从 CSV 文件批量生成 Agent 任务，适合大规模自动化场景（如批量代码审查、批量测试生成）。

## 4.7 配置系统对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **配置格式** | JSON (`settings.json`) | TOML (`config.toml`) |
| **优先级层数** | 5 级 | 5 级 |
| **项目指令** | `CLAUDE.md`（4 级目录遍历） | `AGENTS.md`（向上遍历目录树） |
| **热重载** | `settingsChangeDetector` + debounce | `reloadUserConfig` |
| **特性标志** | GrowthBook + `bun:bundle` DCE | `feature.state` 指标 |
| **MDM 支持** | 3 平台托管设置文件 | `requirements.toml` |
| **配置位置** | `.claude/settings.json`（项目/用户） | `~/.config/codex/config.toml`（用户） |

**Claude Code 的 5 级配置优先级（从低到高）：**
1. 内置默认值
2. 企业 MDM 策略（`/etc/claude-code/settings.json`）
3. 用户全局设置（`~/.claude/settings.json`）
4. 项目共享设置（`.claude/settings.json`）
5. 项目本地设置（`.claude/settings.local.json`）

**Codex CLI 的 5 级配置优先级（从低到高）：**
1. 内置默认值
2. 系统级配置
3. 用户级配置（`~/.config/codex/config.toml`）
4. 项目级配置（`./codex.toml`）
5. 环境变量覆盖

**CLAUDE.md vs AGENTS.md：**

两者都支持项目级指令文件，但实现方式不同：
- **Claude Code**：`CLAUDE.md` 支持在 4 个目录层级中查找（`/etc/claude-code/`、`~/.claude/`、项目根目录、`.claude/`），还支持 `CLAUDE.local.md`（不提交到版本控制）和 `.claude/rules/*.md`（模块化规则）
- **Codex CLI**：`AGENTS.md` 向上遍历目录树查找，支持多个文件合并

## 4.8 认证系统对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **OAuth 流程** | PKCE 流程（6 级解析链） | ChatGPT 账户 + 设备码 |
| **API Key** | `ANTHROPIC_API_KEY` | `OPENAI_API_KEY` |
| **密钥存储** | macOS Keychain / 明文文件 | `keyring-store`（4 平台支持） |
| **Token 刷新** | 5 分钟缓冲 + 主动调度 | 30 秒刷新缓冲 |
| **MDM 密钥** | 3 平台路径 | `requirements.toml` |
| **多后端** | Anthropic / Bedrock / Vertex / Foundry | OpenAI / Ollama / LM Studio |

**Claude Code 的 PKCE 认证链：**

Claude Code 的 OAuth 实现包含 6 级解析链，用于确定认证方式：
1. 环境变量（`ANTHROPIC_API_KEY`）
2. OAuth Token（Keychain / 文件）
3. MDM 托管密钥
4. Bedrock / Vertex 认证
5. Foundry 认证
6. 交互式 OAuth 登录

Token 刷新采用 5 分钟缓冲策略——在 Token 过期前 5 分钟主动触发刷新，避免请求失败。

**Codex CLI 的认证系统：**

Codex CLI 支持两种主要认证方式：
1. **API Key**：通过 `OPENAI_API_KEY` 环境变量或配置文件
2. **ChatGPT 账户**：通过设备码流程登录，支持 `keyring-store` 跨平台密钥存储（macOS Keychain、Linux Secret Service、Windows Credential Manager、freedesktop）

Token 刷新采用更激进的 30 秒缓冲策略，减少认证失败的概率。

## 4.9 遥测对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **遥测框架** | OTel + Statsig + GrowthBook + Sentry | 内置 OTel + `AnalyticsEventsClient` |
| **性能指标** | 8 种（API 延迟、Token 使用、工具执行时间等） | 5 种（请求延迟、Token 计数等） |
| **业务事件** | 5 种（会话开始/结束、工具使用等） | 6+ 种（会话、工具、沙箱、Agent 事件等） |
| **Prompt 脱敏** | 默认脱敏 | 默认脱敏 |
| **禁用方式** | `DISABLE_TELEMETRY=1` | `LLM_CLI_TELEMETRY_DISABLED=1` |
| **错误追踪** | Sentry | 内置错误报告 |

**Claude Code 的遥测体系：**

Claude Code 使用了业界最丰富的遥测技术栈：
- **OpenTelemetry**：标准化指标收集和导出
- **Statsig**：实时特性门和 A/B 测试
- **GrowthBook**：特性标志管理和渐进式功能发布
- **Sentry**：错误追踪和崩溃报告

这种多框架组合提供了强大的可观测性，但也增加了系统复杂性和潜在的隐私顾虑。

**Codex CLI 的遥测体系：**

Codex CLI 的遥测更加精简，主要依赖内置的 OpenTelemetry 实现和自定义的 `AnalyticsEventsClient`。作为开源项目，Codex CLI 的遥测默认脱敏且可完全禁用，更符合开源社区对隐私的期望。

## 4.10 IDE 集成对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **通信协议** | Bridge 协议（33 文件实现）+ JWT 认证 | `app-server` JSON-RPC 2.0 |
| **VS Code** | `anthropic.claude-code` 扩展 | `openai.chatgpt` 扩展 |
| **JetBrains** | Marketplace 插件 | 无公开信息 |
| **实时同步** | 光标位置 / 文件内容 / Git 状态 / 调试信息 | Thread / Turn / Item 原语 |
| **LSP 集成** | 内置 LSP 客户端 | 依赖 IDE 侧实现 |
| **Neovim** | 社区插件 | 社区插件 |

**Claude Code 的 Bridge 协议：**

Claude Code 的 IDE 集成通过 Bridge 协议实现，这是一个自定义的双向通信协议，包含 33 个文件的完整实现。Bridge 支持：
- JWT 认证确保通信安全
- 实时同步光标位置、文件内容、Git 状态
- 调试信息集成（断点、变量查看）
- 内置 LSP 客户端提供代码智能

**Codex CLI 的 app-server：**

Codex CLI 通过 `app-server` 提供 JSON-RPC 2.0 接口，使用 Thread/Turn/Item 三级原语组织对话。这种设计更加标准化，但功能相对简单——不包含光标同步和调试集成。

## 4.11 错误处理对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **重试策略** | 指数退避 + 抖动（`withRetry`） | 指数退避 + API `Duration` 头 |
| **错误分类** | `classifyError` 6 类 | `CodexErr` 枚举 15+ 变体 |
| **上下文溢出** | `reactiveCompact()` — 压缩旧消息 | `ContextWindowExceeded` — 移除最旧历史项 |
| **中止传播** | `siblingAbortController` | `CancellationToken` |
| **工具失败** | 错误消息反馈 + 同批取消 | 错误消息反馈给模型 |

**Claude Code 的错误分类（6 类）：**

1. **Rate Limit**：速率限制，等待后重试
2. **Auth Error**：认证失败，提示重新登录
3. **Server Error**：服务端错误，指数退避重试
4. **Context Too Long**：上下文过长，触发压缩
5. **Network Error**：网络错误，自动重试
6. **Unknown Error**：未知错误，记录日志

**Codex CLI 的 CodexErr 枚举（15+ 变体）：**

Codex CLI 的错误类型更加精细，包括：
- `ContextWindowExceeded` — 上下文窗口溢出
- `SandboxError` — 沙箱执行失败
- `ToolExecutionError` — 工具执行错误
- `ApiError` — API 调用错误
- `ConfigError` — 配置错误
- `AuthError` — 认证错误
- `RateLimitError` — 速率限制
- `NetworkError` — 网络错误
- `TimeoutError` — 超时
- `InvalidInput` — 无效输入
- 等 15+ 种变体

Rust 的 `enum` 类型系统确保所有错误变体都被显式处理，避免了未处理错误的遗漏。

## 4.12 技术栈与工程实践差异

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **语言** | TypeScript（Bun 运行时） | Rust（Edition 2024） |
| **代码规模** | ~50 万行，1884 个 TS 文件 | ~8 万行 Rust，60+ Crate |
| **模块化** | 目录 + 文件组织 | 60+ Crate 微服务化 |
| **UI 框架** | React + Ink（声明式终端 UI） | Ratatui（命令式终端 UI） |
| **构建** | Bun bundle（特性标志 DCE） | Bazel 9（CI）+ Cargo（开发） |
| **可复现构建** | 无 | Nix（`flake.nix`） |
| **类型安全** | TypeScript + Zod 运行时验证 | Rust 编译时保证 |
| **性能** | 依赖 Bun 运行时优化 | Rust 原生性能（零成本抽象） |
| **包管理** | Bun 内置 | Cargo + pnpm（monorepo） |
| **代码高亮** | 内置实现 | tree-sitter |
| **终端样式** | Chalk | crossterm |
| **异步运行时** | Bun 内置事件循环 | Tokio 1 |
| **HTTP 客户端** | Bun 内置 fetch | reqwest 0.12 |

**深入分析：**

**语言选择的影响：**

Claude Code 选择 TypeScript + Bun 是一个务实的决定——TypeScript 拥有庞大的生态系统和开发者社区，React/Ink 提供了成熟的终端 UI 方案，Bun 提供了出色的运行时性能。代价是 ~50 万行的代码规模（相比 Rust 实现的 ~8 万行），这反映了 TypeScript 在表达力和简洁性之间的权衡。

Codex CLI 选择 Rust 是一个面向未来的决定——Rust 的内存安全保证、零成本抽象和原生性能使其非常适合安全敏感的系统级工具。60+ Crate 的微服务化架构在 Rust 中自然实现（得益于 Cargo 的 workspace 支持），每个 Crate 职责单一，编译隔离，依赖关系清晰。~8 万行 Rust 代码实现了与 ~50 万行 TypeScript 相当的功能，这得益于 Rust 的表达力和标准库的丰富性。

**构建系统差异：**

Claude Code 使用 Bun bundle 进行构建，通过 `bun:bundle` 的特性标志实现死代码消除（DCE）。这种方式简单高效，但不支持跨平台编译和可复现构建。

Codex CLI 采用双构建系统策略——Bazel 用于生产级 CI/CD 和跨平台编译，Cargo 用于日常开发。此外，Nix 的 `flake.nix` 提供了完全可复现的构建环境，确保在任何机器上都能构建出完全相同的二进制。

## 4.13 综合对比表

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **安全哲学** | 权限治理（信任权限系统） | OS 级沙箱（不信任 AI） |
| **Agent Loop** | 流式异步生成器，双层循环 | 提交-事件模式，无状态循环 |
| **工具并发** | 只读并行（最多 10 个），写串行 | 全部串行（沙箱约束） |
| **文件编辑** | 搜索并替换（search-and-replace） | 统一差异补丁（unified diff） |
| **上下文压缩** | 四层渐进式压缩 + LRU 缓存 | 本地 + 远程压缩 + Token 截断 |
| **持久记忆** | MEMORY.md + Dream Task 后台整合 | AGENTS.md + rollout 持久化 |
| **多 Agent** | 三级架构（子 Agent / 协调器 / 团队） | 完整多 Agent + 守护者模式 + 批量 CSV |
| **代码规模** | ~50 万行 TypeScript | ~8 万行 Rust |
| **开源** | 闭源 | Apache 2.0 |
| **提示词缓存** | 动态边界分割 + Fork 前缀共享 | 精确前缀匹配 + Zstd 压缩 |
| **配置系统** | JSON 5 级优先级 | TOML 5 级优先级 |
| **IDE 集成** | Bridge 协议（33 文件）+ JWT | app-server JSON-RPC 2.0 |
| **遥测** | OTel + Statsig + GrowthBook + Sentry | 内置 OTel + AnalyticsEventsClient |
| **错误处理** | 6 类错误分类 | 15+ 枚举变体 |
| **可复现构建** | 无 | Nix 支持 |
| **模型支持** | Claude 系列（4 后端） | OpenAI + Ollama + LM Studio |

---

# 关键设计模式总结

## 5.1 Claude Code 核心设计模式

| 设计模式 | 实现位置 | 说明 |
|----------|----------|------|
| **流式异步生成器** | `query.ts` | Agent 循环核心，yield 事件而非批量返回，实现实时流式交互。这是 Claude Code 区别于传统请求-响应模式的关键设计——生成器模型天然支持背压、取消和流式消费 |
| **工具使用循环** | QueryEngine | 模型提议工具调用 → 执行 → 反馈结果，经典 ReAct 模式的流式变体。通过 `StreamingToolExecutor` 在模型生成时就开始执行已完成的工具调用，显著减少端到端延迟 |
| **分层上下文** | context.ts | 系统提示 + CLAUDE.md + 对话 + 压缩，多层上下文叠加。通过 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 标记分离静态和动态内容，最大化 API 缓存命中率 |
| **权限门控** | utils/permissions/ | 每个工具调用经过多层权限管道（模式 → 规则 → Hook → LLM 分类器 → 用户确认），灵活但复杂。7 层安全机制提供纵深防御 |
| **特性门控** | GrowthBook + bun:bundle | 运行时特性标志 + 构建时死代码消除，支持渐进式功能发布。未激活的特性在构建时被完全剥离，零运行时开销 |
| **延迟加载** | ToolSearchTool | 100+ MCP 工具按需加载 schema，大幅节省 token。系统提示仅包含工具名称列表，模型需要时再查询完整 schema |
| **多 Agent 隔离** | AgentTool | 克隆文件缓存、独立 AbortController、过滤工具池。Fork 机制共享前缀最大化 API 缓存命中 |
| **Fork 缓存优化** | AgentTool | 共享前缀最大化 API 提示缓存命中。多 Agent 场景下缓存命中率通常 >80% |
| **Hook 可扩展性** | hooks/ | 5 种 Hook 类型（Command/Prompt/Agent/HTTP/Function）覆盖 13 个生命周期事件。是 Claude Code 可扩展性的骨干 |
| **声明式终端 UI** | React/Ink | 140+ 组件、70+ Hooks、完整 React 生态模式。声明式渲染模型简化了复杂的终端 UI 开发 |
| **梦境任务** | autoDream/ | 后台记忆整合，会话学习持久化。在用户空闲时自动回顾会话并更新记忆文件，实现跨会话学习 |
| **LRU 文件缓存** | QueryEngine | 100 文件 / 25MB 上限，去重读取和变更检测。跨轮次跟踪文件内容，避免冗余读取 |

**深入分析：**

Claude Code 的设计模式体现了"**功能优先**"的哲学——通过丰富的设计模式实现尽可能多的功能，即使这增加了系统复杂性。流式异步生成器是最核心的模式，它不仅驱动了 Agent 循环，还影响了整个系统的架构——UI 层通过消费生成器事件来更新界面，遥测系统通过监听事件来收集指标，日志系统通过拦截事件来记录操作。

特性门控模式（GrowthBook + bun:bundle）是 Claude Code 作为闭源产品的独特优势——可以在不发布新版本的情况下远程控制功能的可用性，这对于快速迭代和 A/B 测试至关重要。

## 5.2 Codex CLI 核心设计模式

| 设计模式 | 实现位置 | 说明 |
|----------|----------|------|
| **Crate 微服务化** | codex-rs/ | 60+ Crate 高粒度模块化，每个 Crate 职责单一。Cargo workspace 提供天然的编译隔离和依赖管理 |
| **事件驱动架构** | codex.rs | 事件循环通过 Submission Queue / Event Queue 将模型响应、工具执行、用户输入解耦。Rust 的所有权模型确保通道传递的安全性 |
| **委托模式** | codex_delegate.rs | 将具体操作委托给专门的子系统处理（沙箱、Guardian、上下文管理器），保持核心循环的简洁性 |
| **策略模式** | sandboxing/ | 沙箱策略、执行策略、审批策略均可配置和替换。平台抽象层屏蔽 macOS/Linux/Windows 差异 |
| **Guardian 守护者** | guardian/ | 安全中间层，决定哪些操作可以自动执行。与沙箱配合提供双层安全保障 |
| **平台抽象层** | sandboxing/ | 屏蔽 macOS/Linux/Windows 差异，上层代码无需关心平台细节。每个平台有独立的 Crate 实现 |
| **无状态请求** | client.rs | 每个请求携带完整历史，支持零数据保留配置。简化服务端逻辑，提高可恢复性 |
| **提示词前缀缓存** | client.rs | 后续请求是前一次的精确前缀，最大化缓存命中率。无状态设计天然适合前缀缓存 |
| **双构建系统** | BUILD.bazel + Cargo.toml | Bazel 用于生产 CI/CD 和跨平台编译，Cargo 用于日常开发。Nix 提供可复现构建环境 |
| **渐进式迁移** | tools/ crate | 从 core 渐进式提取共享代码到独立 Crate，避免大规模重构风险。体现了 Rust 生态中常见的演进式架构方法 |

**深入分析：**

Codex CLI 的设计模式体现了"**安全优先**"的哲学——每个设计决策都以安全为第一考量。Crate 微服务化不仅是模块化的需要，更是安全隔离的需要——沙箱相关的 Crate（`linux-sandbox`、`windows-sandbox-rs`、`process-hardening`）与核心逻辑完全隔离，减少了安全漏洞的 blast radius。

无状态请求模式是 Codex CLI 最独特的设计选择之一。在大多数 Agent 系统都采用有状态设计的背景下，Codex CLI 的无状态方案看似"倒退"，但实际上带来了显著的优势：服务端无需维护会话状态、进程崩溃可恢复、支持零数据保留配置。提示词前缀缓存弥补了无状态设计的性能劣势。

## 5.3 可借鉴的架构思想

### 1. 分层状态管理（Claude Code）

将进程级（全局单例、基础设施）、会话级（UI、Hooks）、轮次级（Query、Services、Tools）状态分离管理，避免生命周期混乱。这种分层思想不仅适用于 AI 编码助手，也适用于任何需要管理多种生命周期状态的复杂应用。Claude Code 的六层架构（UI → Hooks → State → Query → Services → Tools）是一个优秀的分层设计参考。

### 2. OS 级沙箱（Codex CLI）

安全由操作系统内核保证而非软件层，更可靠且不可绕过。这是安全设计的黄金原则——将安全边界放在尽可能底层的可信组件中。对于任何需要处理不可信输入的系统（不仅仅是 AI），OS 级沙箱都是值得考虑的安全方案。

### 3. 延迟加载工具 Schema（Claude Code）

在工具数量庞大时有效节省 token，避免系统提示膨胀。这个模式可以推广到任何 LLM 应用——当需要向模型提供大量结构化信息时，按需加载比一次性提供更高效。ToolSearchTool 的"先发现后使用"模式是一个通用的 LLM 优化策略。

### 4. Crate 微服务化（Codex CLI）

高粒度模块化带来更好的编译隔离和职责清晰。Rust 的 Cargo workspace 天然支持这种模式，但核心理念——将大型系统分解为职责单一的小模块——适用于任何语言。Codex CLI 的 60+ Crate 组织方式展示了如何在 Rust 中实现"微服务化"的单体应用。

### 5. 渐进式压缩（Claude Code）

四层压缩策略优雅地处理上下文窗口限制。从微压缩（清除旧工具结果）到自动压缩（LLM 摘要）到会话记忆压缩（提取到持久文件）到反应式压缩（API 错误触发），每一层处理不同严重程度的上下文压力。这种渐进式策略可以推广到任何需要管理有限资源的系统。

### 6. 无状态请求 + 前缀缓存（Codex CLI）

简化服务端逻辑同时保持性能。这个模式的核心洞察是：无状态设计带来的可恢复性和可扩展性优势，可以通过前缀缓存来弥补性能劣势。对于任何需要与 LLM API 交互的系统，这都是值得考虑的架构选择。

### 7. Fork 缓存优化（Claude Code）

多 Agent 场景下最大化 API 缓存利用率。当需要从同一上下文生成多个变体（如多个子 Agent、多个候选方案）时，Fork 机制确保它们共享相同的前缀，仅计算不同的后缀。这个模式可以推广到任何需要批量调用 LLM 的场景。

### 8. 平台抽象沙箱（Codex CLI）

统一接口屏蔽平台差异，上层代码无需关心底层实现。`sandboxing` Crate 定义了统一的沙箱 trait，`linux-sandbox`、`windows-sandbox-rs` 等平台 Crate 提供具体实现。这种"接口-实现分离"的模式是跨平台开发的最佳实践。

## 5.4 架构演进趋势

### 5.4.1 从 TypeScript 到 Rust

Codex CLI 从 TypeScript 迁移到 Rust 的决策反映了 AI 编码助手领域的一个重要趋势：**对性能和安全的要求正在推动语言层面的升级**。

**迁移动机分析：**
- **性能**：Rust 的零成本抽象和原生性能对于需要频繁执行沙箱操作、文件 I/O 和网络请求的 AI 编码助手至关重要
- **安全性**：Rust 的内存安全保证（所有权系统、借用检查器）在系统级编程中提供了编译时安全保证，减少了运行时漏洞
- **并发**：Rust 的 `Send + Sync` trait 和无数据竞争的并发模型天然适合多 Agent 系统的并发需求
- **代码规模**：~8 万行 Rust 实现了 ~50 万行 TypeScript 的等效功能，代码维护成本显著降低

**迁移代价：**
- 开发效率降低（Rust 的学习曲线和编译时间）
- 生态系统较小（相比 npm 的百万级包，Cargo 的包数量有限）
- UI 开发不够便捷（Ratatui vs React/Ink 的开发体验差距）

**趋势预测：** 未来可能会有更多 AI 编码助手采用 Rust 或其他系统级语言，特别是那些对安全性和性能有高要求的企业级产品。但 TypeScript/JavaScript 由于其庞大的生态系统和低学习曲线，仍将是快速原型开发和个人工具的首选。

### 5.4.2 从权限治理到 OS 沙箱

安全模型的演进方向正在从**软件层权限治理**向**OS 级沙箱隔离**转变。

**演进驱动因素：**
1. **提示注入威胁升级**：随着 AI 模型能力的增强，提示注入攻击的复杂性和成功率也在增加，软件层的安全措施越来越难以应对
2. **企业安全合规要求**：企业客户需要"确定性安全"——安全保证不依赖于软件规则的正确性，而是依赖于操作系统内核的安全属性
3. **监管压力**：各国对 AI 安全的监管要求正在趋严，OS 级沙箱提供了更容易审计和验证的安全边界

**Claude Code 的潜在演进方向：**
- 引入轻量级沙箱（如 macOS Seatbelt）作为权限管道的补充层
- 将 MCP 工具纳入沙箱保护范围
- 为危险操作（如 Bash 命令）提供可选的沙箱模式

**Codex CLI 的潜在演进方向：**
- 细化沙箱策略粒度（如按工具类型配置不同的沙箱规则）
- 支持自定义沙箱配置文件
- 为 MCP 工具提供沙箱保护

### 5.4.3 从有状态到无状态

Agent 循环的状态管理趋势正在从**有状态设计**向**无状态设计**演进。

**有状态设计的优势（Claude Code）：**
- 内存效率高（不需要每次请求都序列化完整历史）
- 可以维护跨轮次的文件缓存和状态
- 更低的网络开销

**无状态设计的优势（Codex CLI）：**
- 服务端无需维护会话状态
- 进程崩溃可恢复
- 支持零数据保留配置
- 更容易实现分布式部署

**趋势预测：** 未来的 AI 编码助手可能会采用**混合状态管理**策略——核心 Agent 循环保持无状态（便于恢复和分布式部署），但在本地维护缓存层（文件缓存、压缩结果缓存）以优化性能。这种混合策略结合了两者的优势。

### 5.4.4 MCP 协议标准化

MCP（Model Context Protocol）正在成为 AI 工具生态的**事实标准协议**，两个项目都已支持 MCP 集成。

**当前状态：**
- **Claude Code**：自研 MCP 实现，支持 4 种传输协议（stdio、SSE/HTTP、WebSocket、local），5 种配置作用域
- **Codex CLI**：基于 `rmcp 0.12` 实现，MCP 相关代码分布在 3 个 Crate（`mcp-server`、`rmcp-client`、`mcp-types`）

**标准化趋势：**
1. **工具接口统一**：MCP 提供了标准化的工具描述和调用协议，使工具可以在不同的 AI 助手之间复用
2. **传输协议收敛**：从多种传输协议向少数标准协议收敛（stdio 用于本地工具，HTTP/SSE 用于远程工具）
3. **安全标准化**：MCP 工具的安全保护正在成为共识——两个项目都认识到 MCP 工具是安全盲点，需要额外的保护机制

**未来展望：**
- MCP 协议可能演进为包含安全元数据的版本（如工具权限声明、沙箱要求）
- 工具市场可能出现——标准化的 MCP 协议使工具的分发和发现更加容易
- MCP 可能成为 AI Agent 互操作的通用协议（类似于 HTTP 之于 Web）

---

# 结论

## 6.1 核心发现总结

通过对 Claude Code 和 Codex CLI 的源码级深度分析，我们得出以下核心发现：

**1. 安全哲学的分野是两者最根本的架构区别。** Claude Code 采用"信任但验证"的软件层权限治理，Codex CLI 采用"零信任"的 OS 级沙箱隔离。两者代表了 AI 编码助手安全设计的两个极端，没有绝对的最优解——选择取决于使用场景和安全需求。

**2. Agent Loop 的设计反映了技术栈的深层影响。** Claude Code 的流式异步生成器（TypeScript AsyncGenerator）和 Codex CLI 的提交-事件模式（Rust Channel）都是各自技术栈下的最优选择。Claude Code 的 StreamingToolExecutor（边生成边执行）是一个显著的性能优化，而 Codex CLI 的无状态设计带来了更好的可恢复性。

**3. 工具系统的设计选择体现了不同的工程权衡。** Claude Code 的 search-and-replace 编辑方式降低了模型认知负担但增加了 token 消耗；Codex CLI 的 unified diff patch 方式更适合代码审查但增加了模型出错概率。Claude Code 的 40+ 工具提供了更丰富的功能，Codex CLI 的 25+ 工具保持了系统的简洁性。

**4. 上下文管理是 AI 编码助手的核心竞争力。** Claude Code 的四层渐进式压缩是目前业界最完善的上下文管理方案，Dream Task 的跨会话学习机制更是独树一帜。Codex CLI 的无状态设计 + 前缀缓存是一个简洁有效的替代方案。

**5. 语言选择对架构有深远影响。** TypeScript 的 ~50 万行 vs Rust 的 ~8 万行实现了等效功能，这不仅是代码量的差异，更是架构复杂度、维护成本和安全保证的差异。Rust 的类型系统和所有权模型在系统级编程中提供了不可替代的优势。

## 6.2 适用场景建议

### Claude Code 更适合：

- **个人开发者和小团队**：丰富的工具集和灵活的权限系统适合快速迭代开发
- **需要深度 IDE 集成的场景**：Bridge 协议提供了业界最完善的 IDE 集成（光标同步、调试信息、LSP）
- **需要高级多 Agent 协作的场景**：三级多 Agent 架构（子 Agent / 协调器 / 团队）支持复杂的工作流
- **需要跨会话学习的场景**：Dream Task 后台记忆整合 + MEMORY.md 持久记忆
- **使用 Claude 模型的用户**：对 Claude 系列模型有深度优化，支持 4 个 API 后端
- **需要丰富扩展性的场景**：5 种 Hook 类型、技能系统、插件系统提供了强大的可扩展性

### Codex CLI 更适合：

- **安全敏感的企业环境**：OS 级沙箱提供了不可绕过的安全保证
- **需要多模型支持的场景**：支持 OpenAI、Ollama、LM Studio 等多个模型提供商
- **需要批量自动化处理的场景**：`spawn_agents_on_csv` 支持 CSV 驱动的批量 Agent 任务
- **开源偏好者**：Apache 2.0 许可证，完全透明的代码和决策过程
- **需要可复现构建的场景**：Nix + Bazel 提供了完全可复现的构建环境
- **需要本地模型运行的场景**：原生支持 Ollama 和 LM Studio 本地模型

## 6.3 未来展望

基于对两个项目的深度分析，我们预测 AI 编码助手的架构将在以下方向演进：

**1. 安全模型融合：** 未来的 AI 编码助手可能会融合软件层权限治理和 OS 级沙箱的优势——在沙箱内运行工具的同时，保留精细的权限控制。这种"纵深防御"策略将提供更强的安全保障。

**2. 多模型支持成为标配：** 随着 LLM 市场的成熟，支持多个模型提供商将成为 AI 编码助手的基本要求。模型路由和自动选择（根据任务类型选择最合适的模型）将成为重要特性。

**3. MCP 生态成熟：** MCP 协议将成为 AI 工具生态的事实标准，工具市场和安全标准将逐步建立。AI 编码助手将更多地作为"工具编排器"而非"工具提供者"。

**4. 无状态架构普及：** 无状态 Agent 循环 + 本地缓存层的混合架构将成为主流，平衡性能和可恢复性。

**5. 多 Agent 编排标准化：** 多 Agent 系统将从实验性功能走向标准化，Agent 间通信协议、任务分配策略和结果聚合机制将逐步统一。

**6. 语言层面持续升级：** 更多 AI 编码助手将采用 Rust 或其他系统级语言，以获得更好的性能、安全性和并发支持。但 TypeScript/JavaScript 由于其生态优势，仍将在快速原型开发领域保持主导地位。

---

# 参考来源

1. Claude Code GitHub 仓库: https://github.com/anthropics/claude-code
2. Codex CLI GitHub 仓库: https://github.com/openai/codex
3. Haseeb-Qureshi, "AI Coding Agent Architecture Analysis: Claude Code vs Codex vs Cline vs OpenCode" (GitHub Gist, 2026.03)
4. yanchuk, "Claude Code Agent — Complete Architecture Deep Dive" (GitHub Gist, 2026.03)
5. OpenAI 官方博客, "展开 Codex 智能代理回圈" (2026.01)
6. Corti, "Claude Code vs OpenAI Codex: A Technical Comparison" (2025.05)
7. 6551Team Claude Code 设计指南: https://github.com/6551Team/claude-code-design-guide
8. weisberg, "The Claude Code Architecture and Ecosystem Exhaustive Technical Guide"
9. OpenReplay, "OpenAI Codex vs. Claude Code: Which CLI AI tool is best for coding?" (2025.07)
10. Anthropic 官方文档, Claude Code 安全白皮书
11. OpenAI 官方文档, Codex CLI 安全架构说明
12. Landlock 官方文档: https://www.kernel.org/doc/html/latest/security/landlock.html
13. Apple Seatbelt 官方文档: https://developer.apple.com/documentation/security/app_sandbox
14. MCP 协议规范: https://modelcontextprotocol.io
15. Rust Edition 2024 发布说明: https://blog.rust-lang.org
