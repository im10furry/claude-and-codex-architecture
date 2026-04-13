# 配置系统与状态管理对比

## Claude Code 实现

### CLAUDE.md 加载优先级和合并策略

CLAUDE.md 文件是 Claude Code 的项目级指令系统，支持 4 级目录层级和惰性加载机制：

```
┌─────────────────────────────────────────────────────────────────┐
│                 CLAUDE.md 4 级目录层级                           │
│                                                                 │
│  优先级    位置                              作用域              │
│  ──────    ────                              ────              │
│  最高      /etc/claude-code/CLAUDE.md       系统级（管理员）     │
│  高        ~/.claude/CLAUDE.md              用户级（全局）       │
│  中        ./.claude/CLAUDE.md              项目级（团队共享）   │
│  中        ./.claude/rules/*.md             项目规则（按文件匹配）│
│  低        ./CLAUDE.md                      项目级（本地）       │
│  最低      ./CLAUDE.local.md                本地级（不提交 Git） │
│                                                                 │
│  惰性加载机制：                                                 │
│  1. 启动时仅加载系统级和用户级 CLAUDE.md                        │
│  2. 项目级 CLAUDE.md 在进入项目目录时加载                      │
│  3. rules/*.md 按文件路径匹配条件加载                          │
│  4. CLAUDE.local.md 仅在本地存在时加载                          │
│  5. 文件变更通过 settingsChangeDetector + debounce 触发重新加载 │
└─────────────────────────────────────────────────────────────────┘
```

```typescript
// CLAUDE.md 加载逻辑（简化示意）
async function loadClaudeMdFiles(cwd: string): Promise<ClaudeMdContent[]> {
  const files: ClaudeMdContent[] = [];

  // 1. 系统级
  const systemPath = '/etc/claude-code/CLAUDE.md';
  if (await exists(systemPath)) {
    files.push({ path: systemPath, content: await readFile(systemPath), priority: 0 });
  }

  // 2. 用户级
  const userPath = path.join(os.homedir(), '.claude', 'CLAUDE.md');
  if (await exists(userPath)) {
    files.push({ path: userPath, content: await readFile(userPath), priority: 1 });
  }

  // 3. 项目级
  const projectPaths = [
    path.join(cwd, '.claude', 'CLAUDE.md'),
    path.join(cwd, 'CLAUDE.md'),
  ];
  for (const p of projectPaths) {
    if (await exists(p)) {
      files.push({ path: p, content: await readFile(p), priority: 2 });
    }
  }

  // 4. 项目规则（按路径匹配）
  const rulesDir = path.join(cwd, '.claude', 'rules');
  if (await exists(rulesDir)) {
    const ruleFiles = await readdir(rulesDir);
    for (const ruleFile of ruleFiles.filter(f => f.endsWith('.md'))) {
      files.push({
        path: path.join(rulesDir, ruleFile),
        content: await readFile(path.join(rulesDir, ruleFile)),
        priority: 2,
        isRule: true,
      });
    }
  }

  // 5. 本地级
  const localPath = path.join(cwd, 'CLAUDE.local.md');
  if (await exists(localPath)) {
    files.push({ path: localPath, content: await readFile(localPath), priority: 3 });
  }

  return files.sort((a, b) => a.priority - b.priority);
}
```

### settings.json 5 级优先级

