# 错误处理与会话恢复对比

## Claude Code 实现

### withRetry 重试策略

Claude Code 的重试机制采用**指数退避 + 随机抖动**策略，并尊重服务端的 `retry-after` 头：

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

### 错误分类（classifyError）

Claude Code 将错误分为 **6 大类**，每类有不同的处理策略：

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

### 速率限制处理

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

### 工具执行失败处理

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

### 会话持久化到磁盘

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

### 持久化时机

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

### 恢复会话时的状态重建

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

**已知脆弱性：**
- **attachment 不持久化**：用户上传的文件附件（如图片）不会被持久化到磁盘，恢复会话时这些内容会丢失
- **fork-session 缓存前缀问题**：从 fork 的子 Agent 恢复会话时，API 提示词缓存前缀可能与原始会话不同，导致缓存未命中

---

## Codex CLI 实现

### 错误类型体系

核心实现在 `codex-rs/core/src/error.rs`（659 行），定义了完整的错误类型枚举，包含 **15+ 变体**：

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

### 可重试错误判断

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

### 重试策略

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

### 沙箱执行失败处理

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

### 会话恢复流程

Codex CLI 的会话恢复基于 JSONL Rollout 文件和 SQLite 双层存储：

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
│     ├── SessionMeta -> 恢复会话元数据                          │
│     ├── ResponseItem -> 重建消息历史                           │
│     ├── TurnContext -> 恢复轮次上下文                           │
│     ├── Compacted -> 恢复压缩历史                              │
│     └── EventMsg -> 重建事件状态                               │
│                                                               │
│  3. 创建新的 Codex 实例                                       │
│     ├── 使用恢复的历史初始化 Session                          │
│     ├── RolloutRecorder 以 Resume 模式打开（追加写入）        │
│     └── 继续正常的 SQ/EQ 事件循环                             │
└──────────────────────────────────────────────────────────────┘
```

**RolloutItem 类型覆盖：**

| 类型 | 恢复时处理 |
|------|-----------|
| `SessionMeta` | 恢复会话元数据（thread_id、cwd、Git 信息等） |
| `ResponseItem` | 重建消息历史（文本消息、工具调用、工具结果） |
| `TurnContext` | 恢复轮次上下文（模型信息、沙箱策略、审批策略） |
| `Compacted` | 恢复压缩历史（压缩摘要、替换历史） |
| `EventMsg` | 重建事件状态（执行命令、审批请求等） |

### 会话分叉

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
│  [上下键] 选择  [Enter] 恢复  [/] 搜索  [q] 退出              │
└──────────────────────────────────────────────────────────────┘
```

### 记忆系统

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

## 对比分析

### 错误分类粒度对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **错误分类数** | 6 大类（ErrorCategory 枚举） | 15+ 变体（CodexErr 枚举） |
| **分类方式** | 基于 HTTP 状态码 + 错误消息模式匹配 | Rust 枚举 + thiserror 派生 |
| **类型安全** | 运行时分类（TypeScript string union） | 编译时类型检查（Rust enum） |
| **沙箱错误** | 无独立分类 | `SandboxErr` 独立枚举（5 变体） |
| **上下文溢出** | `context_overflow` 类别 | `ContextWindowExceeded` 变体 |
| **使用限制** | 包含在 `rate_limit` 中 | 独立 `UsageLimitReached` + `QuotaExceeded` |
| **流式错误** | 包含在 `network_error` 中 | 独立 `Stream(String)` 变体 |

### 错误分类详细映射

| 错误场景 | Claude Code 分类 | Codex CLI 分类 |
|----------|-----------------|----------------|
| HTTP 429 | `rate_limit` | `UnexpectedStatus(429)` |
| HTTP 529 | `overloaded` | `ServerOverloaded` |
| HTTP 5xx | `server_error` | `InternalServerError` |
| HTTP 401 | `auth_error` | `UnexpectedStatus(401)` |
| HTTP 403 | `forbidden` | `UnexpectedStatus(403)` |
| 上下文过长 | `context_overflow` | `ContextWindowExceeded` |
| 网络断开 | `network_error` | `Stream("connection reset")` |
| 请求超时 | `timeout` | `Timeout` |
| 用户中断 | 无独立分类 | `TurnAborted` / `Interrupted` |
| 沙箱拒绝 | 无独立分类 | `Sandbox(Denied)` |
| 沙箱超时 | 无独立分类 | `Sandbox(Timeout)` |
| 使用限制 | `rate_limit`（组织级） | `UsageLimitReached` / `QuotaExceeded` |
| 重试超限 | 无独立分类 | `RetryLimit` |
| 无效请求 | 无独立分类 | `InvalidRequest` |

### 重试策略对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **退避算法** | 指数退避 + 随机抖动 | 指数退避（无抖动） |
| **基础延迟** | 1000ms | 1000ms |
| **最大延迟** | 30000ms | 30000ms |
| **抖动** | 有（0-1000ms 随机） | 无 |
| **retry-after 支持** | 支持（秒数 + HTTP 日期） | 不明确 |
| **可重试状态码** | [429, 529, 500, 502, 503] | 429, 502, 503 + Stream/Timeout/ServerOverloaded |
| **组织级限制** | 检测并跳过重试，通知升级 | `UsageLimitReached` 发送升级建议事件 |
| **上下文溢出** | 无特殊处理 | 移除最旧历史项后重试 |
| **Stream 错误** | 统一为 network_error | 按错误内容差异化处理（connection reset vs timeout） |

