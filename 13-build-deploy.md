# 构建系统与部署对比

## Claude Code 实现

### Bun bundle 单一构建

Claude Code 使用 **Bun** 作为运行时和构建工具，采用 `bun:bundle` 进行单一构建：

```
┌─────────────────────────────────────────────────────────────────┐
│                 Claude Code 构建系统                              │
│                                                                 │
│  构建工具：Bun bundle                                           │
│  运行时：Bun（非 Node.js）                                      │
│  包管理：Bun 内置                                               │
│                                                                 │
│  构建流程：                                                      │
│  ┌──────────────────────────────────────────────────────┐      │
│  │  TypeScript 源码 (~50 万行, 1884 个 TS 文件)          │      │
│  │       │                                               │      │
│  │       ▼                                               │      │
│  │  bun:bundle 特性标志死代码消除 (DCE)                   │      │
│  │  - feature('VOICE_MODE') == false -> 剥离语音代码      │      │
│  │  - feature('BRIDGE_MODE') == false -> 剥离 IDE 代码    │      │
│  │  - feature('DAEMON') == false -> 剥离守护进程代码      │      │
│  │  - 未激活特性在构建时完全剥离，零运行时开销            │      │
│  │       │                                               │      │
│  │       ▼                                               │      │
│  │  单一打包产物                                          │      │
│  └──────────────────────────────────────────────────────┘      │
│                                                                 │
│  特性标志列表：                                                  │
│  | 标志            | 功能                    | 预估节省 |       │
│  | PROACTIVE       | 主动 Agent 模式          | ~150KB   |       │
│  | KAIROS          | Kairos 子系统            | ~200KB   |       │
│  | BRIDGE_MODE     | IDE bridge 集成          | ~300KB   |       │
│  | DAEMON          | 后台守护进程模式          | ~100KB   |       │
│  | VOICE_MODE      | 语音输入/输出            | ~200KB   |       │
│  | AGENT_TRIGGERS  | 触发式 Agent 动作         | ~50KB    |       │
│  | MONITOR_TOOL    | 监控工具                 | ~80KB    |       │
│  | COORDINATOR_MODE| 多 Agent 协调器           | ~120KB   |       │
│  | WORKFLOW_SCRIPTS| 工作流自动化脚本          | ~100KB   |       │
└─────────────────────────────────────────────────────────────────┘
```

### 特性标志与死代码消除

```typescript
import { feature } from 'bun:bundle'

// 编译时 DCE：bun:bundle 的 feature() 在构建时剥离
if (feature('VOICE_MODE')) {
  // 此代码在构建时被完全剥离（如果标志未激活）
  // 实现语音输入/输出功能
}
```

### 分发方式

| 方式 | 命令 | 说明 |
|------|------|------|
| **npm 分发** | `npm install -g @anthropic-ai/claude-code` | 主要分发渠道 |
| **单一二进制** | Bun 打包 | 自包含可执行文件 |

### 构建限制

- **无 Nix 支持**：没有可复现构建配置
- **无 Bazel/Cross**：不支持跨平台编译
- **依赖 Bun 运行时**：目标机器需要 Bun 环境（或使用 npm 安装的打包版本）
- **无 monorepo 工作区**：单一项目结构

---

## Codex CLI 实现

### 双构建系统策略

Codex CLI 采用 **Bazel + Cargo** 双构建系统，辅以 Nix 可复现构建：

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

### 构建工具矩阵

| 工具 | 用途 | 说明 |
|------|------|------|
| **Bazel 9** | 主力构建系统，CI/CD，跨平台编译 | 生产级构建，支持远程缓存和分布式构建 |
| **Cargo** | Rust 包管理和开发构建 | 日常开发首选，编译速度快 |
| **Nix** | 可复现构建（flake.nix） | 确保任何机器构建出完全相同的二进制 |
| **just** | 任务运行器 | 统一的开发任务入口 |
| **pnpm** | npm 生态包管理（monorepo） | 管理 JavaScript/TypeScript 相关依赖 |

### 项目文件结构

```
codex-rs/
├── BUILD.bazel            # Bazel 构建配置
├── MODULE.bazel           # Bazel 模块定义
├── flake.nix              # Nix 构建支持
├── justfile               # just 任务运行器配置
├── pnpm-workspace.yaml    # pnpm monorepo 工作区定义
├── Cargo.toml             # Cargo workspace 定义
└── codex-rs/              # 60+ Crate 源码
```

### Cargo Workspace 组织

