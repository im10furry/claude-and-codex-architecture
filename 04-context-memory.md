# 上下文与记忆管理对比：Claude Code vs Codex CLI

## Claude Code 实现

### 多层记忆架构

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

### 四层压缩策略

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

### microCompact 实现细节

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

### autoCompact 摘要提示词

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

### reactiveCompact 截断策略

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

### tokenBudget 计算方式

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

### 持久记忆（memdir）

```
~/.claude/projects/<project-slug>/memory/
├── MEMORY.md           # 索引文件 (最多 200 行)
├── user_role.md        # 用户类型：角色、偏好
├── feedback_testing.md # 反馈类型：要重复/避免的行为
├── project_auth.md     # 项目类型：持续工作上下文
└── reference_docs.md   # 参考类型：外部系统指针
```

### 提示词缓存优化

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

### LRU 文件状态缓存

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

### Dream Task

独特的后台记忆整合机制，在用户空闲时自动回顾会话并更新记忆文件，实现跨会话学习持久化：

```
用户空闲（超过 5 分钟无交互）
  -> autoDream 服务启动（后台子 Agent）
  -> 回顾当前会话历史，提取关键信息
  -> 更新 ~/.claude/projects/<slug>/memory/ 下的记忆文件
  -> 下次会话启动时，Claude 已经"记住"了之前的学习
```

---

## Codex CLI 实现

### compact.rs 压缩算法

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

### 摘要提示词模板

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

### Token 感知截断

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

### 上下文窗口超时处理

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

## 对比分析

### 压缩策略对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **压缩层数** | 四层渐进式（微/自动/记忆/反应式） | 两层（自动压缩 + 截断） |
| **微压缩** | 有（清除旧工具结果，不调用 LLM） | 无 |
| **自动压缩** | 有（~167K tokens 触发，LLM 摘要） | 有（本地 + 远程两种模式） |
| **记忆压缩** | 有（提取到持久记忆文件） | 无 |
| **反应式压缩** | 有（API 错误触发） | 有（ContextWindowExceeded 触发） |
| **压缩触发阈值** | 80%/85%/90%/98% 四级 | 动态（上下文窗口溢出时） |
| **压缩粒度** | 工具结果级 + 消息级 | 消息级 |

### 压缩实现对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **微压缩实现** | microCompact() 函数，原地修改 messages | N/A |
| **微压缩策略** | 基于轮次年龄 + 内容大小阈值 | N/A |
| **大结果持久化** | >50K 字符的工具结果持久化到磁盘 | N/A |
| **自动压缩实现** | autoCompact() + COMPACT_SYSTEM_PROMPT | compact.rs 4 步流程 |
| **摘要提示词** | 内联 TypeScript 模板 | 外部 .md 模板文件 |
| **压缩边界标记** | `[COMPACT_SUMMARY]` + SUMMARY START/END | SUMMARY_PREFIX 常量 |
| **初始上下文注入** | 无 | BeforeLastUserMessage / DoNotInject |

### Token 计算对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **估算方法** | 字符数 / 3 | 字节数 / 4 |
| **适用文本** | 通用（英文为主） | UTF-8 文本 |
| **中文精度** | 低估（1 字符约 1-2 tokens，但按 1/3 计算） | 较好（UTF-8 中文 3 字节/token） |
| **代码精度** | 高估（代码 token 效率较高） | 较好（代码 ASCII 1 字节/token） |
| **误差范围** | -20% 到 +50% | 类似范围 |
| **精确计数** | 使用 API 返回的 usage 字段 | 使用 API 返回的 usage 字段 |
| **预算管理** | TokenBudget 类（阈值 80%/85%） | TruncationPolicy 枚举 |

### 持久记忆对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **持久记忆** | 有（memdir 目录结构） | 无 |
| **记忆文件** | MEMORY.md + 4 个分类文件 | N/A |
| **记忆类型** | 用户角色/反馈/项目上下文/参考文档 | N/A |
| **CLAUDE.md** | 6 级层级（全局/用户/项目/规则/本地） | N/A |
| **Dream Task** | 有（后台自动记忆整合） | N/A |
| **跨会话学习** | 支持（记忆文件持久化） | 不支持 |