```
┌─────────────────────────────────────────────────────────────────┐
│                 settings.json 5 级优先级                         │
│                                                                 │
│  优先级（从低到高）                                             │
│                                                                 │
│  1. flagSettings      特性标志默认值                            │
│     来源：代码内硬编码的默认值                                  │
│                                                                 │
│  2. localSettings     本地设置                                  │
│     来源：.claude/settings.local.json                           │
│     用途：开发者个人偏好，不提交到 Git                          │
│                                                                 │
│  3. projectSettings   项目设置                                  │
│     来源：.claude/settings.json                                 │
│     用途：团队共享的项目配置，提交到 Git                        │
│                                                                 │
│  4. userSettings      用户设置                                  │
│     来源：~/.claude/settings.json                               │
│     用途：用户全局偏好，跨项目生效                              │
│                                                                 │
│  5. policySettings    策略设置（最高优先级）                     │
│     来源：MDM 策略文件                                          │
│     用途：企业管理员强制策略，不可被用户覆盖                    │
│                                                                 │
│  合并策略：mergeWith + settingsMergeCustomizer                  │
│  - 数组字段：高优先级替换低优先级（非追加）                     │
│  - 对象字段：深度合并                                          │
│  - 布尔字段：高优先级覆盖低优先级                               │
│                                                                 │
│  热重载：settingsChangeDetector + debounce                      │
│  - 文件监视器检测 settings.json 变更                            │
│  - debounce 300ms 避免频繁重载                                  │
│  - 重载后触发 ConfigChange Hook                                │
└─────────────────────────────────────────────────────────────────┘
```

```typescript
// settings.ts -- 合并策略
import { mergeWith } from 'lodash';
import { settingsMergeCustomizer } from './settingsMerge';

function settingsMergeCustomizer(objValue: any, srcValue: any, key: string) {
  // 数组字段：替换而非追加
  if (Array.isArray(objValue) && Array.isArray(srcValue)) {
    return srcValue;
  }
  // 其他情况：使用 lodash 默认的深度合并
}

export async function loadSettings(): Promise<Settings> {
  const [flagSettings, localSettings, projectSettings,
         userSettings, policySettings] = await Promise.all([
    loadFeatureFlagSettings(),
    loadLocalSettings(),
    loadProjectSettings(),
    loadUserSettings(),
    loadPolicySettings(),
  ]);

  return mergeWith(
    {}, flagSettings, localSettings, projectSettings,
    userSettings, policySettings, settingsMergeCustomizer,
  );
}
```

### GrowthBook 特性标志系统

```
┌─────────────────────────────────────────────────────────────────┐
│                 GrowthBook 特性标志系统                          │
│                                                                 │
│  初始化属性：                                                   │
│  {                                                              │
│    organizationUUID: string,    // 组织 ID                     │
│    accountUUID: string,         // 账户 ID                     │
│    email: string,               // 用户邮箱                    │
│    platform: string,            // 操作系统 (darwin/linux/win32)│
│    claudeCodeVersion: string,   // Claude Code 版本号          │
│  }                                                              │
│                                                                 │
│  编译时 DCE vs 运行时远程配置轮询：                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 编译时 (bun:bundle feature())                           │   │
│  │  - 构建时完全剥离未激活的代码分支                        │   │
│  │  - 减小打包体积（如 VOICE_MODE ~200KB）                 │   │
│  │  - 适用于永久性功能开关                                  │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │ 运行时 (GrowthBook.isOn())                              │   │
│  │  - 从 CDN 轮询最新配置                                  │   │
│  │  - 支持实时开关（无需重新构建）                          │   │
│  │  - 支持 A/B 测试和灰度发布                               │   │
│  │  - 适用于需要灵活控制的开关                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Feature flag 随机词对命名（防止猜测）：                        │
│  - tengu_frond_boric        -> 禁用 bypass 权限                │
│  - alpine_ripple_flux       -> 控制 auto 模式可用性            │
│  - cobalt_glade_sprout      -> 会话镜像功能开关                │
│  - ember_pine_quartz        -> 协调器模式开关                  │
│                                                                 │
│  关键远程控制能力：                                             │
│  1. 禁用 bypass 权限（安全紧急响应）                           │
│  2. 控制 auto 模式可用性（安全策略调整）                       │
│  3. 会话镜像（企业审计）                                       │
│  4. 功能灰度发布（按用户百分比）                               │
│  5. A/B 测试（不同配置的性能对比）                             │
└─────────────────────────────────────────────────────────────────┘
```

