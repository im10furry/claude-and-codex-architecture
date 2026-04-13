# Agent 循环对比：Claude Code vs Codex CLI

## Claude Code 实现

### query() async generator 核心结构

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

### queryModelWithStreaming() 实现

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

### StreamingToolExecutor 并行执行机制

`StreamingToolExecutor`（~530行）是 Claude Code 中最精巧的组件之一。它实现了一个**事件驱动的状态机**，能够在模型仍在生成响应时就开始执行已完成的工具调用，显著减少端到端延迟。

#### 设计原理

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
class StreamingToolExecutor {
  onEvent(event: StreamEvent): void {
    switch (event.type) {
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

      case 'content_block_delta':
        if (event.delta.type === 'input_json_delta') {
          const block = this.pendingBlocks.get(event.id);
          if (block) {
            block.inputJson += event.delta.partial_json;
          }
        }
        break;

      case 'content_block_stop':
        const block = this.pendingBlocks.get(event.id);
        if (block) {
          this.pendingBlocks.delete(event.id);
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
          this.scheduleExecution(block.id, block.name, input);
        }
        break;
    }
  }
}
```

#### 并发安全的调度执行

```typescript
class StreamingToolExecutor {
  private async scheduleExecution(
    id: string, name: string, input: unknown,
  ): Promise<void> {
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

### max-output-tokens 恢复机制

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

### StreamEvent 类型定义

```typescript
type StopReason = 'end_turn' | 'max_tokens' | 'stop_sequence' | 'tool_use';

type StreamEvent =
  | MessageStartEvent | TextStartEvent | TextDeltaEvent
  | ThinkingStartEvent | ThinkingDeltaEvent
  | ToolUseStartEvent | ToolInputDeltaEvent
  | MessageDeltaEvent | MessageStopEvent
  | ToolExecutionStartEvent | ToolExecutionCompleteEvent
  | ToolExecutionErrorEvent | MicroCompactEvent | AutoCompactEvent
  | AbortedEvent | MaxTokensRetryEvent | MaxTokensExceededEvent
  | ApiErrorEvent | StreamErrorEvent;
```

### QueryEngine 外层循环

QueryEngine 是 `query()` 的外层包装器，提供会话级的管理能力：

```typescript
export class QueryEngine {
  private messages: Message[] = [];
  private fileCache: LRUCache<string, FileContent>;
  private costTracker: CostTracker;
  private abortController: AbortController = new AbortController();
  private permissionState: PermissionState;
  private tokenBudget: TokenBudget;
  private retryState: RetryState = { count: 0, lastError: null, nextRetryAt: 0 };

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
}
```

---

## Codex CLI 实现

### Op/Event 提交-事件模式

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

**核心数据结构 -- Submission：**

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

**核心数据结构 -- Codex 结构体：**

`codex.rs`（~2000 行）定义了整个 Agent 系统的核心结构体：

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

**核心数据结构 -- Session 和 ActiveTurn：**

```rust
pub struct Session {
    pub conversation_id: ThreadId,
    pub state: Arc<RwLock<SessionState>>,
    pub services: Arc<SessionServices>,
}

pub struct ActiveTurn {
    pub turn_id: String,
    pub cancellation_token: CancellationToken,
    pub turn_context: Arc<TurnContext>,
}
```

### 提交循环 (Submission Loop)

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
    Interrupt,
    CleanBackgroundTerminals,
    RealtimeConversationStart(ConversationStartParams),
    RealtimeConversationAudio(ConversationAudioParams),
    RealtimeConversationText(ConversationTextParams),
    RealtimeConversationClose,
    RealtimeConversationListVoices,
    UserInput {
        items: Vec<UserInput>,
        final_output_json_schema: Option<Value>,
        responsesapi_client_metadata: Option<HashMap<String, String>>,
    },
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
    },
    ExecApproval { id: String, approved: bool },
    Shutdown,
}
```

### Responses API 调用格式

Codex CLI 使用 OpenAI **Responses API**（而非传统的 Chat Completions API），每次请求发送完整的 JSON 负载。

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

**input 按序插入规则：**

```
input 序列:
├── [0] developer 消息 — 沙箱权限说明
├── [1] developer 消息 — 开发者指令
├── [2] user 消息 — 用户指令
├── [3] user 消息 — 环境上下文
├── [4] user 消息 — 用户实际输入
├── [5+] 历史消息（压缩后的对话历史）
└── [最后] 当前用户输入
```

### SSE 流式处理

Codex CLI 通过 **Server-Sent Events (SSE)** 接收模型的流式响应。

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

**WebSocket 传输支持：**

除了标准 SSE，Codex CLI 还支持通过 WebSocket 传输响应流：

- **双向通信**：支持在流式传输过程中发送中断信号
- **增量请求**：WebSocket 支持增量式请求更新（incremental request tracking），减少重复传输
- **粘性路由**：WebSocket 连接保持会话亲和性，确保请求路由到同一后端实例

### 工具调用结果反馈

当模型发出工具调用后，Codex CLI 执行工具并将结果以 `function_call_output` 格式反馈给模型。

```json
{
  "type": "function_call_output",
  "call_id": "call_abc123",
  "output": "工具执行结果文本..."
}
```

### 无状态请求 + 前缀缓存

**关键设计 -- 旧提示词是新提示词的精确前缀：**

```
请求 N 的 input:
  [msg_0, msg_1, msg_2, ..., msg_n]

