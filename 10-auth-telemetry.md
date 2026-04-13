# 认证与遥测对比

## Claude Code 实现

### 遥测系统

#### 三层遥测架构

```
┌─────────────────────────────────────────────────────────────────┐
│                 三层遥测架构                                     │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 第 1 层：Statsig（运营指标）                              │   │
│  │  - 用户行为分析                                          │   │
│  │  - 功能使用统计                                          │   │
│  │  - A/B 实验数据                                          │   │
│  │  - 实时仪表盘                                            │   │
│  │  数据流向：Claude Code -> Statsig CDN                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 第 2 层：Sentry（错误日志）                               │   │
│  │  - 未捕获异常                                            │   │
│  │  - API 错误                                              │   │
│  │  - 工具执行失败                                          │   │
│  │  - 崩溃报告                                              │   │
│  │  数据流向：Claude Code -> Sentry CDN                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 第 3 层：OpenTelemetry（管理员监控）                      │   │
│  │  - 管理员可配置的导出器                                  │   │
│  │  - 自定义 Span 和 Metric                                 │   │
│  │  - 与企业可观测性平台集成                                │   │
│  │  数据流向：Claude Code -> OTEL Collector                 │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

#### 8 种核心指标

| 指标名称 | 类型 | 说明 |
|----------|------|------|
| `claude_code.session.count` | Counter | 会话总数 |
| `lines_of_code.count` | Counter | 生成/修改的代码行数 |
| `pull_request.count` | Counter | 创建的 PR 数量 |
| `commit.count` | Counter | 创建的 commit 数量 |
| `cost.usage` | Gauge | 当前会话成本（美元） |
| `token.usage` | Gauge | 当前会话 token 使用量 |
| `code_edit_tool.decision` | Histogram | 代码编辑工具的决策时间 |
| `active_time.total` | Gauge | 用户活跃时间（秒） |

#### 5 种核心事件

| 事件名称 | 触发时机 | 包含数据 |
|----------|----------|----------|
| `user_prompt` | 用户提交消息 | 消息长度、模式类型 |
| `tool_result` | 工具执行完成 | 工具名称、执行时长、是否成功 |
| `api_request` | API 调用完成 | 模型、token 用量、延迟、缓存命中率 |
| `api_error` | API 调用失败 | 错误类型、状态码、重试次数 |
| `tool_decision` | 权限决策完成 | 工具名称、决策结果（允许/拒绝/询问） |

#### 隐私保护措施

```
┌─────────────────────────────────────────────────────────────────┐
│                 隐私保护措施                                     │
│                                                                 │
│  1. Prompt 默认脱敏                                            │
│     - 用户 prompt 内容在发送到遥测前进行哈希处理                │
│     - 仅记录 prompt 的 token 数量，不记录内容                   │
│     - 工具输入中的敏感信息（路径、命令）进行模式替换             │
│                                                                 │
│  2. OTEL_LOG_USER_PROMPTS 环境变量                             │
│     - 默认 false：不记录用户 prompt 内容                        │
│     - 设为 true：记录完整的 prompt（仅用于调试）                │
│     - 仅在用户明确启用时生效                                    │
│                                                                 │
│  3. Bedrock/Vertex 禁用非必要流量                              │
│     - 使用 AWS Bedrock 或 GCP Vertex 时                        │
│     - 自动禁用 Statsig 和 GrowthBook 遥测                      │
│     - 仅保留 OpenTelemetry（由企业管理员控制）                  │
│     - 确保数据不离开企业基础设施                                │
│                                                                 │
│  4. 数据最小化原则                                            │
│     - 仅收集必要的指标和事件                                    │
│     - 不记录对话内容（除非用户明确启用）                        │
│     - 错误日志中自动脱敏文件路径和命令内容                      │
└─────────────────────────────────────────────────────────────────┘
```

### 认证系统

#### OAuth 2.0 PKCE 流程（8 步）

Claude Code 使用 OAuth 2.0 PKCE（Proof Key for Code Exchange）进行认证，而非传统的 OAuth 授权码流程。PKCE 的选择理由：
- **安全性**：即使授权码被截获，攻击者也无法交换 token（没有 code_verifier）
- **无需 client_secret**：CLI 应用无法安全存储 client_secret，PKCE 消除了这个需求
- **公共客户端友好**：原生应用和 CLI 工具的最佳实践

```
┌─────────────────────────────────────────────────────────────────┐
│                 OAuth 2.0 PKCE 8 步流程                         │
│                                                                 │
│  Step 1: 生成 PKCE 参数                                        │
│    code_verifier = randomURLSafeString(128)                    │
│    code_challenge = SHA256(code_verifier) -> base64url          │
│                                                                 │
│  Step 2: 打开浏览器授权页面                                    │
│    GET https://console.anthropic.com/oauth/authorize            │
│      ?client_id=claude-code                                    │
│      &response_type=code                                       │
│      &redirect_uri=http://localhost:PORT/callback              │
│      &code_challenge={code_challenge}                          │
│      &code_challenge_method=S256                               │
│      &scope=openid profile email                                │
│                                                                 │
│  Step 3: 用户在浏览器中登录并授权                               │
│    Anthropic 授权服务器验证用户身份                              │
│                                                                 │
│  Step 4: 回调重定向到本地 HTTP 服务器                           │
│    Claude Code 启动临时 HTTP 服务器监听回调                     │
│    GET http://localhost:PORT/callback?code=AUTH_CODE            │
│                                                                 │
│  Step 5: 用授权码交换 token                                     │
│    POST https://console.anthropic.com/oauth/token               │
│      {                                                          │
│        grant_type: 'authorization_code',                        │
│        code: AUTH_CODE,                                         │
│        redirect_uri: 'http://localhost:PORT/callback',          │
│        client_id: 'claude-code',                                │
│        code_verifier: CODE_VERIFIER  // PKCE 关键步骤           │
│      }                                                          │
│                                                                 │
│  Step 6: 获取 access_token + refresh_token                     │
│    {                                                          │
│      access_token: 'eyJ...',                                   │
│      refresh_token: 'eyJ...',                                  │
│      expires_in: 3600,                                         │
│      token_type: 'Bearer'                                      │
│    }                                                          │
│                                                                 │
│  Step 7: 存储 token 到安全存储                                  │
│    macOS -> Keychain                                           │
│    Windows/Linux -> ~/.claude/credentials.json (加密)           │
│                                                                 │
│  Step 8: 关闭临时 HTTP 服务器                                   │
└─────────────────────────────────────────────────────────────────┘
```

#### Token 管理

```typescript
// Token 管理策略