```
┌──────────────────────────────────────────────────────────────┐
│                 Cargo Workspace 分层组织                      │
│                                                               │
│  层级          Crate 数量    说明                              │
│  ────          ──────────    ────                              │
│  入口层        3            cli, tui, exec                     │
│  核心层        5            core, tools, protocol, ...         │
│  沙箱层        6            sandboxing, linux-sandbox, ...     │
│  MCP 层        3            mcp-server, rmcp-client, ...       │
│  模型层        5            codex-client, chatgpt, ollama, ... │
│  状态层        3            rollout, codex-state, state-db     │
│  配置层        2            codex-config, codex-features       │
│  认证层        2            codex-login, codex-keyring-store   │
│  遥测层        2            codex-analytics, codex-otel        │
│  工具层        3            codex-tools, codex-apply-patch, .. │
│  网络层        2            codex-network-proxy, ...           │
│  其他          ~20          codex-utils-*, codex-git-utils, .. │
│                                                               │
│  总计: 60+ Crate                                             │
└──────────────────────────────────────────────────────────────┘
```

### 分发方式

| 方式 | 说明 |
|------|------|
| **零依赖原生可执行文件** | Rust 编译产物，无需运行时 |
| **npm 分发** | `npm install -g @openai/codex` |
| **cargo 分发** | `cargo install codex-cli` |
| **pnpm monorepo** | 工作区内包管理 |

### 跨平台编译

Bazel 支持以下平台的交叉编译：
- Linux x86_64
- macOS ARM64
- Windows x86_64

---

## 对比分析

### 构建复杂度对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **构建工具数量** | 1 个（Bun bundle） | 5 个（Bazel/Cargo/Nix/just/pnpm） |
| **构建配置文件** | 少量 | BUILD.bazel + MODULE.bazel + flake.nix + justfile + Cargo.toml |
| **学习曲线** | 低（Bun 简单易用） | 高（Bazel + Nix 学习成本大） |
| **日常开发** | `bun run` | `cargo build` |
| **CI/CD** | `bun bundle` | `bazel build` |
| **构建速度** | 快（Bun 原生优化） | 中（Cargo 增量编译快，Bazel 首次慢） |

### 跨平台支持对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **跨平台编译** | 不支持 | Bazel 跨平台编译 |
| **目标平台** | 依赖 Bun 运行时 | 零依赖原生二进制 |
| **平台覆盖** | macOS/Linux/Windows（需 Bun） | Linux x86_64, macOS ARM64, Windows x86_64 |
| **运行时依赖** | Bun 运行时 | 无（原生二进制） |
| **分发体积** | 中等（Bun 打包） | 小（Rust 静态链接） |

### 可复现性对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **可复现构建** | 无 | Nix（flake.nix） |
| **依赖锁定** | Bun lockfile | Cargo.lock + Nix flake lock |
| **环境一致性** | 依赖 Bun 版本 | Nix 确保完全一致 |
| **远程缓存** | 无 | Bazel 远程缓存 |
| **分布式构建** | 无 | Bazel 分布式构建 |

### 特性标志对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **特性标志系统** | GrowthBook + bun:bundle DCE | `feature.state` OTel 指标 |
| **死代码消除** | 构建时剥离（零运行时开销） | 无构建时 DCE |
| **远程控制** | GrowthBook 远程特性开关 | 无远程控制 |
| **A/B 测试** | 支持 | 不支持 |
| **标志数量** | 9+ 个特性标志 | 未公开 |

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| Bun bundle 构建简单，学习成本低 | 无 Nix/可复现构建支持 |
| bun:bundle DCE 构建时剥离未激活代码，零运行时开销 | 无跨平台编译能力 |
| GrowthBook 远程特性开关 + A/B 测试，产品迭代灵活 | 依赖 Bun 运行时，非原生二进制 |
| npm 分发便捷，安装简单 | 单一构建系统，CI/CD 灵活性不足 |
| 9+ 特性标志精细控制功能发布 | 无 Bazel 远程缓存和分布式构建 |
| TypeScript + Bun 开发效率高 | ~50 万行代码，构建产物体积较大 |
| 原生 JSX/TSX 支持无需转译 | 无 monorepo 工作区管理 |
| 构建速度快（Bun 原生优化） | 无法确保跨机器构建一致性 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| Bazel 9 支持跨平台编译和远程缓存 | 5 个构建工具，学习曲线陡峭 |
| Nix flake.nix 提供完全可复现构建 | 构建配置复杂（BUILD.bazel + MODULE.bazel + flake.nix + ...） |
| 零依赖原生可执行文件，分发简单 | Bazel 首次构建慢，配置复杂 |
| Cargo + pnpm monorepo 双生态支持 | Nix 学习成本高，社区相对小 |
| 60+ Crate 微服务化，编译隔离 | 无构建时死代码消除 |
| pnpm monorepo 工作区管理规范 | 无远程特性开关和 A/B 测试 |
| just 任务运行器统一开发入口 | Rust 编译时间较长（相比 Bun） |
| 双构建系统（Bazel + Cargo）灵活切换 | Bazel 和 Nix 的组合增加了 CI/CD 复杂度 |
| ~8 万行 Rust 实现 ~50 万行 TS 等效功能 | pnpm 仅管理 JS/TS 依赖，Rust 依赖由 Cargo 管理 |
| Bazel 分布式构建加速大型项目 | 构建系统维护成本高（需同时维护 Bazel 和 Cargo 配置） |
