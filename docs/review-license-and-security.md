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

---

## PII（个人身份信息）保护分析

### 结论：没有任何 PII 保护

项目的脱敏系统（`redact/redact.go`）**只针对 secrets（密钥/令牌），不检测 PII**。

脱敏的两层检测机制均无法识别个人信息：

- **熵值检测**（Shannon entropy > 4.5）：PII 是自然语言或结构化数据，熵值远低于阈值
- **gitleaks 模式匹配**：180+ 规则全部针对 API key/token 格式，不包含 PII 模式

### PII 泄露矩阵

| PII 类型 | 会被脱敏吗 | 原因 |
|---------|-----------|------|
| 姓名（如 "张三"、"John Smith"） | **不会** | 熵值极低，不匹配任何模式 |
| 邮箱（如 `zhangsan@gmail.com`） | **不会** | 熵值不足 4.5，gitleaks 不检测邮箱 |
| 手机号（如 `13800138000`） | **不会** | 纯数字不匹配正则 `[A-Za-z0-9+_=-]{10,}` |
| 身份证号（如 `110101199001011234`） | **不会** | 同上，纯数字被排除 |
| 物理地址 | **不会** | 自然语言，熵值极低 |
| 信用卡号 | **不会** | gitleaks 不检测信用卡格式 |
| IP 地址 | **不会** | 含有 `.` 分隔符，不匹配正则 |

### PII 暴露路径

#### 1. Session Transcript（对话记录）

存储在 `entire/checkpoints/v1` 分支的 `full.jsonl` 中，包含用户和 Agent 之间的**完整对话原文**。如果对话中提及了个人信息，会被原样保留。

#### 2. User Prompts（用户输入）

存储在 `prompt.txt` 中。如果用户在 prompt 中包含姓名、邮箱等信息，不会被过滤。

#### 3. Git Commit 元数据

Shadow 分支和 metadata 分支的 commit 对象中包含 `AuthorName` 和 `AuthorEmail`（来自用户的 git 配置）。这些信息直接写入 git 对象，**完全不经过脱敏**。

相关代码路径：
- `checkpoint/checkpoint.go`：`WriteTemporaryOptions.AuthorName/AuthorEmail`
- `checkpoint/checkpoint.go`：`WriteCommittedOptions.AuthorName/AuthorEmail`
- `strategy/common.go`：`GetGitAuthorFromRepo()` 读取 `user.name` 和 `user.email`

#### 4. 遥测数据

`machineid.ProtectedID("entire-cli")` 生成设备指纹发送到 PostHog。虽然不直接包含姓名邮箱，但配合 IP 地址（即使 `DisableGeoIP: true`，PostHog 服务端仍可记录来源 IP）可能关联到个人身份。

#### 5. 本地日志

`docs/architecture/logging.md` 明确规定不记录 PII，但这个约束**只作用于本地日志**（`.entire/logs/`），不影响 transcript 的存储。Transcript 才是主要的 PII 暴露面。

### 与 GDPR/个人信息保护法的冲突

如果用户在 EU 或中国使用此工具，以下行为可能构成合规风险：

| 行为 | 风险 |
|------|------|
| 对话记录推送到 GitHub | 可能违反数据最小化原则 |
| 设备指纹发送到 PostHog | 需要明确告知和同意（当前告知不够显著） |
| Git author 信息写入 metadata 分支 | 邮箱地址是 PII，推送到远端即暴露 |
| 无法删除已推送的 transcript | 可能违反"被遗忘权"（需要 force push + gc） |

---

## 解决方案

### 方案 1：在 pre-push hook 中拦截 shadow 分支推送（防误操作）

**解决问题：** Shadow 分支含未脱敏原始数据，`git push --all` 会误推。

**实现思路：** 在现有的 `PrePush` hook（`hooks_git_cmd.go:190`）中增加检查逻辑。Git 的 pre-push hook 会通过 stdin 接收即将推送的 refspec 列表，可以检测是否包含 shadow 分支并阻断。