### 缓存策略对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **提示词缓存** | cache_control: ephemeral（Anthropic API） | 前缀缓存（OpenAI API） |
| **静态/动态分离** | `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 标记 | 无显式分离 |
| **缓存节省** | ~85% 输入 token 成本 | 依赖 API 服务端前缀缓存 |
| **文件缓存** | LRU（100 文件 / 25MB） | 无显式文件缓存 |
| **Zstd 压缩** | 无 | 支持（超长对话历史） |

### 上下文窗口管理对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **默认窗口** | 200K tokens（扩展至 1M） | 取决于模型（128K-1M） |
| **输出预留** | ~20K tokens | 动态 |
| **溢出处理** | reactiveCompact 截断最旧消息 | 移除最旧历史项后重试 |
| **重试策略** | forceCompact + reactiveCompact | remove_first_item + continue |
| **最大重试** | 动态（直到可压缩） | 动态（直到只剩一条消息） |

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| 四层渐进式压缩策略精细，覆盖从轻量到紧急的全场景 | 四层策略实现复杂，各层触发条件需协调 |
| 微压缩不调用 LLM，零成本清除旧工具结果 | 微压缩仅清除工具结果，不处理文本消息膨胀 |
| 大工具结果（>50K 字符）持久化到磁盘，避免丢失 | 持久化路径管理增加复杂度 |
| autoCompact 摘要提示词详细，保留 6 类关键信息 | 摘要质量依赖模型能力，可能丢失细节 |
| CLAUDE.md 6 级层级提供灵活的项目记忆 | 6 级加载顺序复杂，冲突规则不明确 |
| Dream Task 后台记忆整合，实现跨会话学习 | Dream Task 消耗额外 API 调用，增加成本 |
| TokenBudget 类提供精确的阈值管理（80%/85%） | 字符数/3 的估算对中文严重低估 |
| cache_control: ephemeral 提示词缓存节省 ~85% 成本 | 缓存仅限 Anthropic API，多云切换时失效 |
| LRU 文件缓存（100 文件/25MB）减少冗余 I/O | 缓存一致性需手动维护（编辑时更新） |
| 静态/动态提示词分离最大化缓存命中率 | `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 标记不够优雅 |
| reactiveCompact 作为最后防线处理 API 错误 | 截断策略简单（保留尾部 80%），可能丢失关键上下文 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| compact.rs 442 行实现简洁，逻辑清晰 | 仅两层压缩（自动 + 截断），缺少微压缩层 |
| 两种压缩模式（Pre-turn/Mid-turn）适配不同场景 | 无微压缩，旧工具结果持续占用上下文 |
| 远程压缩判断（should_use_remote_compact_task）灵活 | 远程压缩仅支持 OpenAI，本地模型不可用 |
| 外部 .md 模板文件，摘要提示词易于修改和版本管理 | 摘要提示词内容不如 Claude Code 详细（仅 4 类信息） |
| truncate_with_byte_estimate 50/50 前后缀策略合理 | 50/50 分配对长消息可能截断关键中间内容 |
| UTF-8 安全切割，避免乱码 | 字节数/4 估算对纯英文高估（ASCII 1 字节/token） |
| build_compacted_history_with_limit 从最新消息向前填充 | 无 TokenBudget 类等价物，阈值管理不够精细 |
| 上下文窗口超时处理移除最旧项后重试，保持前缀缓存 | 重试无上限，极端情况可能移除所有历史 |
| Zstd 压缩支持减少超长对话传输数据量 | Zstd 压缩/解压增加 CPU 开销 |
| 前缀缓存利用现代推理引擎 KV 缓存 | 缓存效果完全依赖 API 服务端，不可控 |
| TruncationPolicy 枚举提供多种截断策略 | 无持久记忆系统，跨会话信息丢失 |
| COMPACT_USER_MESSAGE_MAX_TOKENS (20K) 限制单条消息 | 无 CLAUDE.md 等价物，项目上下文管理薄弱 |
| InitialContextInjection 枚举明确控制注入行为 | 无 Dream Task 等价物，无后台记忆整合 |
