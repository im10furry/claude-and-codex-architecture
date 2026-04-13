# IDE 集成与 LSP 对比

## Claude Code 实现

### IDE 集成 -- Bridge 协议

#### Bridge 协议架构

```
┌─────────────────────────────────────────────────────────────────┐
│                 Bridge 协议架构                                   │
│                                                                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────┐ │
│  │ IDE          │     │ Bridge Layer │     │ Claude Code Core │ │
│  │ Extension    │<───>│ (33 files)   │<───>│ (query.ts, etc)  │ │
│  │              │     │              │     │                  │ │
│  │ - VS Code    │     │ - Protocol   │     │ - QueryEngine    │ │
│  │ - JetBrains  │     │ - Transport  │     │ - Tool Registry  │ │
│  │ - Neovim     │     │ - Auth       │     │ - Permission     │ │
│  │ - Emacs      │     │ - Lifecycle  │     │ - Hooks          │ │
│  └──────────────┘     └──────────────┘     └──────────────────┘ │
│                                                                 │
│  通信协议：JSON-RPC over stdio / WebSocket                      │
│                                                                 │
│  核心文件列表（13 个关键文件）：                                 │
│  bridge/                                                        │
│  ├── protocol.ts           # Bridge 协议定义                    │
│  ├── transport.ts          # 传输层抽象（stdio/ws）              │
│  ├── auth.ts               # Bridge 认证                        │
│  ├── lifecycle.ts          # 生命周期管理                       │
│  ├── messageRouter.ts      # 消息路由                           │
│  ├── sessionManager.ts     # 会话管理                           │
│  ├── notificationHandler.ts # 通知处理                          │
│  ├── fileWatcher.ts        # 文件监视同步                       │
│  ├── diffViewer.ts         # Diff 查看器集成                    │
│  ├── planMode.ts           # Plan 模式集成                      │
│  ├── ideMcpServer.ts       # IDE MCP 服务器                     │
│  ├── contextProvider.ts    # 上下文提供者                       │
│  └── healthCheck.ts        # 健康检查                           │
└─────────────────────────────────────────────────────────────────┘
```

#### VS Code 集成

- **官方扩展**：Claude Code 提供 VS Code 扩展，通过 Bridge 协议与核心通信
- **IDE MCP 服务器**：每个 IDE 实例启动一个 MCP 服务器，Claude Code 通过 MCP 获取 IDE 上下文（打开的文件、选中的代码、诊断信息等）
- **实时同步**：文件变更、光标位置、选区变更实时同步到 Claude Code
- **diff viewer**：工具执行结果中的文件变更通过 IDE 的 diff viewer 展示
- **Plan 模式**：在 IDE 中显示 Claude 的执行计划，用户可以审查后批准

### LSP 集成

#### LSPClient.ts

Claude Code 内置了 LSP（Language Server Protocol）客户端，可以直接与语言服务器通信获取代码智能信息：

```typescript
// services/lsp/LSPClient.ts -- JSON-RPC over stdio

export class LSPClient {
  private process: ChildProcess;
  private messageId: number = 0;
  private pendingRequests: Map<number, {
    resolve: (result: any) => void;
    reject: (error: Error) => void;
  }> = new Map();
  private eventQueue: LSPEvent[] = [];
  private initialized: boolean = false;

  constructor(
    private serverCommand: string,
    private serverArgs: string[],
    private rootUri: string,
  ) {}

  async start(): Promise<void> {
    // 启动语言服务器进程
    this.process = spawn(this.serverCommand, this.serverArgs, {
      stdio: ['pipe', 'pipe', 'pipe'],
      cwd: this.rootUri,
    });

    // 监听 stdout 的 JSON-RPC 消息
    this.process.stdout.on('data', (data: Buffer) => {
      const messages = parseLSPMessages(data);
      for (const msg of messages) {
        this.handleMessage(msg);
      }
    });

    // 发送 initialize 请求
    await this.sendRequest('initialize', {
      processId: process.pid,
      rootUri: this.rootUri,
      capabilities: {
        textDocument: {
          completion: { completionItem: { snippetSupport: false } },
          hover: { contentFormat: ['markdown', 'plaintext'] },
          definition: { linkSupport: true },
          references: {},
          typeDefinition: { linkSupport: true },
          callHierarchy: { prepareSupport: true },
        },
      },
    });

    // 发送 initialized 通知
    this.sendNotification('initialized', {});
    this.initialized = true;
  }

  // 延迟队列：确保服务器初始化完成后再处理请求
  private async ensureInitialized(): Promise<void> {
    if (!this.initialized) {
      await new Promise(resolve => {
        const check = () => {
          if (this.initialized) resolve();
          else setTimeout(check, 100);
        };
        check();
      });
    }
  }
}
```

