# 计费与生态系统对比

## Claude Code 实现

### API 按 Token 计费

Claude Code 通过 Anthropic API 按使用量计费，支持多种 API 后端：

| 模型 | 输入价格 | 输出价格 | 上下文窗口 |
|------|----------|----------|------------|
| **Claude Sonnet 4** | $3 / 1M tokens | $15 / 1M tokens | 200K |
| **Claude Opus 4** | $15 / 1M tokens | $75 / 1M tokens | 200K |
| **Claude Haiku 3.5** | $0.80 / 1M tokens | $4 / 1M tokens | 200K |

### API 后端支持

Claude Code 支持 4 个 API 后端，计费方式各不相同：

| 后端 | 计费方式 | 说明 |
|------|----------|------|
| **Anthropic API** | 按 token 计费 | 直接使用 Anthropic API |
| **AWS Bedrock** | 独立计费 | 通过 AWS 账户计费，支持企业协议价 |
| **GCP Vertex** | 独立计费 | 通过 GCP 账户计费，支持企业协议价 |
| **Anthropic Foundry** | 独立计费 | 定制模型微调 |

### 企业订阅

| 计划 | 价格 | 说明 |
|------|------|------|
| **Claude Max** | $100/月/seat | 个人高级订阅 |
| **Claude Team** | $200/月/seat | 团队协作订阅 |

### 成本追踪

Claude Code 内置成本追踪功能：

- **`cost.usage` 指标**：实时追踪当前会话成本（美元）
- **`token.usage` 指标**：实时追踪 token 使用量
- **`maxCostPerSession` MDM 策略**：企业管理员可设置单会话最大成本

### 闭源模式

- **许可证**：闭源
- **自托管**：不支持
- **代码透明度**：无（仅通过逆向分析了解架构）
- **社区贡献**：不接受外部贡献

---

## Codex CLI 实现

### ChatGPT 订阅

| 计划 | 价格 | 说明 |
|------|------|------|
| **ChatGPT Plus** | $20/月 | 含 Codex CLI 使用（有使用限额） |
| **ChatGPT Pro** | $200/月 | 更高使用限额 |

### API 按 Token 计费

Codex CLI 通过 OpenAI API 按使用量计费：

| 模型 | 输入价格 | 输出价格 | 上下文窗口 |
|------|----------|----------|------------|
| **GPT-4.1** | $2 / 1M tokens | $8 / 1M tokens | 1M |
| **GPT-4.1 mini** | $0.40 / 1M tokens | $1.60 / 1M tokens | 1M |
| **GPT-4.1 nano** | $0.10 / 1M tokens | $0.40 / 1M tokens | 1M |
| **o3** | $10 / 1M tokens | $40 / 1M tokens | 200K |
| **o4-mini** | $1.10 / 1M tokens | $4.40 / 1M tokens | 200K |

### 本地模型（免费）

Codex CLI 原生支持本地模型运行，**完全免费**：

| 提供商 | 说明 |
|--------|------|
| **Ollama** | 开源本地模型运行时，支持 Llama、Mistral 等 |
| **LM Studio** | 桌面应用，支持下载和运行开源模型 |

本地模型的优势：
- **零 API 成本**：所有推理在本地完成
- **数据隐私**：代码不离开本地机器
- **无网络依赖**：离线可用
- **无限使用**：不受 API 限额限制

### 开源模式

- **许可证**：Apache 2.0 开源
- **自托管**：完全支持
- **代码透明度**：完全透明（~8 万行 Rust 源码公开）
- **社区贡献**：421+ 贡献者，75,000+ GitHub Stars

---

## 对比分析

### 计费模式对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **免费使用** | 无（需 API Key 或订阅） | 有（本地模型免费） |
| **订阅入口** | Claude Max $100/月 | ChatGPT Plus $20/月 |
| **订阅高端** | Claude Team $200/月 | ChatGPT Pro $200/月 |
| **最低 API 价格** | Haiku: $0.80/$4 per 1M | GPT-4.1 nano: $0.10/$0.40 per 1M |
| **最高 API 价格** | Opus 4: $15/$75 per 1M | o3: $10/$40 per 1M |
| **主流模型价格** | Sonnet 4: $3/$15 per 1M | GPT-4.1: $2/$8 per 1M |
| **上下文窗口** | 200K | 最高 1M（GPT-4.1 系列） |
| **企业 API** | Bedrock/Vertex 独立计费 | 无独立企业 API |
| **成本追踪** | 内置（cost.usage 指标） | 无内置成本追踪 |
| **会话成本限制** | MDM maxCostPerSession | 无 |