**修改点：**
- `strategy/manual_commit_push.go` — `PrePush()` 方法增加 stdin 解析
- 读取 stdin 中的 refspec，如果包含 `entire/<hash>` 格式（但不是 `entire/checkpoints/v1`），返回错误阻断推送
- 输出警告信息告知用户 shadow 分支不应被推送

**示例逻辑：**
```go
// 在 PrePush 中增加
scanner := bufio.NewScanner(os.Stdin)
for scanner.Scan() {
    fields := strings.Fields(scanner.Text())
    localRef := fields[0]
    if strings.HasPrefix(localRef, "refs/heads/entire/") &&
       localRef != "refs/heads/"+paths.MetadataBranchName {
        return fmt.Errorf("blocked push of shadow branch %s (contains unredacted data)", localRef)
    }
}
```

**复杂度：** 低。改动约 20 行代码。

---

### 方案 2：在 `redact/redact.go` 中增加 PII 检测层

**解决问题：** 当前脱敏系统完全不检测 PII。

**实现思路：** 在现有的双层检测基础上，增加第三层——基于正则的 PII 模式匹配。

**修改点：**
- `redact/redact.go` — `String()` 函数中增加第 3 步 PII 检测

**需要覆盖的 PII 模式：**
```go
var piiPatterns = []*regexp.Regexp{
    // 邮箱
    regexp.MustCompile(`[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}`),
    // 中国手机号
    regexp.MustCompile(`\b1[3-9]\d{9}\b`),
    // 中国身份证号
    regexp.MustCompile(`\b\d{17}[\dXx]\b`),
    // 信用卡号（Luhn 校验前的粗筛）
    regexp.MustCompile(`\b(?:\d[ -]*?){13,19}\b`),
    // IPv4
    regexp.MustCompile(`\b(?:\d{1,3}\.){3}\d{1,3}\b`),
    // 美国 SSN
    regexp.MustCompile(`\b\d{3}-\d{2}-\d{4}\b`),
}
```

**注意事项：**
- 正则检测 PII 误报率高（如 "192.168.1.1" 是 IP 也可能是版本号）
- 需要在 `shouldSkipJSONLField` 中排除更多结构字段，避免误伤
- 姓名无法用正则检测（尤其中文姓名），只能覆盖结构化 PII
- 建议通过 settings 配置开关，默认关闭以免影响现有用户

**复杂度：** 中。改动约 80-120 行代码 + 测试。

---

### 方案 3：用户自定义脱敏规则（settings 配置）

**解决问题：** 不同用户/团队有不同的敏感数据定义，内置规则无法覆盖所有场景。

**实现思路：** 在 `.entire/settings.json` 中支持 `redact_patterns` 配置项，允许用户自定义正则脱敏规则。

**修改点：**
- `settings/settings.go` — `EntireSettings` 增加 `RedactPatterns` 字段
- `redact/redact.go` — `String()` 函数在 gitleaks 检测后，追加用户自定义正则匹配

**配置示例：**
```json
{
  "enabled": true,
  "redact_patterns": [
    {
      "name": "china-phone",
      "pattern": "1[3-9]\\d{9}",
      "description": "中国手机号"
    },
    {
      "name": "china-id-card",
      "pattern": "\\d{17}[\\dXx]",
      "description": "中国身份证号"
    },
    {
      "name": "company-internal-id",
      "pattern": "EMP-\\d{6}",
      "description": "内部员工编号"
    }
  ]
}
```

**注意事项：**
- 需要在 settings 加载时预编译正则，避免每次脱敏都重复编译
- `settings.json` 会被 commit 到仓库，团队共享规则；`settings.local.json` 存放个人规则
- 正则错误需要友好提示，不能静默失败

**复杂度：** 中。改动约 100-150 行代码 + 测试。

---

### 方案 4：Shadow 分支也做脱敏（纵深防御）

