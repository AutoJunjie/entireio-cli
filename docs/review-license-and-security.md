# License & Security Review

## 项目 License

项目使用标准 **MIT License**（Copyright 2026 Entire Inc.），是最宽松的开源协议之一。

## 依赖项 License 风险

项目的 `.allowed-licenses` 允许了以下协议：

```
MIT, BSD-2-Clause, BSD-3-Clause, Apache-2.0, MPL-2.0, CC0-1.0
```

### 需要注意的点

| 风险 | 严重程度 | 说明 |
|------|---------|------|
| MPL-2.0 被允许 | 低 | 目前没有 MPL 依赖，但未来可能引入，带来文件级 copyleft 义务（对 MPL 文件的修改必须以 MPL-2.0 开源） |
| 缺少 NOTICE 文件 | 低 | `go-git`、`cobra` 等使用 Apache-2.0，分发二进制时理论上应保留其 NOTICE 文件 |
| License 检查不够本地化 | 低 | `mise run lint` 不包含 license 检查，开发者可能在本地引入不合规依赖 |

### 主要依赖 License 一览

| 包 | 版本 | License | 用途 |
|---|------|---------|------|
| `posthog-go` | v1.10.0 | MIT | 遥测（可 opt-out） |
| `machineid` | v1.0.1 | MIT | 设备指纹（用于遥测） |
| `gitleaks/v8` | v8.30.0 | MIT | Secret 脱敏 |
| `go-git/v5` | v5.17.0 | Apache 2.0 | Git 操作 |
| `cobra` | v1.10.2 | Apache 2.0 | CLI 框架 |
| `charmbracelet/huh` | v0.8.0 | MIT | 交互式表单 |

**总结：** 没有 GPL/AGPL 传染性风险，所有依赖均为宽松协议。

---

## 隐私和数据收集

### PostHog 遥测

- 收集：命令名、agent 名、CLI 版本、flag 名称（不含值）、OS/架构
- 发送到：`https://eu.i.posthog.com`（EU 区域）
- 通过后台子进程发送，不阻塞 CLI
- GeoIP 已禁用（`DisableGeoIP: true`）

### 设备指纹

- 使用 `machineid.ProtectedID("entire-cli")` 生成硬件派生的匿名 ID
- 作为 PostHog 的 `distinct_id`，用于追踪唯一设备

### Opt-out 方式

1. `entire enable` 时选择不分享
2. 设置环境变量：`ENTIRE_TELEMETRY_OPTOUT=1`

---

## Session 密钥泄露风险分析

### 数据流：两层分支架构

```
用户发给 Agent 的消息（可能含密钥）
        │
        ▼
┌─────────────────────────────────┐
│  Shadow 分支 (entire/<hash>)     │  ← 本地临时，❌ 无脱敏
│  存储原始 transcript              │
└───────────┬─────────────────────┘
            │ 用户 commit 时触发 condensation
            ▼
┌─────────────────────────────────┐
│  Metadata 分支                   │  ← ✅ 经过脱敏
│  (entire/checkpoints/v1)        │
│  gitleaks 规则 + 熵值检测         │
└───────────┬─────────────────────┘
            │ pre-push hook 自动推送
            ▼
        GitHub 远端
```

### 脱敏机制

代码在 `redact/redact.go` 中实现了双层检测：

1. **熵值检测**：Shannon 熵 > 4.5 的字符串被替换为 `REDACTED`（大多数 API key 的熵值远超 5.0）
2. **模式匹配**：gitleaks 180+ 内置规则，覆盖 AWS/GCP/Azure/GitHub/Stripe/Slack 等主流平台

所有内容写入 metadata 分支前都经过脱敏（见 `checkpoint/committed.go`）：

- `transcript`（对话记录）→ `redact.JSONLBytes()`
- `prompts`（用户输入）→ `redact.String()`
- `context`（上下文）→ `redact.Bytes()`
- `summary`（摘要）→ `redact.String()`
- 子 agent 的 transcript → `redact.JSONLBytes()`

### 风险点

#### 风险 1：Shadow 分支本地存有未脱敏的原始数据

脱敏**只在 condensation（写入 metadata 分支）时执行**。Shadow 分支 `entire/<hash>` 上的数据是**未脱敏的原始内容**。

虽然 Entire 不会自动 push shadow 分支，但：

- 手动执行 `git push origin entire/<hash>` → 密钥直接暴露
- 使用 `git push --all` → shadow 分支也会被推上去
- **没有任何技术手段阻止这种误操作**（没有 pre-push hook 拦截 shadow 分支的 push）

#### 风险 2：脱敏是 best-effort，有明确的漏网场景

项目文档（`docs/security-and-privacy.md`）明确承认以下局限：

| 漏网场景 | 说明 |
|---------|------|
| 低熵密码 | 短密码如 `MyPass123` 的熵值 < 4.5，不会被检测 |
| 新型 secret | 不在 gitleaks 规则库中的 secret 格式 |
| 文件名中的密钥 | 文件名不扫描 |
| 跳过的 JSON 字段 | `signature`、`*id`、`*ids` 字段被跳过 |
| `image`/`base64` 对象 | 整个对象跳过不扫描 |

#### 风险 3：对话上下文仍可能暴露敏感信息

即使密钥被替换为 `REDACTED`，对话上下文（服务名、endpoint、用法）仍然可见。

#### 风险 4：git reflog 和 gc

Shadow 分支被删除后，数据仍存在于 `git reflog` 中，直到 `git gc` 清理。本地 `.git` 中可能长期保留未脱敏的原始数据。

### 综合评估

| 场景 | 评价 |
|------|------|
| 常见 API key（AWS、GitHub token 等） | 基本能拦住 |
| 短密码、自定义格式的 secret | **大概率拦不住** |
| Shadow 分支误推 | **完全没有防护** |
| 对话上下文泄露 | **无法防护** |

### 建议

1. **不要在 Agent 对话中直接发送密钥。** 改用环境变量、`.env` 文件等方式让 agent 间接引用密钥。
2. **确保仓库是 private 的。** 这是最简单有效的保护。
3. **永远不要 `git push --all`。** 这会把含有未脱敏数据的 shadow 分支推到远端。
4. **推送前检查分支。** 运行 `git branch -a | grep entire/` 确认只推送 `entire/checkpoints/v1`。