### Bootstrap State 完整结构

```typescript
export interface BootstrapState {
  // --- 目录与会话 ---
  cwd: string;                    // 当前工作目录
  homeDir: string;                // 用户主目录
  sessionId: string;              // 会话唯一 ID
  conversationId: string;         // 对话 ID

  // --- 成本追踪 ---
  totalCost: number;              // 累计成本（美元）
  totalInputTokens: number;       // 累计输入 token 数
  totalOutputTokens: number;      // 累计输出 token 数
  totalCacheReadTokens: number;   // 累计缓存读取 token 数
  totalCacheCreationTokens: number; // 累计缓存创建 token 数

  // --- 性能指标 ---
  startupTime: number;            // 启动耗时（ms）
  apiLatencyP50: number;          // API 延迟 P50（ms）
  apiLatencyP99: number;          // API 延迟 P99（ms）
  toolExecutionCount: number;     // 工具执行总次数
  toolExecutionErrors: number;    // 工具执行错误次数

  // --- 认证安全 ---
  authToken: string | null;       // OAuth token
  authMethod: 'oauth' | 'api_key' | 'bare' | null;
  mdmPolicy: MDMPolicy | null;    // MDM 策略

  // --- 遥测 ---
  otelExporter: OTelExporter | null;
  statsigClient: StatsigClient | null;
  growthBook: GrowthBook | null;

  // --- Hooks ---
  hooks: Map<HookEvent, Hook[]>;

  // --- 特性标志 ---
  featureFlags: Map<string, boolean>;

  // --- 设置 ---
  settings: Settings;
}
```

### QueryEngine 状态管理

```typescript
interface QueryEngineState {
  // 预算执行
  tokenBudget: TokenBudget;
  estimatedTokens: number;

  // 重试状态
  retryCount: number;
  lastRetryError: Error | null;
  nextRetryAt: number;

  // 权限状态
  permissionMode: PermissionMode;
  deniedCount: number;
  consecutiveDenials: number;
  lastDeniedTool: string | null;

  // 会话统计
  messageCount: number;
  toolCallCount: number;
  compactCount: number;
}
```

### React Context 使用方式

Claude Code 使用自定义的极简 Store 实现替代 Zustand，通过 React Context 注入：

```typescript
// store.ts -- createStore (~20行)
type Listener<T> = (state: T) => void;

export function createStore<T extends object>(initialState: T) {
  let state = initialState;
  const listeners = new Set<Listener<T>>();

  return {
    getState: () => state,
    setState: (partial: Partial<T>) => {
      state = { ...state, ...partial };
      listeners.forEach(listener => listener(state));
    },
    subscribe: (listener: Listener<T>) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
  };
}

// AppStateProvider -- 防嵌套机制
export function AppStateProvider({ children }: { children: React.ReactNode }) {
  const storeRef = useRef<AppStore | null>(null);

  if (!storeRef.current) {
    storeRef.current = createStore<AppState>(defaultAppState);
  }

  // 防嵌套：如果已经存在 Provider，直接复用
  const existingStore = useContext(AppStoreContext);
  if (existingStore) {
    return <>{children}</>;
  }

  return (
    <AppStoreContext.Provider value={storeRef.current}>
      {children}
    </AppStoreContext.Provider>
  );
}
```

**核心 Context 列表**：

| Context | 用途 | 生命周期 |
|---------|------|----------|
| `AppStoreContext` | 全局应用状态（消息、权限、成本） | 会话级 |
| `NotificationsContext` | 通知队列和显示 | 会话级 |
| `StatsContext` | 会话统计信息 | 会话级 |
| `ModalContext` | 模态框状态管理 | 会话级 |
| `OverlayContext` | 覆盖层（diff viewer 等） | 会话级 |
| `VoiceContext` | 语音输入/输出状态 | 会话级 |
| `MailboxContext` | 多 Agent 消息路由 | 会话级 |