interface TokenManager {
  accessToken: string | null;
  refreshToken: string | null;
  expiresAt: number | null;

  // 5 分钟缓冲：在 token 过期前 5 分钟主动刷新
  REFRESH_BUFFER_MS: 300_000;

  async getValidToken(): Promise<string> {
    // 检查是否有有效 token
    if (this.accessToken && this.expiresAt) {
      const now = Date.now();
      const buffer = this.REFRESH_BUFFER_MS;

      if (now < this.expiresAt - buffer) {
        return this.accessToken; // token 仍然有效
      }

      // 即将过期，主动刷新
      try {
        await this.refreshAccessToken();
        return this.accessToken!;
      } catch (error) {
        // 刷新失败，使用旧 token 直到它真正过期
        if (now < this.expiresAt) {
          return this.accessToken!;
        }
        throw error;
      }
    }

    // 没有 token，需要重新认证
    throw new Error('No valid token. Please run claude auth login.');
  }
}
```

#### 认证解析链（6 级优先级）

```
┌─────────────────────────────────────────────────────────────────┐
│                 认证解析链（6 级优先级）                          │
│                                                                 │
│  优先级    来源                    说明                          │
│  ──────    ────                    ────                          │
│  最高      3P context              第三方集成（IDE/SDK）提供     │
│            的预配置认证                                           │
│                                                                 │
│  高        bare mode                CLAUDE_CODE_BARE=1           │
│            无认证模式              直接使用 API（无 OAuth）       │
│                                                                 │
│  中高      managed OAuth            企业 MDM 管理的 OAuth        │
│            MDM 策略提供            策略强制指定认证方式          │
│                                                                 │
│  中        explicit tokens          环境变量显式指定              │
│            ANTHROPIC_API_KEY        API key（直接使用）          │
│            ANTHROPIC_AUTH_TOKEN     OAuth token（直接使用）      │
│                                                                 │
│  中低      OAuth                    本地存储的 OAuth token       │
│            Keychain/credentials     自动刷新                    │
│                                                                 │
│  最低      API key                  ~/.claude/credentials        │
│            本地存储的 API key       中的 api_key 字段            │
└─────────────────────────────────────────────────────────────────┘
```

#### 安全存储

```
┌─────────────────────────────────────────────────────────────────┐
│                 安全存储策略                                     │
│                                                                 │
│  macOS:                                                         │
│    - 使用系统 Keychain 存储 OAuth token 和 refresh token        │
│    - 服务名称: "com.anthropic.claude-code"                      │
│    - Keychain 缓存 TTL: 5 分钟（避免频繁 Keychain 访问）       │
│    - Stale-while-error: Keychain 访问失败时使用内存缓存         │
│                                                                 │
│  Windows / Linux:                                               │
│    - 使用 ~/.claude/credentials.json（明文）                    │
│    - 文件权限设置为 600（仅用户可读写）                         │
│    - 无 Keychain 缓存（每次直接读取文件）                       │
│    - 未来计划支持 libsecret (Linux) 和 Credential Manager (Win) │
│                                                                 │
│  Token 刷新调度：                                               │
│    - 启动时检查 token 有效性                                    │
│    - 每次 API 调用前检查 token 有效性                          │
│    - 过期前 5 分钟主动刷新                                      │
│    - 刷新失败时使用旧 token 直到真正过期                        │
│    - refresh_token 过期时触发重新认证流程                       │
└─────────────────────────────────────────────────────────────────┘
```

#### MDM 策略（3 平台路径）

```
┌─────────────────────────────────────────────────────────────────┐
│                 MDM 策略（3 平台路径）                           │
│                                                                 │
│  macOS:                                                         │
│    /Library/Managed Preferences/com.anthropic.claude-code.plist  │
│    - 通过 MDM 配置文件推送                                      │
│    - 支持强制权限模式、禁用功能、配置 MCP 服务器               │
│                                                                 │
│  Windows:                                                       │
│    HKLM\Software\Policies\Anthropic\ClaudeCode                  │
│    - 通过注册表推送                                             │
│    - 支持 Group Policy 配置                                     │
│                                                                 │
│  Linux:                                                         │
│    /etc/claude-code/policy.json                                 │
│    - 通过配置管理工具推送                                       │
│    - 支持 Ansible/Puppet/Chef 集成                              │
│                                                                 │
│  MDM 可控制的策略：                                             │
│  - permissionMode: 强制权限模式                                 │
│  - disabledFeatures: 禁用功能列表                               │
│  - allowedMcpServers: 允许的 MCP 服务器白名单                   │
│  - maxCostPerSession: 单会话最大成本                            │
│  - auditLogEnabled: 是否启用审计日志                            │
│  - telemetryEnabled: 是否启用遥测                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Codex CLI 实现