### 会话持久化格式对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **持久化格式** | JSONL | JSONL |
| **存储位置** | `~/.claude/projects/<slug>/sessions/` | `~/.codex/sessions/YYYY/MM/DD/` |
| **索引机制** | sessions.json（手动维护） | SQLite state.db（自动索引） |
| **事件类型** | message / tool_call / checkpoint / compact / permission_decision | ResponseItem / EventMsg / TurnContext / Compacted / SessionMeta |
| **元数据** | SessionMetadata 接口 | SessionMeta RolloutItem |
| **写入方式** | 同步追加 | 异步写入（有界队列，容量 256） |
| **debounce** | 5s（sessions.json 更新） | 无（异步通道缓冲） |
| **Checkpoint** | 支持（完整消息快照） | 支持（Compacted 记录） |

### 恢复可靠性对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **恢复步骤** | 4 步（元数据 -> JSONL -> 消息重建 -> 工具/权限恢复） | 3 步（加载 Rollout -> 重放事件 -> 创建实例） |
| **Checkpoint 优化** | 从最新 checkpoint 恢复，跳过中间事件 | 从 Compacted 记录恢复压缩历史 |
| **权限状态恢复** | 恢复权限模式，重置计数器 | 通过 TurnContext 恢复审批策略 |
| **工具注册表** | 重建工具注册表 | 通过 TurnContext 恢复 |
| **分叉支持** | 有（但存在缓存前缀问题） | 有（独立 Rollout 文件，原始会话不受影响） |
| **TUI 选择器** | 无明确描述 | 有（分页、搜索、排序） |
| **已知脆弱性** | attachment 不持久化、fork 缓存前缀问题 | 无明确记录 |

### 会话恢复完整性对比

```
Claude Code 恢复内容:
  [x] 消息历史（从 checkpoint 或逐条重建）
  [x] 工具注册表
  [x] 权限模式
  [x] 成本统计
  [x] 消息数量
  [ ] 文件附件（已知丢失）
  [ ] 权限拒绝计数器（重置为 0）
  [ ] API 提示缓存（fork 时可能失效）

Codex CLI 恢复内容:
  [x] 消息历史（ResponseItem 重放）
  [x] 模型信息（TurnContext）
  [x] 沙箱策略（TurnContext）
  [x] 审批策略（TurnContext）
  [x] 压缩历史（Compacted）
  [x] 事件状态（EventMsg）
  [x] Git 信息（SessionMeta）
  [ ] 文件附件（未明确）
```

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| **随机抖动**：指数退避 + 随机抖动，有效避免重试风暴（thundering herd） | **错误分类粒度粗**：仅 6 大类，缺少沙箱错误、流式错误等细粒度分类 |
| **retry-after 双格式**：支持秒数和 HTTP 日期两种 retry-after 格式，兼容性好 | **无沙箱错误处理**：没有独立的沙箱错误分类，无法区分沙箱拒绝和普通权限拒绝 |
| **组织级限制检测**：`isOrgRateLimit` 专门检测组织级限制，避免无意义重试 | **无上下文溢出自动恢复**：context_overflow 错误没有自动移除历史项重试的机制 |
| **工具失败差异化**：AbortError/PermissionDeniedError/ValidationError 分别处理，策略清晰 | **attachment 不持久化**：恢复会话时文件附件丢失，影响用户体验 |
| **Checkpoint 优化**：从最新 checkpoint 恢复，跳过中间事件，恢复速度快 | **fork 缓存前缀问题**：从 fork 的子 Agent 恢复时 API 缓存可能失效 |
| **debounce 写入**：sessions.json 更新使用 5s debounce，避免频繁 IO | **同步写入**：JSONL 追加是同步操作，可能阻塞主线程 |
| **权限决策审计**：`permission_decision` 事件记录每次权限决策，审计追踪完整 | **无 TUI 恢复选择器**：缺少可视化的会话恢复选择界面 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| **错误分类精细**：15+ 枚举变体覆盖所有错误场景，类型安全（Rust 编译时检查） | **无随机抖动**：纯指数退避缺少抖动，多实例并发重试时可能产生重试风暴 |
| **沙箱错误独立**：`SandboxErr` 独立枚举（5 变体），能精确定位沙箱层面的问题 | **无 retry-after 支持**：不明确支持 HTTP retry-after 头，可能过早重试 |
| **上下文溢出自动恢复**：`ContextWindowExceeded` 时自动移除最旧历史项并重试 | **使用限制处理简单**：`UsageLimitReached` 仅发送升级建议，无组织级限制区分 |
| **Stream 错误差异化**：按错误内容（connection reset vs timeout）差异化处理，策略精准 | **无权限决策审计**：缺少类似 Claude Code 的 `permission_decision` 事件记录 |
| **异步写入架构**：RolloutRecorder 使用有界队列（容量 256）异步写入，不阻塞主线程 | **恢复步骤较少**：3 步恢复相比 Claude Code 的 4 步，可能遗漏某些状态 |
| **SQLite 索引**：state.db 提供高效的会话搜索、过滤和分页，恢复选择体验好 | **无 debounce**：异步通道缓冲但无 debounce，高频事件可能导致频繁写入 |
| **TUI 恢复选择器**：内置分页、搜索、排序的会话恢复选择界面，用户体验优秀 | **无明确脆弱性记录**：缺少已知脆弱性的文档记录，维护者可能忽视潜在问题 |
| **会话分叉可靠**：独立 Rollout 文件，原始会话不受影响，分叉操作安全 | **无 attachment 持久化**：同样未明确记录文件附件的持久化策略 |
| **thiserror 派生**：错误类型自动生成 Display 实现，错误信息格式统一 | **错误消息粒度不一**：部分变体携带详细消息（String），部分仅有固定描述 |
