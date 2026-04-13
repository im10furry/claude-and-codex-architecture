# 项目概览对比：Claude Code vs Codex CLI

## Claude Code 实现

### 基本信息

| 维度 | Claude Code |
|------|-------------|
| **开发商** | Anthropic |
| **仓库地址** | [github.com/anthropics/claude-code](https://github.com/anthropics/claude-code) |
| **许可证** | 闭源 |
| **主要语言** | TypeScript（严格模式） |
| **运行时** | Bun |
| **代码规模** | ~50 万行，1884 个 TS 文件 |
| **内置工具** | 40+ |
| **斜杠命令** | 87+ |
| **React Hooks** | 70+ |
| **后台服务** | 13 个子系统 |
| **API 后端** | Anthropic / Bedrock / Vertex / Foundry |

### 技术栈

| 组件 | Claude Code |
|------|-------------|
| **主语言** | TypeScript（Bun 运行时） |
| **UI 框架** | React + Ink（终端渲染） |
| **CLI 解析** | Commander.js |
| **异步运行时** | Bun 内置 |
| **HTTP 客户端** | 内置 fetch |
| **Schema 验证** | Zod v4 |
| **遥测** | OpenTelemetry + GrowthBook + Statsig + Sentry |
| **构建** | Bun bundle（特性标志死代码消除） |
| **代码高亮** | 内置 |
| **MCP 协议** | 自研实现（4 种传输） |
| **终端样式** | Chalk |

---

## Codex CLI 实现

### 基本信息

| 维度 | Codex CLI |
|------|-----------|
| **开发商** | OpenAI |
| **仓库地址** | [github.com/openai/codex](https://github.com/openai/codex) |
| **许可证** | Apache 2.0 开源 |
| **主要语言** | Rust（从 TypeScript 迁移） |
| **运行时** | 原生二进制 |
| **代码规模** | ~8 万行 Rust，60+ Crate |
| **内置工具** | 25+ |
| **斜杠命令** | N/A |
| **React Hooks** | N/A |
| **后台服务** | N/A |
| **API 后端** | OpenAI（支持 Ollama/LM Studio 本地模型） |
| **Stars** | 75,000+ |
| **贡献者** | 421+ |

### 技术栈

| 组件 | Codex CLI |
|------|-----------|
| **主语言** | Rust（Edition 2024） |
| **UI 框架** | Ratatui 0.29 + crossterm 0.28 |
| **CLI 解析** | clap 4 |
| **异步运行时** | Tokio 1 |
| **HTTP 客户端** | reqwest 0.12 |
| **Schema 验证** | serde + serde_json |
| **遥测** | 内置 OpenTelemetry SDK |
| **构建** | Bazel 9（CI）+ Cargo（开发） |
| **代码高亮** | tree-sitter |
| **MCP 协议** | rmcp 0.12 |
| **终端样式** | crossterm |
| **可复现构建** | Nix（flake.nix） |

---

## 对比分析

### 基本信息对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **开发商** | Anthropic | OpenAI |
| **开源状态** | 闭源 | Apache 2.0 开源 |
| **主要语言** | TypeScript | Rust |
| **运行时** | Bun | 原生二进制 |
| **代码规模** | ~50 万行 | ~8 万行 |
| **模块化** | 1884 个 TS 文件 | 60+ Crate |
| **内置工具** | 40+ | 25+ |
| **API 后端** | Anthropic / Bedrock / Vertex / Foundry | OpenAI + 本地模型 |
| **社区规模** | 未知（闭源） | 75K+ Stars, 421+ 贡献者 |

### 技术栈对比

| 组件 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **UI 框架** | React + Ink | Ratatui + crossterm |
| **CLI 解析** | Commander.js | clap 4 |
| **异步运行时** | Bun 内置 | Tokio 1 |
| **Schema 验证** | Zod v4 | serde + serde_json |
| **构建系统** | Bun bundle | Bazel 9 + Cargo |
| **MCP 协议** | 自研实现（4 种传输） | rmcp 0.12 |
| **可复现构建** | 无 | Nix flake |

### 计费模式对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **按量计费** | 按 token 计费 | 按 token 计费（API 模式） |
| **Claude Sonnet** | $3/1M input, $15/1M output | N/A |
| **Claude Opus** | $15/1M input, $75/1M output | N/A |
| **订阅模式** | N/A | ChatGPT Plus $20/月 |
| **本地模型** | 不支持 | 支持 Ollama/LM Studio（免费） |
| **企业方案** | Bedrock/Vertex 按云厂商计费 | ChatGPT Enterprise |
| **缓存优惠** | Prompt caching 减少输入成本 | 前缀缓存优化 |

### 安装方式对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **npm 安装** | `npm install -g @anthropic-ai/claude-code` | `npm install -g @openai/codex` |
| **Cargo 安装** | 不支持 | `cargo install codex-cli` |
| **预编译二进制** | 不支持 | 支持（GitHub Releases） |
| **依赖要求** | Node.js + Bun | 无（原生二进制） |
| **跨平台** | macOS/Linux/Windows (WSL) | macOS/Linux/Windows |

### 开源状态对比及影响

| 维度 | Claude Code（闭源） | Codex CLI（Apache 2.0 开源） |
|------|---------------------|---------------------------|
| **代码可见性** | 不可查看源码 | 完全透明 |
| **社区贡献** | 不可 | 421+ 贡献者活跃参与 |
| **自定义修改** | 不可能 | 自由 fork 和修改 |
| **安全审计** | 依赖厂商 | 社区 + 独立审计 |
| **学习价值** | 仅通过逆向分析 | 可直接学习架构设计 |
| **迭代速度** | 厂商主导 | 社区驱动 + 厂商支持 |
| **生态扩展** | 受限于官方 API | 可自由扩展和集成 |

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| 功能极其丰富：40+ 工具、87+ 斜杠命令、70+ Hooks | 闭源，无法审查或修改源码 |
| 多 API 后端支持（Anthropic/Bedrock/Vertex/Foundry） | 依赖 Bun 运行时，部署有额外要求 |
| React/Ink 终端 UI 体验成熟 | TypeScript 运行时性能不如原生二进制 |
| 四层渐进式压缩策略，上下文管理精细 | 代码规模庞大（~50 万行），维护复杂度高 |
| 持久记忆系统（CLAUDE.md + Dream Task） | 不支持本地模型，必须依赖 Anthropic API |
| Prompt caching 显著降低长对话成本 | 计费较高（Opus $15/$75 per 1M tokens） |
| MCP 协议自研实现，支持 4 种传输方式 | 社区生态受限，无法接受外部贡献 |
| 特性标志系统（GrowthBook）灵活控制功能发布 | 无可复现构建支持 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| Apache 2.0 完全开源，社区活跃（75K+ Stars） | 工具数量较少（25+ vs 40+） |
| Rust 原生二进制，性能优异、内存安全 | 从 TypeScript 迁移而来，部分设计仍遗留 TS 痕迹 |
| 支持本地模型（Ollama/LM Studio），可免费使用 | 仅支持 OpenAI API（不含 Bedrock/Vertex 等多云） |
| 60+ Crate 微服务化架构，编译隔离、职责单一 | 60+ Crate 带来较高的架构复杂度 |
| Nix flake 支持可复现构建 | 无持久记忆系统（无 CLAUDE.md 等价物） |
| Bazel 9 CI 构建系统，大型项目工程化成熟 | 压缩策略相对简单（自动压缩 + 截断） |
| 多入口模式（CLI/TUI/Exec）灵活适配不同场景 | 无斜杠命令、无 React Hooks 等高级交互特性 |
| 前缀缓存优化，无状态请求设计 | 上下文管理精度不如 Claude Code 四层策略 |
| WebSocket 双向通信支持，流式中断更优雅 | apply_patch 格式对模型生成质量要求较高 |
| 沙箱安全体系完善（Linux/Windows 平台适配） | UI 框架（Ratatui）功能不如 React/Ink 丰富 |
