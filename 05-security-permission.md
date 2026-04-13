# 安全与权限系统对比

## Claude Code 实现

### 权限管道

Claude Code 采用**软件层权限治理**方案，通过多层权限管道对工具调用进行逐级过滤。整个管道从工具调用到达开始，经过模式检查、规则匹配、LLM 分类器，最终到用户确认。

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

### 6 种权限模式

| 模式 | 符号 | 行为 |
|------|------|------|
| `default` | `>` | 所有非只读工具都需要询问 |
| `acceptEdits` | `>>` | 自动允许 cwd 内文件编辑 |
| `plan` | `?` | 在工具调用之间暂停以供审查 |
| `bypassPermissions` | `!` | 跳过所有检查（危险） |
| `auto` | `A` | LLM 分类器决定（特性门控） |

### 权限管道完整代码流程

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

### 7 层安全机制详细说明

| 层级 | 机制 | 实现位置 | 说明 |
|------|------|----------|------|
| **1** | 危险文件保护 | `utils/permissions/dangerousFiles.ts` | `.gitconfig`、`.bashrc`、`.zshrc`、`.mcp.json`、`/etc/` 下的系统文件被阻止修改 |
| **2** | 危险命令检测 | `BashTool.checkPermissions()` | `rm -rf /`、`git push --force`、`DROP TABLE`、`chmod 777`、`> /dev/sda` 等模式匹配 |
| **3** | Bypass 权限终止开关 | `growthbook.ts` + `isBypassDisabledByRemote()` | GrowthBook 特性门可以远程禁用 bypass 模式 |
| **4** | Auto 模式断路器 | Statsig 实时门 | Statsig 门可以远程禁用 auto 模式，连续拒绝 >3 或总计 >20 时自动回退到 ASK |
| **5** | 拒绝追踪 | `PermissionStats` | 跟踪 `deniedCount` 和 `consecutiveDenials`，超过阈值时切换到更保守的模式 |
| **6** | 技能范围收窄 | `SkillTool` | 编辑 `.claude/skills/X/` 时提供窄范围权限 |
| **7** | MCP Shell 阻止 | `MCPTool` | MCP 来源的技能永不执行 shell 命令，防止远程代码注入 |

### LLM 自动分类器（auto 模式）

Claude Code 的 auto 模式使用一个轻量级 LLM 分类器（Claude 3 Haiku）来判断工具调用是否安全：

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

**关键设计要点：**
- 使用 `claude-3-haiku-20240307`（最轻量模型）降低延迟和成本
- `max_tokens: 1` 限制输出为单个字符，确保响应速度
- 断路器机制：连续拒绝 >=3 或总计 >=20 时自动回退到 ASK 模式
- 远程终止开关：GrowthBook/Statsig 可实时禁用 auto 模式

---

## Codex CLI 实现

### 沙箱系统

Codex CLI 采用**平台原生 OS 内核级沙箱**方案，实现真正的进程级隔离。这是其最重要的安全特性。

#### macOS Seatbelt 沙箱

macOS 使用 Apple 原生的 **Seatbelt** (`sandbox-exec`) 沙箱系统。核心实现在 `core/src/seatbelt.rs`（623 行）。

**SBPL 策略结构：**

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

**特殊保护目录：**

| 目录 | 保护原因 |
|------|----------|
| `.git/` | Git 版本控制数据，防止破坏版本历史 |
| `.codex/` | Codex 自身的配置和状态数据 |
| `.hg/` | Mercurial 版本控制数据 |
| `.svn/` | Subversion 版本控制数据 |

#### Linux Landlock + Bubblewrap

Linux 沙箱采用**三层架构**，核心实现在 `codex-rs/linux-sandbox/src/landlock.rs`（约 300 行）。