#### LSPServerManager.ts

管理多个 LSP 服务器实例，按文件扩展名路由到对应的语言服务器：

```typescript
// services/lsp/LSPServerManager.ts -- 多实例管理

export class LSPServerManager {
  private clients: Map<string, LSPClient> = new Map();
  private extensionMap: Map<string, string> = new Map();

  constructor() {
    // 文件扩展名 -> 语言服务器映射
    this.extensionMap.set('.ts', 'typescript');
    this.extensionMap.set('.tsx', 'typescript');
    this.extensionMap.set('.js', 'typescript');
    this.extensionMap.set('.jsx', 'typescript');
    this.extensionMap.set('.py', 'python');
    this.extensionMap.set('.go', 'gopls');
    this.extensionMap.set('.rs', 'rust-analyzer');
    this.extensionMap.set('.java', 'jdtls');
    this.extensionMap.set('.cpp', 'clangd');
    this.extensionMap.set('.c', 'clangd');
  }

  // 按需启动：第一次访问某语言时才启动对应的服务器
  async getClientForFile(filePath: string): Promise<LSPClient | null> {
    const ext = path.extname(filePath);
    const language = this.extensionMap.get(ext);

    if (!language) return null;

    if (!this.clients.has(language)) {
      const serverConfig = this.getServerConfig(language);
      if (!serverConfig) return null;

      const client = new LSPClient(
        serverConfig.command,
        serverConfig.args,
        this.rootUri,
      );
      await client.start();
      this.clients.set(language, client);
    }

    return this.clients.get(language)!;
  }

  // 关闭所有服务器
  async dispose(): Promise<void> {
    for (const client of this.clients.values()) {
      await client.dispose();
    }
    this.clients.clear();
  }
}
```

#### 支持的 LSP 操作（10 种）

| 操作 | LSP 方法 | 用途 |
|------|----------|------|
| 代码导航 | `textDocument/definition` | 跳转到定义 |
| 代码信息 | `textDocument/hover` | 获取类型信息、文档 |
| 查找引用 | `textDocument/references` | 查找所有引用位置 |
| 类型定义 | `textDocument/typeDefinition` | 跳转到类型定义 |
| 调用层次 | `callHierarchy/incomingCalls` | 查找调用者 |
| 调用层次 | `callHierarchy/outgoingCalls` | 查找被调用者 |
| 符号搜索 | `workspace/symbol` | 全局符号搜索 |
| 文件生命周期 | `textDocument/didOpen` | 通知服务器文件已打开 |
| 文件生命周期 | `textDocument/didChange` | 通知服务器文件已变更 |
| 文件生命周期 | `textDocument/didClose` | 通知服务器文件已关闭 |

---

## Codex CLI 实现

### IDE 集成 -- app-server 协议

Codex CLI 通过 `app-server` 协议与 IDE 集成，使用 **JSON-RPC 2.0** 双向通信。

#### 传输层

| 传输方式 | 说明 | 状态 |
|----------|------|------|
| **stdio** | 通过标准输入/输出通信（默认） | 稳定 |
| **websocket** | 通过 WebSocket 通信 | 实验性 |

#### 核心原语

| 原语 | 说明 |
|------|------|
| **Thread** | 对话线程（对应一个会话） |
| **Turn** | 对话轮次（一次用户输入到模型完成响应） |
| **Item** | 轮次中的项目（消息、工具调用、工具结果） |

#### 关键 API 列表

| API | 方法 | 说明 |
|-----|------|------|
| `thread/start` | POST | 创建新对话线程 |
| `thread/resume` | POST | 恢复已有线程 |
| `thread/fork` | POST | 分叉线程 |
| `turn/start` | POST | 开始新轮次 |
| `turn/steer` | POST | 引导当前轮次 |
| `turn/interrupt` | POST | 中断当前轮次 |
| `plugin/*` | POST | 插件管理 |
| `config/*` | POST | 配置管理 |
| `review/start` | POST | 开始代码审查 |
| `feedback/upload` | POST | 上传用户反馈 |

#### 初始化握手

```json
// 客户端 -> 服务器
{
  "jsonrpc": "2.0",
  "method": "initialize",
  "params": {
    "clientInfo": { "name": "vscode-codex", "version": "1.0.0" },
    "capabilities": {}
  },
  "id": 1
}

// 服务器 -> 客户端
{
  "jsonrpc": "2.0",
  "result": {
    "serverInfo": { "name": "codex-app-server", "version": "0.1.0" },
    "capabilities": {}
  },
  "id": 1
}

// 客户端 -> 服务器（确认初始化完成）
{
  "jsonrpc": "2.0",
  "method": "initialized",
  "params": {}
}
```

