# 多 Agent 系统对比

## Claude Code 实现

Claude Code 支持**三个级别的多 Agent 执行**，从简单的子 Agent 到复杂的持久化团队，形成完整的层级体系。

### 三级架构

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

### 级别 1：子 Agent（AgentTool）

子 Agent 是最基础的多 Agent 单元，由主 Agent 通过 `AgentTool` 动态生成：

- **隔离机制**：每个子 Agent 拥有从父 Agent 克隆的独立文件缓存，确保子 Agent 的文件操作不影响父 Agent 的视图
- **生命周期管理**：独立的 `AbortController` 允许父 Agent 单独中止某个子 Agent
- **审计追踪**：每个子 Agent 维护独立的 JSONL 转录记录（侧链），便于事后审计
- **工具池过滤**：子 Agent 的可用工具可以按定义进行过滤，限制其能力范围
- **结果传递**：子 Agent 的执行结果以文本形式返回给父 Agent，由父 Agent 决定后续操作

### 级别 2：协调器模式

协调器模式通过环境变量 `CLAUDE_CODE_COORDINATOR_MODE=1` 激活：

- **系统提示重写**：激活后，系统提示词被重写为编排模式，主 Agent 变为协调器角色
- **Worker 生成**：协调器通过 AgentTool 生成多个 Worker，每个 Worker 拥有受限的工具集
- **XML 通信协议**：Worker 之间通过 `task-notification` XML 协议传递执行结果
- **结果聚合**：协调器负责收集所有 Worker 的结果，进行聚合后统一响应用户

### 级别 3：团队模式

团队模式是最复杂的多 Agent 形态，支持持久化的团队协作：

- **持久化**：团队配置持久化到 `~/.claude/teams/{name}.json`，跨会话存活
- **进程内运行**：`InProcessTeammates` 在同一进程中运行，共享内存空间
- **消息路由**：`SendMessageTool` 在队友之间路由消息，支持点对点通信
- **知识交换**：共享 scratchpad 文件系统，团队成员可以通过文件交换知识
- **结构化关闭**：采用 `request -> approve` 的结构化关闭协议，确保优雅退出

### Fork 缓存优化

当从同一上下文生成多个 Agent 时，Claude Code 使用 **Fork 机制**最大化 API 提示缓存命中：

- 所有子 Agent 共享相同的前缀（父对话历史）
- 仅最后一条指令不同
- Fork 操作在 API 层面复用已缓存的 prompt 前缀，大幅降低 token 成本和延迟

---

## Codex CLI 实现

Codex CLI 支持完整的多 Agent 编排系统，每个 Agent 运行在独立的沙箱容器中。

### 核心原语

| 原语 | 说明 |
|------|------|
| `spawn` | 创建新的 Agent 实例 |
| `resume` | 恢复已暂停的 Agent |
| `wait` | 等待 Agent 完成 |
| `close` | 终止 Agent |
| `send-message` | 向 Agent 发送消息 |
| `list` | 列出所有 Agent |
| `assign-task` | 向 Agent 分配任务 |

### 批量 Agent 任务

Codex CLI 支持从 CSV 文件批量创建 Agent 任务，每个 CSV 行定义一个独立的 Agent 任务，所有 Agent 并行运行：

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

**关键设计要点：**
- **独立沙箱**：每个 Agent 拥有独立的沙箱环境（`sandbox_policy.clone()`），实现进程级隔离
- **并行执行**：所有 Agent 并行运行，充分利用系统资源
- **CSV 驱动**：通过 CSV 文件定义任务，便于批量处理和自动化

### Guardian 守护者模式

Guardian 作为安全中间层，在多 Agent 场景中决定哪些操作可以自动执行：

```
┌──────────────────────────────────────────────────────────────┐
│                 Guardian 守护者模式                            │
│                                                               │
│  Agent 请求                                                   │
│      │                                                        │
│      v                                                        │
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
│    v         v                                                │
│  自动执行   请求用户审批                                       │
│    │         │                                                │
│    v         v                                                │
│  沙箱执行   ApprovalOverlay                                   │
│              │                                                │
│              v                                                │
│         用户确认/拒绝                                          │
└──────────────────────────────────────────────────────────────┘
```

**Guardian 在多 Agent 中的角色：**
- 每个 Agent 的操作请求都经过 Guardian 评估
- Guardian 根据审批策略（`AskForApproval`）决定是否需要用户介入
- 白名单内的安全命令自动放行，减少用户审批疲劳
- 非白名单命令根据策略决定是否请求用户审批

---

## 对比分析

### 架构范式对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **架构模型** | 三级递进架构（子 Agent -> 协调器 -> 团队） | 扁平原语 + Guardian 守护者 |
| **隔离机制** | 应用层隔离（独立缓存、AbortController） | OS 内核级隔离（独立沙箱容器） |
| **通信方式** | 文本返回 / XML 协议 / 消息路由 | 原语调用（spawn/send-message/wait） |
| **持久化** | 团队文件持久化（JSON） | 会话 Rollout 持久化（JSONL + SQLite） |
| **批量执行** | Fork 机制（共享前缀缓存） | CSV 批量生成（独立沙箱） |
| **安全层** | 工具池过滤 + 权限管道 | Guardian 白名单 + OS 沙箱 |