```
┌─────────────────────────────────────────────────────────────┐
│                 Linux 三层沙箱架构                            │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 第一层: Landlock (内核 5.13+)                         │   │
│  │  - 文件系统访问控制（只读/读写目录）                    │   │
│  │  - 在当前线程上应用，子进程继承                         │   │
│  │  - 适用于简单策略（read-only, workspace-write）        │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 第二层: seccomp (系统调用过滤)                         │   │
│  │  - 限制可用的系统调用                                  │   │
│  │  - 阻止危险系统调用（如 ptrace、mount）                │   │
│  │  - 与 Landlock 配合使用                               │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 第三层: Bubblewrap (用户命名空间隔离)                   │   │
│  │  - 完整的 Linux 命名空间隔离                           │   │
│  │  - 独立的 mount/IPC/network/PID 命名空间               │   │
│  │  - 适用于复杂策略（需要细粒度控制时自动路由）            │   │
│  │  - codex-linux-sandbox 独立进程                        │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  策略路由:                                                   │
│  简单策略 -> Landlock 直接处理                                │
│  复杂策略 -> 自动路由到 Bubblewrap                            │
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

#### Windows Restricted Token

Windows 使用**限制令牌（Restricted Token）**模式实现沙箱：

```
┌─────────────────────────────────────────────────────────────┐
│                 Windows 沙箱策略                              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Restricted Token 模式                                 │   │
│  │  - 创建限制令牌，移除特权 SID                          │   │
│  │  - 限制文件系统访问权限                                │   │
│  │  - 适用于简单策略                                      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Elevated Runner 模式                                   │   │
│  │  - 提升权限的后端进程                                  │   │
│  │  - 支持更细粒度的文件系统控制                          │   │
│  │  - 通过 IPC 与主进程通信                               │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

#### 沙箱策略注入系统提示词

沙箱策略不仅通过操作系统内核执行，还会注入到系统提示词中，让模型在尝试操作前就知道其约束条件：

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

#### 网络访问控制

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
│  │  - 应用层网络代理                                      │   │
│  │  - 审计所有网络请求                                    │   │
│  │  - 支持域名白名单/黑名单                               │   │
│  │  - 记录网络访问日志                                    │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  默认行为:                                                   │
│  read-only 模式 -> 网络完全禁止                               │
│  workspace-write 模式 -> 网络完全禁止                         │
│  danger-full-access 模式 -> 网络完全允许                       │
└─────────────────────────────────────────────────────────────┘
```

### Guardian 安全守护系统

#### 审批策略

Guardian 系统通过 `AskForApproval` 枚举定义四级审批策略：

```rust
#[derive(Debug, Clone, Deserialize, Serialize, PartialEq, JsonSchema)]
pub enum AskForApproval {
    /// 不信任模式 -- 所有命令都需要用户审批
    Untrusted,

    /// 失败时审批 -- 仅当命令失败时需要审批
    OnFailure,

    /// 按需审批 -- 模型认为需要时请求审批
    OnRequest,

    /// 从不审批 -- 所有命令自动执行（危险）
    Never,
}
```

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| `untrusted` | 所有命令都需要用户确认 | 安全敏感环境 |
| `on-failure` | 命令失败时才需要确认 | 开发环境 |
| `on-request` | 模型自行判断是否需要确认 | 日常使用 |
| `never` | 所有命令自动执行 | CI/CD、容器环境 |

#### canAutoApprove 判断逻辑

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

    // 处理管道命令 -- 取第一个命令
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

#### 用户确认 UI

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
│  │ │  命令需要审批                                     │  │  │
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

## 对比分析

### 安全架构范式对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **安全模型** | 软件层权限治理（应用层） | OS 内核级沙箱（系统层） |
| **隔离粒度** | 工具调用级别 | 进程/系统调用级别 |
| **执行前检查** | 7 层权限管道 | SBPL/seccomp/Landlock 规则 |
| **执行时保护** | 无（依赖执行前检查） | OS 内核强制执行 |
| **权限决策者** | LLM 分类器 + 人类用户 | OS 内核 + 白名单 + 人类用户 |
| **远程控制** | GrowthBook/Statsig 特性门 | 无（本地策略决定） |
| **跨平台** | 统一 TypeScript 实现 | 平台特定实现（Seatbelt/Landlock/Restricted Token） |

### 权限模式对比

| Claude Code 模式 | Codex CLI 等价 | 差异说明 |
|------------------|----------------|----------|
| `default` (`>`) | `untrusted` | 两者都要求所有非只读操作需用户确认 |
| `acceptEdits` (`>>`) | `on-request` | Claude Code 仅限 cwd 内编辑；Codex CLI 由模型自行判断 |
| `plan` (`?`) | 无直接等价 | Claude Code 独有的暂停审查模式 |
| `auto` (`A`) | `on-failure` | Claude Code 用 LLM 分类器；Codex CLI 仅在失败时审批 |
| `bypassPermissions` (`!`) | `never` | 两者都是危险的全自动模式 |

### 攻击面分析

| 攻击面 | Claude Code | Codex CLI |
|--------|-------------|-----------|
| **恶意工具调用绕过** | 可能（软件层检查可被绕过） | 极低（OS 内核强制执行） |
| **Shell 注入** | 依赖危险命令模式匹配 | seccomp 系统调用过滤 + 命名空间隔离 |
| **文件系统越权** | 依赖路径检查和规则匹配 | Landlock/Seatbelt 内核级文件系统隔离 |
| **网络越权** | 无独立网络控制 | 多层网络控制（Seatbelt/seccomp/NetworkProxy） |
| **LLM 分类器被欺骗** | 可能（prompt injection） | 不适用（不依赖 LLM 做安全决策） |
| **远程策略下发** | GrowthBook CDN 轮询 | 无远程策略通道 |

### 安全保证层级

```
Claude Code 安全保证链:
  应用层规则 -> LLM 分类器 -> 人类确认 -> 执行
  [软件层]    [AI 层]       [人类层]    [无保护]