#### 背压处理

当客户端无法及时处理事件时，服务器通过有界队列实现背压：

```rust
/// 有界事件队列（防止内存溢出）
const EVENT_QUEUE_CAPACITY: usize = 1024;

/// 背压错误码
const BACKPRESSURE_ERROR_CODE: i32 = -32001;

// 当队列满时，返回背压错误
if event_queue.is_full() {
    return Err(JsonRpcError {
        code: BACKPRESSURE_ERROR_CODE,
        message: "Event queue full, client is too slow".to_string(),
        data: None,
    });
}
```

### LSP 集成

Codex CLI **没有内置 LSP 客户端**。代码智能功能依赖 IDE 自身的语言服务器提供。Codex CLI 的 IDE 集成主要通过 app-server 协议与 IDE 扩展通信，但不直接与语言服务器交互。

---

## 对比分析

### IDE 集成协议对比

| 维度 | Claude Code (Bridge) | Codex CLI (app-server) |
|------|---------------------|----------------------|
| **协议** | JSON-RPC over stdio/WebSocket | JSON-RPC 2.0 over stdio/WebSocket |
| **文件规模** | 33 文件，13 个关键文件 | 未公开详细文件数 |
| **认证** | JWT 认证 | capabilities 交换 |
| **IDE 支持** | VS Code、JetBrains、Neovim、Emacs | VS Code（社区扩展） |
| **实时同步** | 文件变更、光标位置、选区变更 | 无明确实时同步 |
| **diff viewer** | 集成 IDE diff viewer | 无 |
| **Plan 模式** | IDE 中显示执行计划 | 无 |
| **MCP 服务器** | IDE MCP 服务器提供上下文 | 无 IDE MCP |
| **线程管理** | 会话管理 | thread/start/resume/fork |
| **轮次控制** | 无 | turn/start/steer/interrupt |
| **背压处理** | 未明确 | 有界队列（1024 容量） |
| **代码审查** | 无 | review/start API |

### LSP 集成对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **内置 LSP 客户端** | 有（LSPClient.ts） | 无 |
| **多实例管理** | 有（LSPServerManager） | N/A |
| **支持语言数** | 10+ 种（TS/Python/Go/Rust/Java/C++ 等） | N/A |
| **LSP 操作数** | 10 种 | N/A |
| **按需启动** | 支持（首次访问时启动） | N/A |
| **代码导航** | 跳转定义、类型定义、引用查找 | 依赖 IDE |
| **调用层次** | incomingCalls/outgoingCalls | 依赖 IDE |
| **符号搜索** | workspace/symbol | 依赖 IDE |
| **文件生命周期** | didOpen/didChange/didClose | 依赖 IDE |

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| Bridge 协议 33 文件，IDE 集成最为完善 | Bridge 层代码量大，维护成本高 |
| 内置 LSP 客户端，独立于 IDE 提供代码智能 | LSP 服务器需用户本地安装，增加依赖 |
| 支持 4 种 IDE（VS Code/JetBrains/Neovim/Emacs） | 非 VS Code IDE 的集成质量可能不一致 |
| 实时同步光标位置和选区，交互体验优秀 | 实时同步增加通信开销 |
| IDE MCP 服务器提供丰富上下文（诊断/打开文件/选区） | 无背压处理机制，IDE 卡顿可能影响 CLI |
| diff viewer 集成，代码变更可视化效果好 | Plan 模式仅在部分 IDE 中支持 |
| 10 种 LSP 操作覆盖代码导航全链路 | LSPServerManager 按需启动有冷启动延迟 |
| JWT 认证保护 Bridge 通信安全 | JWT 密钥管理复杂度较高 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| JSON-RPC 2.0 标准协议，实现简洁 | IDE 集成功能相对简单 |
| 有界队列背压处理，防止内存溢出 | 无内置 LSP 客户端，代码智能依赖 IDE |
| thread/start/resume/fork 线程管理灵活 | 无实时同步（光标/选区/文件变更） |
| turn/start/steer/interrupt 轮次控制精细 | 无 diff viewer 集成 |
| review/start API 支持代码审查工作流 | 无 IDE MCP 服务器，上下文获取能力弱 |
| WebSocket 传输支持远程 IDE 连接 | WebSocket 传输仍为实验性 |
| capabilities 交换机制标准化 | capabilities 未充分利用（当前为空对象） |
| 轻量级协议，资源消耗低 | 仅 VS Code 有社区扩展，IDE 覆盖面窄 |