---

## Codex CLI 实现

### 配置文件发现与加载

Codex CLI 使用 **5 级优先级** 配置系统：

```
优先级从高到低:
┌──────────────────────────────────────────────────────────────┐
│  1. 环境变量 (CODEX_*)                                       │
│     CODEX_MODEL=gpt-4o                                       │
│     CODEX_SANDBOX=workspace-write                             │
│     OPENAI_API_KEY=sk-...                                    │
│                                                               │
│  2. CLI 标志 (--model, --sandbox, --approval-policy)          │
│     codex --model gpt-4o --sandbox workspace-write            │
│                                                               │
│  3. Profile (codex --profile <name>)                         │
│     ~/.codex/profiles/<name>.toml                             │
│                                                               │
│  4. 全局 config.toml                                          │
│     ~/.codex/config.toml                                      │
│                                                               │
│  5. 内置默认值                                                │
│     model = "o4-mini"                                         │
│     sandbox_mode = "read-only"                                │
│     approval_policy = "on-request"                            │
└──────────────────────────────────────────────────────────────┘
```

**ConfigLayerStack 合并：**

```rust
/// 配置层栈 -- 按优先级从低到高合并
pub struct ConfigLayerStack {
    layers: Vec<ConfigLayer>,
}

pub struct ConfigLayer {
    pub name: ConfigLayerSource,
    pub config: TomlValue,
    pub is_disabled: bool,
}

pub enum ConfigLayerSource {
    /// 内置默认值
    Builtin,
    /// 全局配置文件
    Global { path: PathBuf },
    /// 项目配置文件
    Project { path: PathBuf },
    /// 环境变量
    Environment,
    /// CLI 标志
    CliFlags,
    /// Profile
    Profile { name: String },
}
```

### AGENTS.md / codex.md 发现机制

核心实现在 `codex-rs/core/src/project_doc.rs`（315 行）。

**发现算法（4 步）：**

```
步骤 1: 确定项目根目录
┌──────────────────────────────────────────────────────────────┐
│  从当前工作目录向上遍历，查找 project_root_markers            │
│  默认标记: [".git"]                                          │
│  可配置: project_root_markers = [".git", "package.json"]     │
│                                                               │
│  /home/user/project/src/  <- cwd                              │
│  /home/user/project/      <- 找到 .git，确定为项目根          │
│  /home/user/                                                   │
│  /home/                                                       │
│  /                                                            │
└──────────────────────────────────────────────────────────────┘

步骤 2: 从项目根到 cwd 收集 AGENTS.md
┌──────────────────────────────────────────────────────────────┐
│  搜索路径（从根到 cwd）:                                      │
│  /home/user/project/AGENTS.md          <- 项目级指令          │
│  /home/user/project/src/AGENTS.md      <- 子目录指令          │
│                                                               │
│  候选文件名（按优先级）:                                       │
│  1. AGENTS.override.md  (本地覆盖，最高优先级)                │
│  2. AGENTS.md           (标准文件名)                          │
│  3. codex.md            (备用文件名，可配置)                   │
└──────────────────────────────────────────────────────────────┘

步骤 3: 读取并拼接内容
┌──────────────────────────────────────────────────────────────┐
│  大小限制: 32 KiB (project_doc_max_bytes)                     │
│  如果总大小超过限制，从最早发现的文件开始截断                   │
│  多个文件之间用 "\n\n" 分隔拼接                                │
└──────────────────────────────────────────────────────────────┘

步骤 4: 组装用户指令
┌──────────────────────────────────────────────────────────────┐
│  最终指令 = Config::instructions                              │
│            + "\n\n--- project-doc ---\n\n"                    │
│            + AGENTS.md 内容                                    │
│            + JS REPL 指令（如果启用）                           │
│            + 子 Agent 指令（如果启用）                          │
└──────────────────────────────────────────────────────────────┘
```

### 配置文件格式