### 遥测系统

#### 内置 OpenTelemetry SDK

Codex CLI 内置了 OpenTelemetry SDK，通过 `codex-otel` crate 实现：

```toml
# config.toml 中的 OTel 配置
[otel]
enabled = true
endpoint = "http://localhost:4318"  # OTLP exporter endpoint
```

#### 5 种核心指标

| 指标名称 | 类型 | 说明 |
|----------|------|------|
| `feature.state` | Gauge | 特性标志状态 |
| `approval.requested` | Counter | 审批请求次数 |
| `tool.call` | Counter | 工具调用次数 |
| `conversation.turn.count` | Counter | 对话轮次计数 |
| `shell_snapshot` | Histogram | Shell 命令执行快照 |

#### AnalyticsEventsClient

```rust
/// 分析事件客户端
pub struct AnalyticsEventsClient {
    /// 事件发送端点
    endpoint: String,

    /// 安装 ID
    installation_id: String,
}

impl AnalyticsEventsClient {
    /// 发送分析事件
    pub async fn track_event(&self, event: AnalyticsEvent) {
        // POST /codex/analytics-events/events
    }

    /// 跟踪压缩事件
    pub async fn track_compaction(&self, event: CodexCompactionEvent) {
        // 记录压缩触发原因、持续时间、token 变化等
    }
}
```

