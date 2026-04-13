# 设计模式与结论

## Claude Code 核心设计模式

| 设计模式 | 实现位置 | 说明 |
|----------|----------|------|
| **流式异步生成器** | `query.ts` | Agent 循环核心，yield 事件而非批量返回，实现实时流式交互。这是 Claude Code 区别于传统请求-响应模式的关键设计——生成器模型天然支持背压、取消和流式消费 |
| **工具使用循环** | QueryEngine | 模型提议工具调用 -> 执行 -> 反馈结果，经典 ReAct 模式的流式变体。通过 `StreamingToolExecutor` 在模型生成时就开始执行已完成的工具调用，显著减少端到端延迟 |
| **分层上下文** | context.ts | 系统提示 + CLAUDE.md + 对话 + 压缩，多层上下文叠加。通过 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 标记分离静态和动态内容，最大化 API 缓存命中率 |
| **权限门控** | utils/permissions/ | 每个工具调用经过多层权限管道（模式 -> 规则 -> Hook -> LLM 分类器 -> 用户确认），灵活但复杂。7 层安全机制提供纵深防御 |
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

---

## Codex CLI 核心设计模式

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

---

## 可借鉴的架构思想

### 1. 分层状态管理（Claude Code）

将进程级（全局单例、基础设施）、会话级（UI、Hooks）、轮次级（Query、Services、Tools）状态分离管理，避免生命周期混乱。Claude Code 的六层架构（UI -> Hooks -> State -> Query -> Services -> Tools）是一个优秀的分层设计参考。

### 2. OS 级沙箱（Codex CLI）

安全由操作系统内核保证而非软件层，更可靠且不可绕过。这是安全设计的黄金原则——将安全边界放在尽可能底层的可信组件中。

### 3. 延迟加载工具 Schema（Claude Code）

在工具数量庞大时有效节省 token，避免系统提示膨胀。ToolSearchTool 的"先发现后使用"模式是一个通用的 LLM 优化策略。

### 4. Crate 微服务化（Codex CLI）

高粒度模块化带来更好的编译隔离和职责清晰。60+ Crate 的组织方式展示了如何在 Rust 中实现"微服务化"的单体应用。

### 5. 渐进式压缩（Claude Code）

四层压缩策略优雅地处理上下文窗口限制。从微压缩到自动压缩到会话记忆压缩到反应式压缩，每一层处理不同严重程度的上下文压力。

### 6. 无状态请求 + 前缀缓存（Codex CLI）

简化服务端逻辑同时保持性能。无状态设计带来的可恢复性和可扩展性优势，可以通过前缀缓存来弥补性能劣势。

### 7. Fork 缓存优化（Claude Code）

多 Agent 场景下最大化 API 缓存利用率。Fork 机制确保子 Agent 共享相同的前缀，仅计算不同的后缀。

### 8. 平台抽象沙箱（Codex CLI）

统一接口屏蔽平台差异，上层代码无需关心底层实现。`sandboxing` Crate 定义统一的沙箱 trait，各平台 Crate 提供具体实现。

---

## 架构演进趋势

### 从 TypeScript 到 Rust

Codex CLI 从 TypeScript 迁移到 Rust 的决策反映了 AI 编码助手领域的一个重要趋势：**对性能和安全的要求正在推动语言层面的升级**。

- **性能**：Rust 的零成本抽象和原生性能对于频繁执行沙箱操作、文件 I/O 和网络请求的 AI 编码助手至关重要
- **安全性**：Rust 的内存安全保证在系统级编程中提供了编译时安全保证
- **并发**：Rust 的 `Send + Sync` trait 和无数据竞争的并发模型天然适合多 Agent 系统
- **代码规模**：~8 万行 Rust 实现了 ~50 万行 TypeScript 的等效功能

### 从权限治理到 OS 沙箱

安全模型的演进方向正在从**软件层权限治理**向**OS 级沙箱隔离**转变。

- **Claude Code 潜在演进**：引入轻量级沙箱（如 macOS Seatbelt）、将 MCP 工具纳入沙箱保护
- **Codex CLI 潜在演进**：细化沙箱策略粒度、支持自定义沙箱配置文件

### 从有状态到无状态

Agent 循环的状态管理趋势正在从**有状态设计**向**无状态设计**演进。未来的 AI 编码助手可能会采用**混合状态管理**策略——核心 Agent 循环保持无状态，但在本地维护缓存层以优化性能。

### MCP 协议标准化

MCP（Model Context Protocol）正在成为 AI 工具生态的**事实标准协议**。

- **工具接口统一**：MCP 提供了标准化的工具描述和调用协议
- **传输协议收敛**：从多种传输协议向少数标准协议收敛
- **安全标准化**：MCP 工具的安全保护正在成为共识
- **未来展望**：工具市场可能出现，MCP 可能成为 AI Agent 互操作的通用协议

---

## 综合对比

### 核心发现总结

**1. 安全哲学的分野是两者最根本的架构区别。** Claude Code 采用"信任但验证"的软件层权限治理，Codex CLI 采用"零信任"的 OS 级沙箱隔离。两者代表了 AI 编码助手安全设计的两个极端。

**2. Agent Loop 的设计反映了技术栈的深层影响。** Claude Code 的流式异步生成器（TypeScript AsyncGenerator）和 Codex CLI 的提交-事件模式（Rust Channel）都是各自技术栈下的最优选择。