### Agent 生命周期管理对比

| 操作 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **创建** | `AgentTool` 调用 | `spawn` 原语 |
| **暂停/恢复** | `AbortController` 中止 | `resume` 原语 |
| **终止** | AbortController 中止 | `close` 原语 |
| **等待完成** | Promise 链 | `wait` 原语 |
| **消息传递** | `SendMessageTool` / XML 协议 | `send-message` 原语 |
| **任务分配** | 协调器模式 / 团队模式 | `assign-task` 原语 |
| **列表查看** | 团队文件读取 | `list` 原语 |

### 隔离机制深度对比

```
Claude Code 隔离层级:
  子 Agent
  ├── 独立文件缓存（应用层）
  ├── 独立 AbortController（应用层）
  ├── 独立 JSONL 转录（应用层）
  └── 过滤的工具池（应用层）
  [全部在同一个 Node.js 进程中运行]

Codex CLI 隔离层级:
  Agent
  ├── 独立沙箱容器（OS 内核层）
  │   ├── Landlock 文件系统隔离
  │   ├── seccomp 系统调用过滤
  │   └── Bubblewrap 命名空间隔离
  ├── Guardian 审批（应用层）
  └── 独立 Rollout 记录（应用层）
  [运行在独立进程/命名空间中]
```

### 通信模型对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **父子通信** | 文本返回（子 -> 父） | 原语调用（双向） |
| **兄弟通信** | XML task-notification 协议 | send-message 原语 |
| **团队通信** | SendMessageTool 路由 | send-message 原语 |
| **知识共享** | 共享 scratchpad 文件系统 | 独立沙箱（无共享） |
| **结果聚合** | 协调器聚合 | 父 Agent 等待收集 |

### 规模化能力对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **批量创建** | Fork 机制（手动） | `spawn_agents_on_csv`（自动化） |
| **并行度** | 受限于单进程内存 | 受限于系统资源（独立进程） |
| **资源隔离** | 共享进程资源 | 独立进程资源 |
| **故障隔离** | 子 Agent 崩溃可能影响父进程 | Agent 崩溃不影响其他 Agent |
| **API 缓存优化** | Fork 共享前缀缓存 | 无 API 缓存优化 |

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| **三级架构灵活**：从简单的子 Agent 到复杂的持久化团队，按需选择合适的复杂度级别 | **隔离深度不足**：所有 Agent 运行在同一 Node.js 进程中，子 Agent 崩溃可能影响父进程 |
| **Fork 缓存优化**：多个子 Agent 共享 API 提示前缀缓存，大幅降低 token 成本和延迟 | **无 OS 级隔离**：Agent 之间仅靠应用层隔离，无法防止恶意 Agent 影响系统 |
| **团队持久化**：团队配置持久化到磁盘，跨会话存活，支持长期协作场景 | **进程内运行限制**：InProcessTeammates 共享进程资源，大规模并行时可能成为瓶颈 |
| **丰富的通信方式**：支持文本返回、XML 协议、消息路由等多种通信模式 | **知识共享风险**：共享 scratchpad 文件系统可能导致 Agent 间意外干扰 |
| **协调器模式**：专门的编排模式，支持 Worker 结果聚合和统一响应 | **批量创建不够自动化**：相比 CSV 驱动的批量创建，Fork 机制更依赖手动编排 |
| **结构化关闭协议**：团队模式采用 request -> approve 协议，确保优雅退出 | **XML 协议复杂度**：task-notification XML 协议增加了通信复杂度 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| **OS 级隔离**：每个 Agent 运行在独立沙箱容器中，进程级隔离保证故障不扩散 | **通信原语较基础**：spawn/send-message/wait 等原语功能单一，复杂编排需要额外逻辑 |
| **批量自动化**：`spawn_agents_on_csv` 从 CSV 文件批量创建 Agent，适合大规模并行任务 | **无 API 缓存优化**：每个 Agent 独立发起 API 请求，无法共享提示缓存 |
| **Guardian 统一安全**：所有 Agent 的操作都经过 Guardian 评估，安全策略一致 | **无持久化团队**：Agent 生命周期与会话绑定，没有跨会话的团队持久化机制 |
| **故障完全隔离**：Agent 崩溃在沙箱内，不影响其他 Agent 或主进程 | **无知识共享机制**：独立沙箱意味着 Agent 之间无法直接共享文件或状态 |
| **原语简洁**：7 个核心原语覆盖完整的 Agent 生命周期，API 设计清晰 | **无协调器角色**：缺少专门的协调器模式，复杂编排需要用户自行实现 |
| **独立 Rollout 记录**：每个 Agent 有独立的会话记录，便于独立审计和调试 | **Guardian 白名单局限**：基于确定性匹配的白名单可能过于严格，限制 Agent 自主性 |