**插件遥测事件：**

```rust
// 插件安装/禁用事件
AnalyticsEvent::PluginInstalled { plugin_id, version }
AnalyticsEvent::PluginDisabled { plugin_id, reason }
```

#### 隐私保护

- **默认不记录 prompt**：系统提示和用户输入不会被发送到遥测端点
- **LLM_CLI_TELEMETRY_DISABLED**：设置此环境变量可完全禁用遥测
- **本地优先**：所有会话数据存储在本地 `~/.codex/` 目录

### 认证系统

#### ChatGPT 账户登录（3 种流程）

**浏览器环境（localhost:1455）：**

```
┌──────────────────────────────────────────────────────────────┐
│  ChatGPT OAuth 流程（浏览器环境）                             │
│                                                               │
│  1. codex login                                              │
│     │                                                        │
│     ▼                                                        │
│  2. 启动本地 HTTP 服务器 (localhost:1455)                     │
│     │                                                        │
│     ▼                                                        │
│  3. 打开默认浏览器，跳转到 ChatGPT 授权页面                    │
│     │                                                        │
│     ▼                                                        │
│  4. 用户在浏览器中登录并授权                                   │
│     │                                                        │
│     ▼                                                        │
│  5. ChatGPT 重定向到 localhost:1455/callback?code=...        │
│     │                                                        │
│     ▼                                                        │
│  6. 本地服务器接收授权码，交换 access_token                    │
│     │                                                        │
│     ▼                                                        │
│  7. 存储 token 到 keyring                                     │
└──────────────────────────────────────────────────────────────┘
```

**无浏览器环境（设备码认证流程 3 步）：**

```
1. codex login
   -> 显示设备码: XXXX-XXXX
   -> 显示验证 URL: https://chatgpt.com/device

2. 用户在其他设备上打开 URL，输入设备码

3. codex login 轮询等待授权完成
   -> 授权成功，存储 token
```

**远程服务器（SSH 端口转发）：**

```bash
# 在远程服务器上
codex login

# 在本地机器上建立端口转发
ssh -L 1455:localhost:1455 user@remote-server

# 然后在本地浏览器中完成授权
```

#### API Key 登录

```bash
# 方式 1: 通过命令行
codex login --api-key sk-...

# 方式 2: 通过环境变量
export OPENAI_API_KEY=sk-...
```

#### keyring-store 密钥管理

密钥管理通过 `codex-keyring-store` crate 实现，支持 4 个平台：

| 平台 | 后端 | 说明 |
|------|------|------|
| **macOS** | Keychain | 系统密钥链 |
| **Windows** | Credential Manager | Windows 凭据管理器 |
| **Linux** | DBus + keyutils | 通过 DBus 协议访问系统密钥环 |
| **FreeBSD** | DBus | 通过 DBus 协议 |

**三种存储模式：**

```rust
pub enum OAuthCredentialsStoreMode {
    /// 自动选择（优先 Keyring，回退到文件）
    Auto,

    /// 仅使用 Keyring
    Keyring,

    /// 仅使用文件存储
    File,
}
```

**安全设计：**

```rust
/// 密钥服务名称
const KEYRING_SERVICE: &str = "codex-cli";

/// 使用 SHA-256 哈希作为存储键
fn hash_key(key: &str) -> String {
    use sha2::{Sha256, Digest};
    let mut hasher = Sha256::new();
    hasher.update(key.as_bytes());
    format!("{:x}", hasher.finalize())
}

/// 刷新令牌时间偏移（30 秒）
const REFRESH_SKEW_MILLIS: u64 = 30_000;
```