完整的 TOML 配置文件示例：

```toml
# ~/.codex/config.toml

# 模型配置
model = "o4-mini"
model_provider = "openai"

# 沙箱模式
sandbox_mode = "workspace-write"

# 审批策略
approval_policy = "on-request"

# 模型提供商配置
[model_provider]
name = "openai"

# MCP 服务器配置
[mcp_servers]
my-server = { command = "npx", args = ["-y", "@my/mcp-server"] }
another-server = { command = "python", args = ["-m", "my_mcp"] }

# 项目文档配置
project_doc_max_bytes = 32768
project_doc_fallback_filenames = ["codex.md", "CODEX.md"]
project_root_markers = [".git", "package.json", "Cargo.toml"]

# 需求配置
[requirements]
# 定义项目特定的约束条件

# OpenTelemetry 配置
[otel]
enabled = true
endpoint = "http://localhost:4318"
```

### 模型路由

**ModelProviderInfo 注册表：**

```rust
/// 模型提供商信息
pub struct ModelProviderInfo {
    /// 提供商标识符
    pub id: String,

    /// 提供商名称
    pub name: String,

    /// 基础指令模板
    pub base_instructions: BaseInstructions,

    /// 支持的模型列表
    pub models: Vec<ModelInfo>,

    /// 是否为 OpenAI 提供商
    pub is_openai: bool,

    /// 流式请求最大重试次数
    pub stream_max_retries: usize,
}
```

**认证方式路由：**

| 认证方式 | 说明 | 配置 |
|----------|------|------|
| **ChatGPT OAuth** | 通过浏览器登录 ChatGPT 账户 | `codex login` |
| **API Key** | 直接使用 OpenAI API Key | `codex login --api-key` 或 `OPENAI_API_KEY` |
| **本地模型** | Ollama / LM Studio | `model_provider = "ollama"` |
| **Azure** | Azure OpenAI Service | `model_provider = "azure"` |

### 双层存储架构

Codex CLI 使用**双层存储架构**持久化会话状态：

```
┌──────────────────────────────────────────────────────────────┐
│                 双层存储架构                                   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 第一层: JSONL Rollout 文件                            │   │
│  │  ~/.codex/sessions/YYYY/MM/DD/                       │   │
│  │  rollout-2025-05-07T17-24-21-{uuid}.jsonl            │   │
│  │                                                       │   │
│  │  - 完整的会话事件流                                   │   │
│  │  - 可用 jq/fx 等工具直接查看                          │   │
│  │  - 支持会话恢复和分叉                                 │   │
│  │  - 人类可读的 JSON Lines 格式                         │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 第二层: SQLite 数据库                                 │   │
│  │  ~/.codex/state.db                                    │   │
│  │                                                       │   │
│  │  - 线程元数据索引                                     │   │
│  │  - 快速搜索和过滤                                     │   │
│  │  - 线程列表分页                                       │   │
│  │  - 从 Rollout 文件 read-repair                        │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### Rollout 文件格式

**RolloutLine 格式：**

每行是一个 JSON 对象，包含时间戳和事件类型：

```json
{"timestamp":"2025-05-07T17:24:21.123Z","item":{"SessionMeta":{...}}}
{"timestamp":"2025-05-07T17:24:22.456Z","item":{"ResponseItem":{...}}}
{"timestamp":"2025-05-07T17:24:23.789Z","item":{"EventMsg":{...}}}
{"timestamp":"2025-05-07T17:24:24.012Z","item":{"TurnContext":{...}}}
{"timestamp":"2025-05-07T17:24:25.345Z","item":{"Compacted":{...}}}
```

**RolloutItem 类型表：**

| 类型 | 说明 | 包含内容 |
|------|------|----------|
| `ResponseItem` | 模型响应项 | 文本消息、工具调用、工具结果 |
| `EventMsg` | 事件消息 | 执行命令开始/结束、审批请求等 |
| `TurnContext` | 轮次上下文 | 模型信息、沙箱策略、审批策略等 |
| `Compacted` | 压缩记录 | 压缩摘要、替换历史 |
| `SessionMeta` | 会话元数据 | 会话 ID、时间戳、cwd、Git 信息等 |

### RolloutRecorder

核心实现在 `codex-rs/rollout/src/recorder.rs`（1111 行）。

**异步写入架构：**

```rust
/// Rollout 记录器 -- 异步写入会话事件
#[derive(Clone)]
pub struct RolloutRecorder {
    /// 命令发送通道（有界队列，容量 256）
    tx: Sender<RolloutCmd>,