### 生态系统对比

| 维度 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **许可证** | 闭源 | Apache 2.0 开源 |
| **GitHub Stars** | 未公开 | 75,000+ |
| **贡献者** | 未公开 | 421+ |
| **MCP 生态** | 自研 MCP 实现（4 种传输） | rmcp 0.12 标准 |
| **插件系统** | 有（plugins/） | 有（plugin/* API） |
| **IDE 集成** | VS Code/JetBrains/Neovim/Emacs | VS Code（社区扩展） |
| **本地模型** | 不支持 | Ollama + LM Studio |
| **模型提供商** | Anthropic（4 后端） | OpenAI + Ollama + LM Studio |
| **社区生态** | Anthropic 官方主导 | 开源社区驱动 |
| **自托管** | 不支持 | 完全支持 |
| **企业部署** | MDM 策略 + Bedrock/Vertex | 开源 + 自定义部署 |

### API 成本效率对比

以处理 100K input tokens + 50K output tokens 为例：

| 模型 | 平台 | 输入成本 | 输出成本 | 总成本 |
|------|------|----------|----------|--------|
| Claude Sonnet 4 | Anthropic | $0.30 | $0.75 | **$1.05** |
| Claude Opus 4 | Anthropic | $1.50 | $3.75 | **$5.25** |
| Claude Haiku 3.5 | Anthropic | $0.08 | $0.20 | **$0.28** |
| GPT-4.1 | OpenAI | $0.20 | $0.40 | **$0.60** |
| GPT-4.1 mini | OpenAI | $0.04 | $0.08 | **$0.12** |
| GPT-4.1 nano | OpenAI | $0.01 | $0.02 | **$0.03** |
| o3 | OpenAI | $1.00 | $2.00 | **$3.00** |
| o4-mini | OpenAI | $0.11 | $0.22 | **$0.33** |
| 本地模型 | Ollama/LM Studio | $0.00 | $0.00 | **$0.00** |

---

## 优缺点总结

### Claude Code

| 优点 | 缺点 |
|------|------|
| Claude Haiku 3.5 性价比高（$0.80/$4 per 1M） | 无免费使用选项，必须付费 |
| Bedrock/Vertex 企业 API 支持企业协议价 | Claude Opus 4 价格昂贵（$15/$75 per 1M） |
| 内置成本追踪（cost.usage + token.usage 指标） | 无本地模型支持，无法离线使用 |
| MDM maxCostPerSession 企业成本控制 | 闭源，无法审计代码安全性 |
| 4 个 API 后端灵活切换 | 无法自托管，数据必须经过 Anthropic |
| Claude Max/Team 订阅适合个人和团队 | 订阅价格较高（$100-200/月/seat） |
| 上下文窗口 200K，满足大多数场景 | 上下文窗口不及 GPT-4.1 的 1M |
| Anthropic 官方维护，质量有保障 | 社区无法贡献代码和功能 |
| MCP 生态成熟，4 种传输协议 | 插件生态相对封闭 |

### Codex CLI

| 优点 | 缺点 |
|------|------|
| 本地模型完全免费（Ollama/LM Studio） | o3 推理模型价格较高（$10/$40 per 1M） |
| ChatGPT Plus $20/月，入门门槛低 | ChatGPT Plus 使用限额可能不够 |
| GPT-4.1 nano 极低价格（$0.10/$0.40 per 1M） | 无企业级 API 后端（如 Bedrock/Vertex） |
| Apache 2.0 开源，代码完全透明 | 无内置成本追踪功能 |
| 可自托管，数据不出本地 | 无 MDM 策略，企业成本控制弱 |
| 75,000+ Stars，421+ 贡献者，社区活跃 | IDE 集成覆盖面较窄 |
| GPT-4.1 系列 1M 上下文窗口 | 本地模型需要 GPU 资源，硬件要求高 |
| 支持多模型提供商（OpenAI/Ollama/LM Studio） | 开源项目的长期维护依赖社区 |
| 零 API 成本运行本地模型 | 本地模型质量不及云端模型 |
| 完全可定制和可扩展 | 无官方企业支持（仅社区支持） |
