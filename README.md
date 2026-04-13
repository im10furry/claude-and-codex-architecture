# Claude Code vs Codex CLI — 源码架构对比分析

> 本目录包含 Anthropic Claude Code 与 OpenAI Codex CLI 两个 AI 编码助手的源码级架构对比分析，按功能模块拆分为独立文件，每个文件均包含两个平台的同功能实现对比及优缺点分析。

---

## 文件索引

### 基础概览

| 文件 | 内容 | 核心对比点 |
|------|------|-----------|
| [00-overview.md](./00-overview.md) | 项目概览、计费、安装、开源策略 | TypeScript/Bun vs Rust、闭源 vs Apache 2.0、API 计费 vs 订阅+本地免费 |

### 核心架构

| 文件 | 内容 | 核心对比点 |
|------|------|-----------|
| [01-architecture.md](./01-architecture.md) | 整体架构设计 | 六层分层 vs 60+ Crate 微服务化、React/Ink vs Ratatui |
| [02-agent-loop.md](./02-agent-loop.md) | Agent 循环实现 | 流式异步生成器 vs Op/Event 提交-事件模式、有状态 vs 无状态 |
| [03-tool-system.md](./03-tool-system.md) | 工具系统 | 40+ vs 25+ 工具、search-and-replace vs unified diff、并发策略 |
| [04-context-memory.md](./04-context-memory.md) | 上下文与记忆管理 | 四层渐进式压缩 vs 自动压缩+截断、持久记忆策略 |

### 安全与权限

| 文件 | 内容 | 核心对比点 |
|------|------|-----------|
| [05-security-permission.md](./05-security-permission.md) | 安全与权限系统 | 软件层权限管道 vs OS 内核级沙箱、LLM 分类器 vs Guardian |

### 多 Agent 与编排

| 文件 | 内容 | 核心对比点 |
|------|------|-----------|
| [06-multi-agent.md](./06-multi-agent.md) | 多 Agent 系统 | 三级架构 vs 守护者模式、Fork 缓存 vs 沙箱隔离 |

### 配置与状态

| 文件 | 内容 | 核心对比点 |
|------|------|-----------|
| [07-config-state.md](./07-config-state.md) | 配置系统与状态管理 | JSON vs TOML、CLAUDE.md vs AGENTS.md、React Context vs Rust Mutex |

### 可靠性

| 文件 | 内容 | 核心对比点 |
|------|------|-----------|
| [08-error-session.md](./08-error-session.md) | 错误处理与会话恢复 | 6 类 vs 15+ 错误变体、JSONL vs JSONL+SQLite 双层存储 |

### 扩展能力

| 文件 | 内容 | 核心对比点 |
|------|------|-----------|
| [09-hook-skill-mcp.md](./09-hook-skill-mcp.md) | Hook、技能与 MCP 集成 | 13 事件 x 5 Hook 类型 vs 无独立 Hook、4 传输 vs 2 传输 |

### 运维与可观测性

| 文件 | 内容 | 核心对比点 |
|------|------|-----------|
| [10-auth-telemetry.md](./10-auth-telemetry.md) | 认证与遥测 | OAuth PKCE 6 级解析链 vs 3 种登录流程、三层 vs 单层遥测 |

### 开发者体验

| 文件 | 内容 | 核心对比点 |
|------|------|-----------|
| [11-ide-lsp.md](./11-ide-lsp.md) | IDE 集成与 LSP | Bridge 33 文件 vs app-server JSON-RPC、内置 LSP vs 依赖 IDE |
| [12-ui-ux.md](./12-ui-ux.md) | UI/UX 实现 | React+Ink 140 组件 vs Ratatui 命令式、Vim/语音/键绑定 |

### 工程实践

| 文件 | 内容 | 核心对比点 |
|------|------|-----------|
| [13-build-deploy.md](./13-build-deploy.md) | 构建系统与部署 | Bun bundle vs Bazel+Cargo+Nix、单一构建 vs 双构建系统 |

### 商业与生态

| 文件 | 内容 | 核心对比点 |
|------|------|-----------|
| [14-pricing-ecosystem.md](./14-pricing-ecosystem.md) | 计费与生态系统 | API token 计费 vs 订阅+本地免费、闭源 vs Apache 2.0 开源 |

### 总结

| 文件 | 内容 | 核心对比点 |
|------|------|-----------|
| [15-design-patterns.md](./15-design-patterns.md) | 设计模式、演进趋势与结论 | 12 种核心模式、4 大趋势、适用场景建议 |

---

## 快速对比速查表

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **语言/运行时** | TypeScript / Bun | Rust / 原生二进制 |
| **代码规模** | ~50 万行 | ~8 万行 |
| **UI 框架** | React + Ink（声明式） | Ratatui（命令式） |
| **Agent Loop** | 流式异步生成器 | Op/Event 提交-事件 |
| **工具数量** | 40+ | 25+ |
| **文件编辑** | search-and-replace | unified diff patch |
| **安全模型** | 多层权限管道 | OS 级沙箱 |
| **上下文压缩** | 四层渐进式 | 自动 + 截断 |
| **多 Agent** | 三级架构 | 守护者模式 |
| **配置格式** | JSON | TOML |
| **IDE 集成** | Bridge 协议 | app-server JSON-RPC |
| **LSP** | 内置客户端 | 无内置 |
| **构建** | Bun bundle | Bazel + Cargo |
| **许可证** | 闭源 | Apache 2.0 |
| **本地模型** | 不支持 | Ollama / LM Studio |
| **最低月费** | $0（API）/$100（Max） | $0（本地）/ $20（Plus） |

---

## 参考来源

1. [Claude Code GitHub](https://github.com/anthropics/claude-code)
2. [Codex CLI GitHub](https://github.com/openai/codex)
3. [Claude Code 架构深度分析 (Gist)](https://gist.github.com/yanchuk/0c47dd351c2805236e44ec3935e9095d)
4. [AI Coding Agent Architecture Analysis (Gist)](https://gist.github.com/)
5. [OpenAI 官方博客：展开 Codex 智能代理回圈](https://openai.com/zh-Hant-HK/index/unrolling-the-codex-agent-loop/)
6. [6551Team Claude Code 设计指南](https://github.com/6551Team/claude-code-design-guide)
7. [Claude Code 数据使用文档](https://docs.claude.com/en/docs/claude-code/data-usage)
8. [Anthropic API 定价](https://docs.anthropic.com/en/docs/about-claude/pricing)
9. [OpenAI API 定价](https://openai.com/api/pricing/)