    /// 后台写入任务状态
    writer_task: Arc<RolloutWriterTask>,

    /// Rollout 文件路径
    pub(crate) rollout_path: PathBuf,

    /// SQLite 数据库句柄
    state_db: Option<StateDbHandle>,

    /// 事件持久化模式
    event_persistence_mode: EventPersistenceMode,
}

/// 写入命令
enum RolloutCmd {
    /// 添加事件项
    AddItems(Vec<RolloutItem>),

    /// 持久化（创建文件并写入所有缓冲项）
    Persist { ack: oneshot::Sender<std::io::Result<()>> },

    /// 刷新（将缓冲项写入已打开的文件）
    Flush { ack: oneshot::Sender<std::io::Result<()>> },

    /// 关闭（写入所有缓冲项后停止）
    Shutdown { ack: oneshot::Sender<std::io::Result<()>> },
}
```

### 会话恢复与分叉

**恢复流程：**

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

**分叉流程：**

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

### 配置文件格式对比

| 维度 | Claude Code (JSON) | Codex CLI (TOML) |
|------|--------------------|--------------------|
| **格式** | JSON (settings.json) | TOML (config.toml) |
| **可读性** | 较差（大括号嵌套、需引号） | 优秀（简洁、支持注释） |
| **注释支持** | 不支持（需 JSONC 扩展） | 原生支持 `#` 注释 |
| **类型安全** | 运行时验证 | 编译时类型检查（Rust serde） |
| **合并策略** | lodash mergeWith + 自定义 customizer | ConfigLayerStack 按优先级合并 |
| **热重载** | settingsChangeDetector + debounce 300ms | 无明确热重载机制 |

### 项目指令文件对比

| 维度 | CLAUDE.md | AGENTS.md |
|------|-----------|-----------|
| **文件名** | `CLAUDE.md` / `CLAUDE.local.md` | `AGENTS.md` / `AGENTS.override.md` / `codex.md` |
| **层级数** | 4 级（系统/用户/项目/本地） | 多级（项目根到 cwd 逐级收集） |
| **加载策略** | 惰性加载（按需） | 启动时一次性加载 |
| **大小限制** | 无明确限制 | 32 KiB (project_doc_max_bytes) |
| **规则匹配** | `.claude/rules/*.md` 按文件路径匹配 | 无独立规则匹配机制 |
| **拼接方式** | 按优先级排序后合并 | 按发现顺序用 `\n\n` 分隔拼接 |
| **覆盖机制** | 高优先级覆盖低优先级 | AGENTS.override.md 覆盖 AGENTS.md |

### 配置优先级对比

| 优先级 | Claude Code | Codex CLI |
|--------|-------------|-----------|
| **最高** | MDM 策略 (policySettings) | 环境变量 (CODEX_*) |
| **高** | 用户设置 (~/.claude/settings.json) | CLI 标志 (--model, --sandbox) |
| **中** | 项目设置 (.claude/settings.json) | Profile (~/.codex/profiles/) |
| **低** | 本地设置 (.claude/settings.local.json) | 全局配置 (~/.codex/config.toml) |
| **最低** | 特性标志默认值 (flagSettings) | 内置默认值 |