**解决问题：** 即使 shadow 分支被误推，数据也已经过脱敏。

**实现思路：** 在 `checkpoint/temporary.go` 的 `WriteTemporary` 中，对 transcript、prompt、context 也执行脱敏。

**修改点：**
- `checkpoint/temporary.go` — `WriteTemporary()` 和 `WriteTemporaryTask()` 中增加 redact 调用

**当前状态：** 从代码看，`temporary.go` 中已经对 subagent transcript 和 incremental data 做了脱敏（`redact.JSONLBytes`），但**主 transcript 写入 shadow 分支时并未脱敏**。这是因为 shadow 分支直接从磁盘读取 agent 的原始 transcript 文件。

**注意事项：**
- 脱敏后 rewind 功能恢复的内容也是脱敏过的，密钥信息不可恢复
- 双重脱敏（shadow + metadata）会略微增加 CPU 开销，但 gitleaks 检测很快
- 这是纵深防御的最后一道防线

**复杂度：** 低。改动约 10-20 行代码。

---

### 方案 5：Git commit 元数据匿名化

**解决问题：** Shadow 和 metadata 分支的 commit 对象暴露真实 `user.name` 和 `user.email`。

**实现思路：** 在创建 shadow/metadata 分支的 commit 时，使用匿名化的 author 信息。

**修改点：**
- `checkpoint/committed.go` — `GetGitAuthorFromRepo()` 返回值用于 commit，可改为返回固定匿名值
- 或在 `WriteCommitted` / `WriteTemporary` 中覆盖 `AuthorName`/`AuthorEmail`

**示例：**
```go
// 用匿名信息替代真实 git author
const anonymousAuthorName = "Entire CLI"
const anonymousAuthorEmail = "noreply@entire.io"
```

**注意事项：**
- 仅影响 entire 自己创建的内部 commit（shadow/metadata 分支），不影响用户工作分支上的 commit
- 如果团队需要追踪谁的 session 产生了哪些 checkpoint，匿名化会丢失这个信息
- 可以做成 settings 开关：`"anonymous_commits": true`

**复杂度：** 低。改动约 10 行代码。

---

### 推荐实施优先级

| 优先级 | 方案 | 理由 |
|-------|------|------|
| P0 | 方案 1：拦截 shadow 分支推送 | 改动最小，防护效果最大，阻止最严重的泄露路径 |
| P0 | 方案 4：Shadow 分支也做脱敏 | 改动小，纵深防御，即使方案 1 被绕过也有保护 |
| P1 | 方案 5：Commit 元数据匿名化 | 改动小，堵住 git author 泄露 PII 的路径 |
| P1 | 方案 2：内置 PII 检测 | 覆盖最常见的 PII 格式，需要权衡误报率 |
| P2 | 方案 3：用户自定义脱敏规则 | 灵活性最高，但需要用户主动配置 |

**建议组合：** 先实施 P0（方案 1 + 4），再做 P1（方案 2 + 5），最后按需做 P2。P0 两个方案合计改动不超过 40 行代码，可以快速落地。

---

## 总结与建议

### 用户侧建议（当前可立即执行）

1. **不要在 Agent 对话中发送任何敏感信息**（密钥、个人信息均包括在内）。改用环境变量、`.env` 文件等间接方式。
2. **确保仓库是 private 的。** 这是最简单有效的保护，公开仓库的 transcript 对全网可见。
3. **永远不要 `git push --all`。** 这会把含有未脱敏数据的 shadow 分支推到远端。
4. **推送前检查分支。** 运行 `git branch -a | grep entire/` 确认只推送 `entire/checkpoints/v1`。
5. **考虑禁用 session push。** 在 `.entire/settings.json` 中设置 `push_sessions: false`，手动控制何时推送 transcript。
6. **如果涉及 PII 合规**，建议完全不推送 `entire/checkpoints/v1` 分支，或自建 PII 检测层。