---

## 对比分析

### 遥测架构对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **架构层次** | 三层（Statsig + Sentry + OTel） | 单层（内置 OTel + Analytics） |
| **核心指标数** | 8 种 | 5 种 |
| **核心事件数** | 5 种 | 通过 AnalyticsEventsClient 自定义 |
| **A/B 测试** | 支持（Statsig + GrowthBook） | 不支持 |
| **错误追踪** | Sentry 集成 | 无独立错误追踪 |
| **企业可观测性** | OTel 管理员可配置导出器 | OTel endpoint 配置 |
| **完全禁用** | 无全局开关 | `LLM_CLI_TELEMETRY_DISABLED` 环境变量 |
| **Bedrock/Vertex** | 自动禁用非必要遥测 | N/A |
| **Prompt 脱敏** | 默认哈希处理 | 默认不记录 |

### 认证流程对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **OAuth 流程** | PKCE 8 步 | 标准 OAuth + 设备码 + SSH 转发 |
| **认证流程数** | 1 种（PKCE） | 3 种（浏览器/设备码/SSH） |
| **认证解析链** | 6 级优先级 | 未明确分层 |
| **API Key 支持** | 环境变量 | 命令行 + 环境变量 |
| **Bare Mode** | 支持（CLAUDE_CODE_BARE=1） | 无对应模式 |
| **MDM 策略** | 3 平台路径 | 无 |
| **Token 缓冲** | 5 分钟 | 30 秒 |

### 密钥存储方案对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **macOS** | Keychain（5 分钟缓存） | Keychain |
| **Windows** | credentials.json（权限 600） | Credential Manager |
| **Linux** | credentials.json（权限 600） | DBus + keyutils |
| **FreeBSD** | 不支持 | DBus |
| **存储模式** | 固定（平台决定） | 3 种（Auto/Keyring/File） |
| **密钥哈希** | 无 | SHA-256 哈希存储键 |
| **Stale-while-error** | 支持（Keychain 缓存） | 未提及 |

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| 三层遥测架构覆盖运营、错误、企业监控，全面性强 | 遥测架构复杂，依赖多个外部服务（Statsig/Sentry/OTel） |
| 8 种核心指标 + 5 种核心事件，覆盖业务全链路 | 无全局遥测禁用开关，隐私控制粒度不够 |
| Bedrock/Vertex 自动禁用非必要遥测，企业友好 | Statsig 和 Sentry 数据发送到第三方 CDN |
| OAuth PKCE 8 步流程安全性高，无 client_secret 泄露风险 | 仅支持 1 种认证流程，无设备码/SSH 转发支持 |
| 6 级认证解析链覆盖所有使用场景（IDE/MDM/API Key/OAuth） | Windows/Linux 使用明文 credentials.json 存储 |
| MDM 策略支持 3 平台，企业部署友好 | Linux/Windows 缺少系统级密钥链支持 |
| 5 分钟 Token 刷新缓冲，减少认证中断 | Keychain 缓存仅 macOS 支持 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| 3 种认证流程（浏览器/设备码/SSH），适应各种环境 | 无 PKCE 保护（依赖标准 OAuth 流程） |
| `LLM_CLI_TELEMETRY_DISABLED` 全局禁用开关，隐私友好 | 遥测架构简单，缺少 A/B 测试和错误追踪 |
| keyring-store 支持 4 平台，包括 FreeBSD | 仅 5 种核心指标，覆盖面有限 |
| 3 种存储模式（Auto/Keyring/File）灵活可控 | 无 MDM 策略支持，企业部署能力弱 |
| SHA-256 哈希存储键，安全性更高 | 无认证解析链，优先级不明确 |
| Windows 使用 Credential Manager，安全性优于明文文件 | 30 秒刷新偏移较短，可能频繁触发刷新 |
| API Key 支持命令行直接传入，使用便捷 | 无 bare mode 或第三方集成认证支持 |