Codex CLI 安全保证链:
  Guardian 白名单 -> 人类确认 -> OS 沙箱 -> 执行
  [应用层]         [人类层]    [内核层]    [内核保护]
```

Claude Code 的安全保证在执行阶段终止，即一旦通过权限检查，命令在 OS 层面没有任何额外限制。而 Codex CLI 即使通过了所有应用层检查，OS 沙箱仍然在执行阶段提供内核级的强制隔离。

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| **灵活性极高**：6 种权限模式覆盖从严格到宽松的各种场景，用户可按需切换 | **安全性依赖应用层**：所有安全检查在软件层完成，一旦被绕过则无内核级保护 |
| **LLM 智能分类**：auto 模式使用 Claude 3 Haiku 进行语义级安全判断，能理解上下文意图 | **LLM 分类器可被欺骗**：prompt injection 可能导致分类器误判，存在安全风险 |
| **远程紧急响应**：通过 GrowthBook/Statsig 可实时禁用危险模式（如 bypass），快速响应安全事件 | **无 OS 级隔离**：命令执行后不受沙箱保护，恶意命令可能影响整个系统 |
| **丰富的规则系统**：支持 Deny/Allow/Ask 三种规则动作，可按工具名、路径模式等灵活配置 | **网络控制薄弱**：没有独立的网络访问控制层，无法阻止工具执行时的网络请求 |
| **人类始终在环**：default 模式下所有非只读操作都需要人类确认，安全意识强 | **bypass 模式风险高**：`!` 模式跳过所有检查，一旦启用则完全失去保护 |
| **断路器机制**：连续拒绝计数器自动回退到更保守模式，防止 auto 模式失控 | **危险命令检测有限**：基于模式匹配，可能遗漏变种命令 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| **OS 内核级安全保证**：Seatbelt/Landlock/seccomp 在内核层面强制执行，即使应用层被绕过也无法越权 | **功能受限**：沙箱严格限制文件系统和网络访问，某些合法操作可能被阻止 |
| **跨平台沙箱**：macOS (Seatbelt)、Linux (Landlock+Bubblewrap)、Windows (Restricted Token) 全平台覆盖 | **Linux 内核版本要求**：Landlock 需要内核 5.13+，旧系统无法使用最优沙箱 |
| **多层网络控制**：Seatbelt 网络规则 + seccomp 系统调用过滤 + NetworkProxy 应用层代理，三重保障 | **审批策略粒度较粗**：4 级审批策略相比 Claude Code 的 6 种模式灵活性不足 |
| **白名单机制简单可靠**：`is_known_safe_command` 基于确定性匹配，不存在 LLM 被欺骗的风险 | **无 LLM 智能判断**：不能像 Claude Code 的 auto 模式那样理解上下文语义 |
| **提示词注入沙箱策略**：将权限约束注入系统提示词，减少模型尝试被阻止操作的无用 API 调用 | **无远程紧急响应**：没有类似 GrowthBook 的远程策略下发能力，无法实时调整安全策略 |
| **进程级隔离**：Bubblewrap 提供完整的命名空间隔离（mount/IPC/network/PID），安全性极高 | **白名单维护成本**：安全命令白名单需要手动维护，新增安全命令需要更新代码 |
| **特殊目录保护**：`.git/`、`.codex/` 等目录在沙箱层面被保护，防止版本控制数据被破坏 | **danger-full-access 模式风险**：该模式允许网络访问和完整文件系统权限，安全保证大幅降低 |