### 状态管理架构对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **状态管理** | React Context + 自定义 Store | Rust Mutex + Arc 共享状态 |
| **响应式** | 发布-订阅模式（listeners） | Mutex 锁保护 |
| **Context 数量** | 7 个核心 Context | 无明确分层 |
| **线程安全** | 单线程（Node.js） | 编译时保证（Rust 所有权） |
| **状态持久化** | JSONL 转录文件 | JSONL + SQLite 双层 |
| **索引/搜索** | sessions.json 手动索引 | SQLite 自动索引 |
| **跨会话记忆** | 无内置机制 | memories 模块 |

### 热重载机制对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **配置热重载** | settingsChangeDetector + debounce 300ms | 无明确热重载 |
| **指令热重载** | debounce 触发 CLAUDE.md 重新加载 | 无明确热重载 |
| **Hook 触发** | ConfigChange Hook | 无 |
| **特性标志** | GrowthBook CDN 实时轮询 | 无远程特性标志 |

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| **React Context 响应式**：发布-订阅模式确保 UI 实时响应状态变化，开发体验好 | **JSON 格式可读性差**：settings.json 不支持注释，配置维护成本高 |
| **GrowthBook 远程控制**：支持实时特性标志轮询、A/B 测试、灰度发布，运维能力强 | **无 SQLite 索引**：会话索引依赖手动维护的 sessions.json，搜索和过滤能力弱 |
| **惰性加载**：CLAUDE.md 按需加载，启动速度快，资源占用低 | **7 个 Context 复杂度**：多个 Context 增加了代码复杂度和理解成本 |
| **MDM 策略支持**：企业管理员可通过 MDM 策略强制配置，适合企业部署 | **无跨会话记忆**：缺少类似 Codex CLI 的 memories 模块，跨会话知识保持能力弱 |
| **热重载机制**：配置和指令文件变更自动检测并重载，无需重启 | **无 Profile 机制**：缺少配置 Profile，不同场景需要手动修改配置 |
| **规则匹配**：`.claude/rules/*.md` 按文件路径匹配，实现细粒度的目录级指令 | **无大小限制**：CLAUDE.md 无明确大小限制，可能导致 prompt 过长 |
| **防嵌套 Provider**：AppStateProvider 内置防嵌套机制，避免 Context 重复初始化 | **JSON 无注释**：配置文件无法添加说明注释，团队协作时理解成本高 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| **TOML 格式可读性优秀**：原生支持注释、语法简洁，配置维护成本低 | **无热重载**：配置变更需要重启会话，开发体验不如 Claude Code |
| **SQLite 索引**：state.db 提供高效的会话搜索、过滤和分页能力 | **无远程特性标志**：缺少 GrowthBook 类似的远程控制能力，运维灵活性不足 |
| **双层存储架构**：JSONL（人类可读）+ SQLite（高效索引），兼顾可读性和性能 | **无 MDM 策略**：缺少企业管理员强制策略机制，企业部署能力弱 |
| **Profile 机制**：支持命名 Profile（dev/staging/prod），一键切换配置场景 | **无惰性加载**：AGENTS.md 启动时一次性加载，可能影响启动速度 |
| **跨会话记忆**：memories 模块自动提取和注入跨会话知识，长期协作能力强 | **32 KiB 大小限制**：AGENTS.md 有严格大小限制，大型项目可能不够用 |
| **Rust 类型安全**：serde 编译时类型检查，配置错误在编译阶段就能发现 | **无规则匹配**：缺少按文件路径匹配的规则机制，指令粒度较粗 |
| **多种认证方式**：支持 ChatGPT OAuth、API Key、本地模型、Azure 等多种认证方式 | **无 ConfigChange Hook**：缺少配置变更的 Hook 机制，扩展性受限 |
| **模型提供商注册表**：ModelProviderInfo 统一管理多个模型提供商，切换方便 | **AGENTS.md 发现算法简单**：仅按目录层级收集，缺少更灵活的匹配策略 |