**3. 工具系统的设计选择体现了不同的工程权衡。** Claude Code 的 search-and-replace 编辑方式降低了模型认知负担但增加了 token 消耗；Codex CLI 的 unified diff patch 方式更适合代码审查但增加了模型出错概率。

**4. 上下文管理是 AI 编码助手的核心竞争力。** Claude Code 的四层渐进式压缩是目前业界最完善的上下文管理方案，Dream Task 的跨会话学习机制更是独树一帜。

**5. 语言选择对架构有深远影响。** TypeScript 的 ~50 万行 vs Rust 的 ~8 万行实现了等效功能，这不仅是代码量的差异，更是架构复杂度、维护成本和安全保证的差异。

### 综合对比表

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

### 适用场景建议

#### Claude Code 更适合

- **个人开发者和小团队**：丰富的工具集和灵活的权限系统适合快速迭代开发
- **需要深度 IDE 集成的场景**：Bridge 协议提供了业界最完善的 IDE 集成（光标同步、调试信息、LSP）
- **需要高级多 Agent 协作的场景**：三级多 Agent 架构（子 Agent / 协调器 / 团队）支持复杂的工作流
- **需要跨会话学习的场景**：Dream Task 后台记忆整合 + MEMORY.md 持久记忆
- **使用 Claude 模型的用户**：对 Claude 系列模型有深度优化，支持 4 个 API 后端
- **需要丰富扩展性的场景**：5 种 Hook 类型、技能系统、插件系统提供了强大的可扩展性

#### Codex CLI 更适合

- **安全敏感的企业环境**：OS 级沙箱提供了不可绕过的安全保证
- **需要多模型支持的场景**：支持 OpenAI、Ollama、LM Studio 等多个模型提供商
- **需要批量自动化处理的场景**：`spawn_agents_on_csv` 支持 CSV 驱动的批量 Agent 任务
- **开源偏好者**：Apache 2.0 许可证，完全透明的代码和决策过程
- **需要可复现构建的场景**：Nix + Bazel 提供了完全可复现的构建环境
- **需要本地模型运行的场景**：原生支持 Ollama 和 LM Studio 本地模型

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| 功能极为丰富（40+ 工具、87+ 命令、13 Hook 事件、5 Hook 类型） | 系统复杂度极高（~50 万行 TS），维护成本大 |
| 四层渐进式压缩 + Dream Task 跨会话学习，上下文管理业界领先 | 闭源，无法审计代码安全性和正确性 |
| Bridge 协议 + 内置 LSP 客户端，IDE 集成最完善 | 无 OS 级沙箱，安全性依赖软件层规则 |
| React/Ink 声明式 UI（140+ 组件、70+ Hooks），交互体验优秀 | 依赖 Bun 运行时，非原生二进制性能 |
| GrowthBook + bun:bundle 特性门控，支持远程 A/B 测试 | 无可复现构建（Nix），无跨平台编译 |
| 6 级认证解析链 + MDM 策略，企业部署友好 | Windows/Linux 密钥存储安全性不足 |
| 三层遥测架构（Statsig + Sentry + OTel），可观测性全面 | 遥测数据发送到第三方，隐私顾虑 |
| Claude Haiku 性价比高，Bedrock/Vertex 企业 API 灵活 | 无免费使用选项，无本地模型支持 |
| StreamingToolExecutor 边生成边执行，端到端延迟低 | 有状态设计，进程崩溃不可恢复 |
| Fork 缓存优化，多 Agent 缓存命中率 >80% | 权限管道复杂（7 层），配置和理解成本高 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| OS 级沙箱（Landlock/Seatbelt/Restricted Token），安全性不可绕过 | 功能相对精简（25+ 工具），扩展性不如 Claude Code |
| Apache 2.0 开源，代码完全透明，421+ 贡献者 | 无独立 Hook 系统，可扩展性受限 |
| 零依赖原生二进制，性能优秀（Rust 零成本抽象） | Ratatui 命令式 UI 开发效率低于 React/Ink |
| Nix + Bazel 可复现构建 + 跨平台编译 | 构建系统复杂（5 个工具），学习曲线陡峭 |
| 本地模型免费（Ollama/LM Studio），零 API 成本 | 本地模型需要 GPU 资源，质量不及云端 |
| 无状态请求设计，进程崩溃可恢复 | 无内置 LSP 客户端，代码智能依赖 IDE |
| keyring-store 4 平台密钥管理，安全性高 | 遥测架构简单，缺少 A/B 测试和错误追踪 |
| ChatGPT Plus $20/月入门门槛低 | o3 推理模型价格较高 |
| GPT-4.1 系列 1M 上下文窗口 | 无 Dream Task 跨会话学习机制 |
| 60+ Crate 微服务化，编译隔离，职责清晰 | 无 MDM 策略，企业部署能力弱 |
| 15+ 错误枚举变体，Rust 类型系统确保显式处理 | 无 Vim 模式，无斜杠命令系统 |
| WebRTC + Opus 实时语音，延迟 ~10-30ms | IDE 集成仅 VS Code 社区扩展 |
| 批量 CSV Agent 任务，自动化处理能力强 | 无 Plan 模式，无 Doctor 诊断界面 |
