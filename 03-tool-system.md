# 工具系统对比：Claude Code vs Codex CLI

## Claude Code 实现

### 工具注册表

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

### Tool 接口完整定义

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

### buildTool() 工厂函数和默认值

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

### runTools() 编排实现

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

### isConcurrencySafe 判断逻辑

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

### siblingAbortController 错误传播

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

### 延迟加载（ToolSearchTool）

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

## Codex CLI 实现

### 内置工具

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

### 工具 Schema 系统

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

### apply_patch.rs 文件编辑

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

## 对比分析

### 工具数量与分类对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **内置工具总数** | 40+ | 25+ |
| **核心文件操作** | Read, Write, Edit（search-and-replace） | apply_patch（unified diff） |
| **Shell 执行** | Bash（直接执行） | shell（沙箱执行） |
| **搜索工具** | Glob, Grep | 无独立搜索工具（依赖 shell） |
| **任务管理** | TaskCreate/Get/Update/List/Output/Stop | 无 |
| **计划模式** | EnterPlanMode, ExitPlanMode | 无 |
| **Web 工具** | WebFetch, WebSearch | web_search（API 提供） |
| **MCP 工具** | 自研实现（4 种传输） | rmcp 0.12 |
| **Agent 工具** | AgentTool, SkillTool | 无 |
| **特性门控工具** | 6+（KAIROS/COORDINATOR/WORKFLOW 等） | js_repl（feature-gated） |
| **斜杠命令** | 87+ | N/A |

### 文件编辑策略对比

| 维度 | Claude Code（search-and-replace） | Codex CLI（apply_patch / unified diff） |
|------|----------------------------------|----------------------------------------|
| **编辑方式** | 搜索旧内容 -> 替换为新内容 | 声明式 diff（+/- 行标记） |
| **定位机制** | 精确字符串匹配 | 上下文行（@@ 标记）+ seek_sequence |
| **多文件编辑** | 每次调用编辑一个文件 | 单次 patch 可编辑多个文件 |
| **文件创建** | Write 工具 | `*** Add File` hunk |
| **文件删除** | Bash 工具 | `*** Delete File` hunk |
| **文件重命名** | Bash 工具 | `*** Move to` 支持 |
| **冲突处理** | 精确匹配失败则报错 | 多级 @@ 标记 + EOF 容错 + 宽松模式 |
| **模型友好度** | 高（简单直观） | 中（需学习 patch 格式） |
| **原子性** | 单文件原子操作 | 多文件原子 patch |

### 工具接口设计对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **接口定义** | TypeScript interface（13 个字段） | Rust struct（ToolSpec + ToolDefinition） |
| **输入验证** | validateInput? 可选方法 | serde JSON Schema 自动验证 |
| **权限检查** | checkPermissions? 可选方法 | Guardian 审批系统 |
| **并发安全** | isConcurrencySafe() 方法 | 无显式并发控制 |
| **工厂模式** | buildTool() 工厂函数 | ToolDefinition handler 闭包 |
| **延迟加载** | shouldDefer + ToolSearchTool | tool_search / tool_suggest |
| **特性门控** | GrowthBook feature flags | Rust cfg features |

### 并发模型对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **并发策略** | 安全/不安全批次分离 | 无显式并发（串行执行） |
| **最大并发数** | 10 | N/A |
| **安全判断** | 静态工具名白名单 | N/A |
| **错误传播** | siblingAbortController 级联 | CancellationToken 取消 |
| **顺序保证** | nextOrder 计数器 + 排序 | 按调用顺序串行 |
| **流式并行** | StreamingToolExecutor（边生成边执行） | 响应完成后执行 |

### 延迟加载策略对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **实现方式** | ToolSearchTool（专用工具） | tool_search / tool_suggest 函数 |
| **触发方式** | 模型主动调用 ToolSearch | 按需搜索 / 上下文推荐 |
| **Token 节省** | ~24,400 tokens/轮次（50 MCP 工具） | 类似效果 |
| **加载粒度** | 单个工具 schema | 搜索结果集 |
| **MCP 支持** | 全部 MCP 工具延迟加载 | MCP 工具动态发现 |

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| 40+ 工具覆盖全面，功能极其丰富 | 工具数量多导致系统提示词膨胀 |
| search-and-replace 编辑方式直观，模型易学 | 单次调用只能编辑一个文件，多文件修改效率低 |
| isConcurrencySafe 白名单 + StreamingToolExecutor 实现真正的流式并行 | 并发安全判断基于静态工具名，无法根据输入动态判断 |
| ToolSearchTool 延迟加载显著节省 token（~24,400/轮次） | 延迟加载增加了一轮额外的工具调用开销 |
| buildTool() 工厂函数统一默认值，工具开发简洁 | Tool 接口字段过多（13 个），实现完整工具成本高 |
| Hook 系统（PreToolUse/PostToolUse）提供工具执行前后拦截 | Hook 执行增加延迟，可能影响工具并发执行 |
| siblingAbortController 三级错误传播，中止控制精细 | 层级嵌套的 AbortController 管理复杂 |
| GrowthBook 特性门控灵活控制工具可用性 | 特性标志依赖远程服务，离线时可能失效 |
| 87+ 斜杠命令提供丰富的快捷操作 | 斜杠命令与工具系统分离，维护两套体系 |
| TaskCreate/Get/Update/List/Output/Stop 完整的任务管理 | 任务管理工具增加系统复杂度 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| apply_patch 支持单次多文件原子编辑，效率高 | patch 格式对模型生成质量要求高，格式错误率高 |
| 独立 crate（codex-apply-patch），可单独测试和运行 | 解析器复杂（parser.rs 741 行 + lib.rs 1672 行） |
| 宽松模式（Lenient）兼容 GPT-4.1 heredoc 格式 | 宽松模式增加解析器复杂度 |
| seek_sequence + @@ 标记精确定位，冲突处理多级 | 上下文匹配不如精确字符串匹配可靠 |
| `*** Move to` 原生支持文件重命名 | 重命名功能依赖 patch 格式正确性 |
| Guardian 审批系统与工具执行深度集成 | 审批流程增加工具执行延迟 |
| 沙箱执行所有 shell 命令，安全性高 | 沙箱限制可能阻止合法操作 |
| serde JSON Schema 自动验证输入 | 验证错误信息不如 Zod 友好 |
| tool_search / tool_suggest 双模式动态发现 | 动态发现机制不如 ToolSearchTool 成熟 |
| 工具数量精简（25+），系统提示词紧凑 | 功能覆盖面不如 Claude Code（缺少搜索、任务管理等） |
| shell 工具统一执行入口，简洁高效 | 缺少独立的 Glob/Grep 工具，搜索依赖 shell 命令 |