请求 N+1 的 input:
  [msg_0, msg_1, msg_2, ..., msg_n, tool_result_n+1]
  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  精确前缀匹配 — API 可以复用之前计算的 KV 缓存
```

**不使用 `previous_response_id` 的原因：**

1. **零数据保留（ZDR）**：不依赖服务端状态意味着可以实现零数据保留模式
2. **简化实现**：不需要处理服务端状态不一致、会话过期等问题
3. **前缀缓存优化**：精确前缀匹配在现代 LLM 推理引擎中已经非常高效

**stream_request 函数（核心请求流程）：**

```rust
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

## 对比分析

### 循环模型对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **循环模式** | 流式异步生成器（AsyncGenerator） | 提交-事件模式（SQ/EQ） |
| **核心函数** | `query()` async generator | Submission Loop + `stream_request()` |
| **事件传递** | `yield` / `yield*` 委托 | Event Queue (channel) |
| **用户输入** | 直接调用 `sendMessage()` | 通过 `Op::UserInput` 提交 |
| **中止机制** | AbortController 层级传递 | CancellationToken + Op::Interrupt |
| **状态管理** | 有状态（闭包变量 + messages 引用） | 无状态请求 + 前缀缓存 |

### 流式处理对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **流式协议** | Anthropic SSE（SDK 封装） | OpenAI SSE + WebSocket |
| **事件类型** | 18+ StreamEvent 类型 | 6+ ResponseEvent 类型 |
| **中间件** | StreamingToolExecutor 状态机 | 直接 SSE 解析 + Event Queue |
| **双向通信** | 不支持（单向 SSE） | 支持（WebSocket） |
| **流式中断** | AbortController.abort() | WebSocket 中断 + CancellationToken |

### 工具执行对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **执行时机** | 流式中并行启动（StreamingToolExecutor） | 响应完成后串行执行 |
| **并发策略** | isConcurrencySafe 判断 + 批次执行 | Guardian 审批 + 沙箱执行 |
| **最大并发** | 10 个工具并行 | 无显式并发限制 |
| **顺序保证** | nextOrder 计数器 + 排序 | 按工具调用顺序串行 |
| **错误传播** | siblingAbortController 级联 | CancellationToken 取消 |

### API 调用对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **API 类型** | Anthropic Messages API | OpenAI Responses API |
| **请求格式** | messages + system + tools | instructions + tools + input |
| **缓存策略** | cache_control: ephemeral | 前缀缓存（精确前缀匹配） |
| **状态依赖** | 有状态（messages 数组累积） | 无状态（每次发送完整历史） |
| **max_tokens 处理** | 最多 3 次自动恢复 | 上下文窗口超时处理 |

### 错误处理与重试对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **重试策略** | 指数退避（rate_limit/overloaded） | 移除最旧历史项后重试 |
| **上下文溢出** | forceCompact() + reactiveCompact() | 移除最旧历史项 |
| **错误分类** | 5 类（rate_limit/overloaded/context_overflow/auth/other） | ContextWindowExceeded 等 |
| **最大重试** | 3 次（max_tokens）/ 3 次（API 错误） | 动态（直到历史可压缩） |

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| AsyncGenerator 模型优雅，yield 事件零延迟传递给 UI | 有状态设计，messages 引用传递增加复杂度 |
| StreamingToolExecutor 实现流式并行工具执行，显著降低延迟 | StreamingToolExecutor 状态机复杂（~530 行），调试困难 |
| 18+ StreamEvent 类型覆盖所有场景，类型安全 | 事件类型过多，新增类型需修改多处代码 |
| max_tokens 自动恢复机制，对用户透明 | 恢复机制最多 3 次，复杂任务可能被截断 |
| QueryEngine 外层包装提供会话级重试和错误分类 | 重试逻辑与核心循环分离，状态同步需小心 |
| 微压缩 + 自动压缩集成在循环中，上下文管理无缝 | 压缩逻辑嵌入循环，增加了循环的复杂度 |
| AbortController 层级传播，中止信号精确控制 | 层级嵌套的 AbortController 管理复杂 |
| Prompt caching（ephemeral）显著降低重复输入成本 | 缓存仅限 Anthropic API，不适用于其他后端 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| SQ/EQ 模式解耦客户端和 Agent 循环，架构清晰 | 通道通信引入间接性，调试不如直接调用直观 |
| 无状态请求设计，零数据保留，隐私友好 | 每次发送完整历史，带宽消耗较大 |
| 前缀缓存优化，利用现代推理引擎的 KV 缓存 | 缓存效果依赖 API 服务端实现，不可控 |
| WebSocket 双向通信，支持流式中断 | WebSocket 连接管理复杂，需处理重连 |
| CancellationToken 协作式取消，Tokio 原生支持 | 取消粒度较粗，无法精确控制单个工具 |
| Op 枚举类型安全，serde 自动序列化/反序列化 | Op 变体较多，新增变体需修改多处匹配 |
| Responses API 简洁的三字段格式（instructions/tools/input） | 仅支持 OpenAI API，不兼容其他提供商 |
| Zstd 压缩支持，减少超长对话的传输数据量 | 压缩/解压增加 CPU 开销 |
| 远程压缩判断（should_use_remote_compact_task）灵活 | 远程压缩依赖 OpenAI 服务端，本地模型不可用 |
| drain_to_completed 递归模式简洁清晰 | 递归深度不受限，极端情况可能栈溢出 |
