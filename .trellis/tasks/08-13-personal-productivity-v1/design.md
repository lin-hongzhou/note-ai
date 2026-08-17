# 个人效率与个人财务管理应用 V1 — 架构设计

> **状态：规划中，技术架构正逐项确认。**  
> 对应需求：[`prd.md`](./prd.md)。本设计只锁定 V1 可行性所必需的架构边界与跨域约束：模块化单体、REST/OpenAPI、离线同步、数据保护、AI/ASR 和可观测性。**精确依赖版本、完整 OpenAPI 字段、数据库 migration SQL、页面视觉/组件设计与编码规范均留待后续独立任务逐步定稿。**任何实现必须先遵守 PRD 中的用户数据、AI、语音和同步不可妥协规则。

## 1. 设计目标、约束与非目标

### 1.1 设计目标

1. **可靠记录优先**：文本记录、离线工作区和同步不可依赖 AI 或语音成功。
2. **双端分工而非复制**：Android 优先快速记录、离线和行动；Web 优先整理、搜索、回顾和数据工作台。
3. **隐私最小化**：第三方只接收一次动作所需数据；音频不形成服务端或跨端存档；日志不保存正文。
4. **可替换边界**：AI 整理和语音转写通过小型服务端端口隔离提供商；V1 的文本整理和语音转写各只实现一个 OpenAI 适配器。
5. **可验证扩展**：先支持邀请制 Alpha 至数百用户，不预建微服务、消息总线、GraphQL、通用工作流或多模型路由。

### 1.2 硬约束

| 约束 | 设计响应 |
| --- | --- |
| Android：Kotlin + XML + MVVM | 单向状态流的 ViewModel；Repository 隔离本地 Room、API 和同步队列；XML Fragment/Activity 负责渲染和导航。 |
| Web：Vue | Vue 3 + TypeScript + Vite；按业务域路由、组合式 store；桌面优先数据工作台。 |
| 后端：Go | 模块化单体、分层 HTTP/应用/领域/持久化架构；REST API。 |
| 多用户和多设备同步 | 服务端权威数据 + Android 本地操作队列 + 变更游标；客户端操作幂等、版本冲突显式化。 |
| 字段加密 + 搜索/统计 | 敏感字段应用层信封加密；搜索使用受限的加密检索令牌并回验；统计在认证后的服务端安全解密计算。 |
| 语音无音频存档 | Android 私有加密临时文件；服务端仅请求内流转；不写对象存储、数据库或日志。 |

### 1.3 非目标

- REST 之外的 GraphQL/RPC、WebSocket 协作同步、CQRS/事件总线、微服务拆分；
- 多 AI / 多 ASR 提供商、自动失败转移、模型成本路由；
- Android 离线语音识别及 Web 语音输入；
- 端到端加密（E2EE）。V1 采用**服务端可解密的字段级信封加密**，以便当前文本的 AI 整理、服务器计算统计和跨端同步；这是与功能需求的明确取舍。

---

## 2. 总体架构

```text
┌──────────────────────────┐       HTTPS / JSON        ┌─────────────────────────────┐
│ Android App              │ ────────────────────────► │ Go API（模块化单体）         │
│ Kotlin + XML + MVVM      │ ◄──────────────────────── │ /api/v1                     │
│ Room 加密本地工作区       │     REST + sync cursor     │ ┌─ Auth / Accounts           │
│ 操作队列 / 临时加密音频   │                            │ ├─ Capture / AI Orchestrator │
└──────────────────────────┘                            │ ├─ Todo / Ledger / Note      │
                                                          │ ├─ Review / Dashboard        │
┌──────────────────────────┐                            │ ├─ Sync / Trash              │
│ Desktop Web User App     │ ────────────────────────► │ └─ Admin / Audit / Ops       │
│ Vue 3 + TypeScript       │                            └─────┬──────────┬───────────┘
└──────────────────────────┘                                  │          │
                                                                │          │ outbound TLS only
                                                     ┌──────────▼──┐  ┌────▼─────────────────────┐
                                                     │ PostgreSQL  │  │ OpenAI adapters           │
                                                     │ source of   │  │ - Structured organizer    │
                                                     │ truth       │  │ - Audio transcription     │
                                                     └─────────────┘  └──────────────────────────┘
                                                                │
                                                        ┌───────▼────────┐
                                                        │ KMS / secret   │
                                                        │ manager + mail │
                                                        └────────────────┘
```

### 2.1 部署单元

V1 包含四个逻辑部署单元：

1. **Go API**：唯一业务写入口，包含 HTTP API、同步、AI/ASR 协调、定时清理和管理 API；部署为同一可水平扩展应用。
2. **PostgreSQL**：唯一业务真相源，保存加密业务字段、结构化元数据、同步版本、回收站状态、审计和受控运维记录。
3. **异步/定时 worker**：与 API 使用同一代码库和同一数据库，通过可恢复任务表/数据库租约执行邮件、回收站清理、账号删除到期处理、诊断快照过期、提醒和指标聚合。V1 不引入 Redis 或消息队列作为业务真相源。
4. **Web 静态站点**：用户端和管理员端可同一 Vue 项目、不同路由与权限壳层；与 API 分离部署。

**不使用对象存储保存录音。** 音频转写采用请求内受限流转；如代理、负载均衡或 APM 默认会记录请求体，部署配置必须明确关闭该行为。

### 2.2 模块边界（Go）

```text
backend/
├─ cmd/api                 # HTTP server / worker entry points
├─ internal/platform        # config, HTTP, auth primitives, crypto, DB, logging, mail, clock
├─ internal/identity        # users, invite, credentials, sessions, account deletion
├─ internal/capture         # capture inbox, AI orchestration, preference rules
├─ internal/todo            # actionable todo, future plan, weekly review decisions
├─ internal/ledger          # categories, entries, summaries, transparent change insights
├─ internal/note            # notes, metadata, note-to-todo relations, encrypted keyword search
├─ internal/sync            # operations, cursors, conflict records
├─ internal/voice           # request-scoped transcription orchestration
├─ internal/dashboard       # today overview and aggregated read models
├─ internal/admin           # invitations, user status, protected metrics/log views
└─ internal/ops             # audit, redaction, quota, scheduled retention jobs
```

每个业务模块都遵循：

```text
HTTP handler → application use case → domain policy → repository / provider port
```

- **handler**：认证身份、请求大小/格式、响应序列化、请求关联 ID；不得包含业务判断。
- **application use case**：事务边界、授权、加解密、幂等、调用领域规则和写入变更日志。
- **domain policy**：纯业务规则（如今日排序、周回顾推荐、账目变化规则）；用固定时钟和 fixture 可单元测试。
- **repository/provider port**：数据库或第三方实现；不得让提供商直接写业务表。

### 2.3 外部服务端口

```go
// 只表达内部需求，不泄漏某个提供商的 SDK 类型。
type TextOrganizer interface {
    Organize(ctx context.Context, request OrganizeRequest) (OrganizeResult, error)
}

type TranscriptionProvider interface {
    Transcribe(ctx context.Context, audio AudioStream, options TranscriptionOptions) (Transcript, error)
}

type Mailer interface { /* verification / reset / notifications */ }
type KeyManager interface { /* wrap / unwrap per-user DEK */ }
```

V1 实现：

- `OpenAITextOrganizer`：针对记账、待办、智能快速记录和用户主动发起的笔记元信息建议，要求返回受限 JSON Schema 的建议结果。
- `OpenAITranscriptionProvider`：Q57 已确认的唯一 V1 ASR 适配器；仅处理 Android 在线语音转写，语言限普通话/英语，服务端按请求流转音频。
- `FakeTextOrganizer`、`FakeTranscriptionProvider`：测试和本地开发使用固定 fixture，保证测试不依赖真实模型输出。

**边界规则：**提供商适配器只能返回“转写文本”或“受限结构化建议”，不得得到数据库连接、用户会话、业务 repository、原始历史正文或管理权限。

---

## 3. 客户端架构

### 3.1 Android：Kotlin + XML + MVVM

```text
feature screen (Fragment / XML)
  → ViewModel: UiState + one-shot UiEffect
  → Use case
  → Repository
       ├─ encrypted Room database (local work copy)
       ├─ Sync engine / operation queue
       ├─ Retrofit/OkHttp API client
       └─ Voice recorder + temporary-file manager
```

#### 最低架构规则

- 每个业务域有独立 feature：`auth`、`capture`、`todo`、`ledger`、`note`、`review`、`dashboard`、`trash`、`settings`、`voice`。
- ViewModel 暴露不可变 `UiState`，用户动作通过明确事件进入；Fragment 不直接访问网络、数据库或加密密钥。
- 本地数据库是 Android 的**工作副本**，不是跨设备真相源；所有本地变更同时写入 `pending_operations`。
- 认证刷新令牌、数据库密钥封装材料和录音临时密钥使用 Android Keystore 保护；退出、账号停用和账号删除受理后清除本地敏感工作副本。
- 后台同步使用受约束的系统任务：仅网络可用、指数退避、可手动重试。同步状态必须可见。
- Android 不保存服务端密码，不把访问令牌写入普通 SharedPreferences，也不在崩溃日志中写入内容字段。

#### Android 本地表（概念）

| 表 | 用途 |
| --- | --- |
| `local_todos` / `local_ledger_entries` / `local_notes` | 最近服务端版本和本地未同步字段，包含 `server_revision`、`dirty`、`deleted`。 |
| `pending_operations` | 创建/更新/删除/解决冲突操作；稳定 UUID 幂等键、实体 ID、基准版本、最小加密操作载荷、重试状态。 |
| `sync_state` | 每个账号/设备的服务端变更游标、上次成功/错误。 |
| `voice_drafts` | 临时音频路径、创建/到期时间、状态和不可恢复删除提示；不保存转写正文。 |
| `local_conflicts` | 服务端版本和本地版本的可显示副本，待用户选择/合并。 |

### 3.2 Web：Vue 3 + TypeScript

```text
route view → feature composable/store → typed API client → Go REST API
```

- 用户端采用桌面优先的左右/多栏工作台：待确认收件箱、搜索筛选、周回顾、财务汇总和设置均可操作，不是只读看板。
- 管理后台和用户端使用独立路由守卫、布局与权限边界；管理员身份不隐式拥有用户正文查看权限。
- Web 不承诺离线编辑。浏览器仅保存短时 UI 状态；会话采用安全 Cookie 或等效的安全会话机制，不把长期凭据写入 `localStorage`。
- 以 API schema 生成或严格维护 TypeScript DTO；前端不得自行猜测服务端枚举、状态机或金额格式。

### 3.3 路由与页面最小集合

| 端 | 最小页面/入口 |
| --- | --- |
| Android | 登录/注册、今日概览、记一笔、加待办、写笔记、智能收件箱、待办、账目、笔记、回收站、设置、语音草稿；桌面小组件配置。 |
| Web 用户端 | 登录、数据工作台、待确认收件箱、待办/以后想做、账目与统计、笔记搜索、每周回顾、回收站、账户与隐私设置。 |
| Web 管理端 | 管理员登录、邀请码、用户状态、AI 指标、删除申请、审计、受控运行日志/AI 错误。 |

---

## 4. REST API 与契约

### 4.1 共同约定

- 基路径：`/api/v1`。
- JSON 请求/响应使用 `camelCase`；时间为 ISO 8601 UTC，用户时区通过账户设置/请求上下文转换展示。
- 所有创建、更新、删除、恢复和同步推送必须带 `Idempotency-Key: <UUID>`；同一用户、端点、键重复提交返回第一次结果，不重复创建数据。
- 更新必须带 `expectedRevision`。版本不一致返回 `409 CONFLICT`，不做静默最后写入胜出。
- 认证后的所有资源由 access session 确定 user scope；请求体中不允许传入 `userId`。
- API schema 以 OpenAPI 文档为唯一客户端契约；实现前先定义 DTO、枚举、错误码和示例。

统一错误响应：

```json
{
  "error": {
    "code": "SYNC_CONFLICT",
    "message": "此内容已在其他设备更新，请选择要保留的版本。",
    "requestId": "req_01...",
    "retryable": false,
    "details": [
      {"field": "expectedRevision", "reason": "stale"}
    ]
  }
}
```

- `message` 是安全、可本地化的用户提示；不得泄露堆栈、SQL、第三方响应正文或账户是否存在等敏感信息。
- `details` 只提供可公开的字段错误或冲突信息。
- 服务器记录 `requestId` 和内部错误详情，但日志同样受脱敏规则约束。

### 4.2 认证、设备与账户

```text
POST   /auth/register                 # invite + email + password
POST   /auth/email-verifications      # verify token
POST   /auth/sessions                 # login
DELETE /auth/sessions/current          # logout current
POST   /auth/password-resets
POST   /auth/password-resets/confirm
GET    /me
PATCH  /me/settings                    # timezone, base currency, notifications, AI toggle/preferences
GET    /me/devices
DELETE /me/devices/{deviceId}
POST   /me/deletion-requests
POST   /me/deletion-requests/{id}/withdraw
```

**单币种取舍：**账户创建时选择一个基础货币（默认 CNY，可选择 USD 等单一 ISO 货币），V1 所有账目必须使用该基础货币。它不是多币种账本；账户已有账目后不能自行改币种，以免隐式换算或破坏统计。

### 4.3 业务资源

```text
GET/POST              /todos
GET/PATCH/DELETE      /todos/{id}
POST                  /todos/{id}/complete
POST                  /todos/{id}/reopen
POST                  /todos/{id}/restore
GET                   /future-plans/recommendations
POST                  /weekly-reviews
GET/PATCH             /weekly-reviews/{id}
POST                  /weekly-reviews/{id}/decisions

GET/POST              /ledger/categories
PATCH/DELETE          /ledger/categories/{id}
GET/POST              /ledger/entries
GET/PATCH/DELETE      /ledger/entries/{id}
POST                  /ledger/entries/{id}/restore
GET                   /ledger/summary?from=&to=&groupBy=day|week|month|category
GET                   /ledger/insights?period=

GET/POST              /notes
GET/PATCH/DELETE      /notes/{id}
POST                  /notes/{id}/restore
GET                   /notes/search?q=&from=&to=&type=&starred=&reviewLater=
POST                  /notes/{id}/linked-todos

GET/POST              /captures
GET/PATCH/DELETE      /captures/{id}
POST                  /captures/{id}/convert
POST                  /ai/organize
POST                  /notes/{id}/ai-metadata-suggestions
```

说明：

- `POST /ai/organize` 只产生短寿命或可保存的**建议草稿**；确认动作由资源创建/更新 API 或 `captures/{id}/convert` 完成。
- 记账、待办和笔记正文资源不接受模型/客户端标记的“已确认”绕过字段；服务端依据实际操作创建正式数据。
- 查询分页采用稳定游标，结果必须限制到认证用户。

### 4.4 语音转写

```text
POST /voice/transcriptions
Content-Type: multipart/form-data
fields: audio, languageHint (zh|en|auto), sourceIntent (note|smart|manual)
```

**服务端处理规则：**

1. 验证已登录、速率、MIME、文件大小、时长上限和语言 hint；请求体日志禁止记录。
2. 使用受限读取器把音频在当前请求中转发给 `TranscriptionProvider`；不得写数据库、对象存储、临时目录或异步队列。
3. 返回 `transcript`、检测语言和不敏感处理元数据；无论成功、失败或客户端中断，都释放内存/流资源。
4. 客户端是临时录音的唯一保存者，负责在确认/取消/到期时删除。

V1 建议把单次录音限制为 **3 分钟或 10 MiB（以先到者为准）**。这是防止长录音、内存滥用和成本失控的产品防护，不是永久能力上限；未来若真实需求存在，可在压测和隐私复核后扩展。

### 4.5 同步、冲突与回收站

```text
POST /sync/push
{
  "deviceId": "...",
  "operations": [
    {
      "operationId": "uuid",
      "entityType": "todo|ledgerEntry|note|capture",
      "entityId": "uuid",
      "kind": "create|update|delete|restore|resolveConflict",
      "baseRevision": 7,
      "payload": {"...": "..."}
    }
  ]
}

GET /sync/pull?cursor=<opaque>&limit=200
POST /conflicts/{id}/resolve
GET  /trash
POST /trash/{entityType}/{id}/restore
DELETE /trash/{entityType}/{id}           # immediate permanent delete
```

- `push` 中每个操作单独返回 accepted / duplicate / conflict / rejected；一次请求内某个失败不应导致已验证、彼此独立的操作丢失。
- 服务器事务中：检查幂等键 → 校验认证与版本 → 写资源/操作记录 → 写变更日志 → 提交。
- `pull` 返回创建、更新、删除（墓碑）和冲突状态的游标增量。客户端先成功处理/落地变更，才推进本地 cursor。
- 删除墓碑须保留至所有合理离线窗口及回收站期结束，防止离线旧设备把已删除记录复活。

### 4.6 运营后台 API

```text
POST/GET/PATCH  /admin/invitations
GET/PATCH       /admin/users
GET/PATCH       /admin/deletion-requests
GET             /admin/ai-metrics
GET             /admin/audit-events
GET             /admin/operation-logs
GET             /admin/ai-diagnostics
```

后台查询的默认 DTO 必须无业务正文和完整 AI 输入/输出。查看 AI 受控诊断快照需要显式权限和独立审计事件。

---

## 5. 关键数据模型

### 5.1 共同记录字段

所有用户拥有的业务表至少有：

```text
id (UUID) | user_id | revision | created_at | updated_at | deleted_at? |
created_by_device_id? | last_operation_id | encryption_key_version
```

- 逻辑删除使用 `deleted_at`；回收站到期或用户立即永久删除后以后台任务物理清理。
- `revision` 是单实体单调递增整数，用于乐观并发控制。
- `last_operation_id` 支持审计与重复请求诊断，不能替代完整幂等记录表。

### 5.2 主要表（概念模型）

| 表 | 核心字段 / 说明 |
| --- | --- |
| `users` | `id`、`email_ciphertext`、`email_lookup_hmac`（唯一查找）、`password_hash`、`status`、`timezone`、`base_currency`、`ai_enabled`、删除申请状态。 |
| `user_key_envelopes` | 每用户数据密钥（DEK）被 KMS 主密钥包裹后的密文、key version、状态；不保存明文 DEK。 |
| `sessions` / `devices` | `user_id`、设备元数据、刷新令牌哈希、到期/撤销时间。 |
| `invitations` | 邀请码哈希、状态、生成/过期/使用时间、使用者；绝不保存可再次展示的明文邀请码。 |
| `todos` | `kind`（action/future）、`title_ciphertext`、`status`、`priority`、`planned_at`、`due_at`、回顾可见性/最近回顾时间、加密说明字段。 |
| `ledger_categories` | `user_id`、内置/自定义标记、`name_ciphertext`、显示排序、停用状态。 |
| `ledger_entries` | `direction`、`amount_minor_ciphertext`、`occurred_at`、`category_id`、`payment_method`、`note_ciphertext`、基础货币快照。 |
| `notes` | `title_ciphertext`、`body_ciphertext`、`note_type`、`starred`、`review_later`、加密标签字段。 |
| `note_todo_links` | `note_id`、`todo_id`、创建来源；两端均按同一用户校验。 |
| `captures` | `raw_text_ciphertext`、输入来源（typed/voice）、候选类型、状态（pending/converted/discarded）、加密建议摘要。 |
| `ai_suggestions` | 关联 capture/资源、加密结构化建议、schema version、状态（pending/accepted/edited/rejected）、无正文的模型/用量元数据。 |
| `preference_rules` | 用户确认结果形成的有限规则；键和值加密或枚举化；可逐条禁用/删除。 |
| `sync_operations` | `user_id`、`operation_id`、endpoint/resource、处理结果、时间；保留期按同步/审计需求配置，不存正文。 |
| `change_log` | 全局递增 change sequence、用户、实体类型/ID/revision/删除状态；供 `sync/pull` 使用。 |
| `conflicts` | 资源 ID、base/server/client revision、两个加密版本快照、待处理/已解决状态；敏感字段不可静默合并。 |
| `trash_items` | 资源类型/ID、删除时间、到期时间、删除来源；实体本身仍在原表逻辑删除。 |
| `weekly_reviews` / `review_decisions` | 回顾窗口、完成/跳过、候选项、用户决策和理由（如有）。 |
| `audit_events` | 操作者、动作、对象类型/ID、结果、关联 ID、时间、少量脱敏上下文；不存内容。 |
| `operation_logs` / `ai_diagnostics` | 严格脱敏运行元数据；诊断快照到期时间为创建后 7 天。 |
| `retention_jobs` | 可恢复任务、租约、状态、下一运行时间；支撑过期清理和通知，不作为业务内容队列。 |

### 5.3 金额、时间与分类约束

- 金额以最小货币单位有符号整数表示（例如分），显示层按账户基础货币格式化；禁止 `float` 或 `double`。
- `direction` 决定收入/支出，金额绝对值必须大于零；结余只在计算时生成，不存为账户余额。
- `occurred_at`、`planned_at`、`due_at` 存 UTC；展示和“今日/本周”判断以用户时区进行。
- 内置分类对同一用户可见且不可破坏历史账目；自定义分类软停用时历史数据保持原关联。

### 5.4 字段加密与可检索性

**加密方案：**

1. 每位用户拥有逻辑 DEK；DEK 使用 KMS/秘密管理系统的 KEK 包裹，表中只保存包裹密文和版本。
2. 每个敏感字段采用随机 nonce 的 AEAD 加密；AAD 绑定 `environment + user_id + entity_type + entity_id + field + key_version`，防止密文被交换到其他用户或字段。
3. 邮箱查找使用独立密钥计算的规范化 HMAC；邮箱正文仍加密保存。
4. 笔记标题/正文关键词搜索使用规范化 token 的 HMAC 检索令牌：先按用户范围匹配 token，再在服务端解密候选记录做精确回验。该方案允许关键词找回，但会泄露相同 token 的等值/频率模式；这是 V1 在“字段加密”和“关键词搜索”之间显式接受的最小泄露面。
5. 账目汇总和收支变化计算在授权请求内解密该用户、该时间范围内的必要金额；不建立保存明文金额的分析仓库。

**密钥与轮换：**

- 本地开发可用明确标识的开发密钥；任何 Alpha/生产环境必须使用受访问控制和轮换能力的 KMS/秘密管理服务。
- 密钥版本写入密文元数据；读取时支持旧版本，后台重加密作业可逐步轮换，不能一次性全表重写。
- 账号永久删除时排队删除/失效用户 DEK 包裹材料；这提供加密擦除层，同时仍按备份策略清理副本。

---

## 6. 关键流程

### 6.1 纯文本手动记录（任何网络状态）

```text
Android 输入文本
  → ViewModel 验证基础字段
  → Room 写入本地实体 + pending operation（同一事务）
  → UI 显示“待同步”
  → 网络可用时 sync push
  → 服务端幂等校验、授权、加密、写业务表 + change log
  → 客户端拉取确认 revision，显示“已同步”
```

- 对账目和待办，用户可直接手动填写并保存，AI 可选。
- 对笔记，用户可直接原样保存；任何网络故障不得阻断此路径。

### 6.2 AI 整理与待确认收件箱

```text
用户主动点“智能整理”或明确入口请求建议
  → 客户端发送当前文本 + 明确意图
  → 服务端检查 AI 开关、个人/全局配额、速率
  → 读取最小必要结构化偏好规则
  → 调用 TextOrganizer，获得 schema 约束建议
  → 服务端 JSON/schema/业务/权限校验
     ├─ 有效：创建加密 suggestion/capture 草稿，返回用户确认界面
     └─ 无效、失败、不确定：创建/保留原文 capture，状态 pending
  → 用户编辑、确认或转为其他类型
  → 正式 use case 写业务记录和 change log
```

- “模型高置信度”不是自动保存授权。
- 笔记默认跳过此流程；只有用户请求笔记元信息建议时才发送当前笔记。
- 偏好规则只在用户确认/修正结果后更新，不能由模型猜测或由完整历史反推。

### 6.3 Android 在线语音到文本

```text
开始录音
  → 生成 voice draft（私有加密临时文件 + 24h 到期）
  → 用户点“转写”且网络可用
  → HTTPS 上传至 /voice/transcriptions
  → 服务端请求内转发 OpenAI transcription
  → 返回 transcript
  → Android 显示可编辑文本
  → 用户确认：进入笔记原文 / 手动表单 / 智能整理
       成功、取消或放弃：立即删除临时音频
     失败：临时音频保留至可重试或到期；也可手动文本草稿
```

状态机：

```text
recording → ready_to_transcribe → transcribing → transcript_ready
                         ↘ failed_retryable → retrying
任何状态 → discarded / expired
transcript_ready → confirmed
```

`discarded`、`confirmed`、`expired` 均立即删除音频文件和本地密钥材料；只保留不含音频/转写正文的状态提示。剩余 12 小时与 1 小时由本地调度器安排提示；没有通知权限时，应用内持续显示。

### 6.4 同步冲突

```text
设备 A 与设备 B 基于 revision 7 修改同一笔记
A 先提交 → server revision 8
B 提交 expectedRevision 7 → 409 + server revision 8 + conflictId
B 保存本地版本与服务端版本 → 待处理冲突
用户选保留某版本或手动合并 → resolveConflict(expected revisions)
服务端创建 revision 9 → change log → 两设备拉取一致结果
```

- 对不同非敏感字段、且可证明无重叠的并发修改，未来可以安全合并；V1 优先统一保守策略，尤其正文、金额、时间必须显式决策。
- 冲突版本也是用户内容，按字段加密和回收/清理规则保护。

### 6.5 删除、恢复和账号删除

```text
资源删除 → 原表 deleted_at + trash item(expires_at = +30d) + change log
恢复 → 清除 deleted_at / trash item + 新 revision + change log
立即永久删除 / 到期任务 → 清理资源、搜索令牌、关联草稿/建议/冲突和内容密钥材料

账号删除申请 → pending review
人工受理 → disabled + revoke sessions + block sync/AI
30 天可撤销期 → withdraw 恢复（若尚未清理）
到期 → 删除可识别账户和业务数据、失效用户 DEK、按公开备份周期清理残留副本
```

---

## 7. 错误处理、可观测性与运营

### 7.1 错误分类与客户端行为

| 类别 | HTTP / 代码示例 | 客户端行为 |
| --- | --- | --- |
| 输入校验 | `400 VALIDATION_ERROR` | 标出可修复字段，不丢草稿。 |
| 未认证/会话失效 | `401 UNAUTHENTICATED` | 尝试一次安全刷新；失败后登录，保留本地未同步操作。 |
| 无权/账户停用 | `403 FORBIDDEN` / `ACCOUNT_DISABLED` | 停止同步与 AI；说明原因并清理受控会话数据。 |
| 找不到/已永久删除 | `404 NOT_FOUND` / `410 GONE` | 移除本地陈旧副本，提示内容不可恢复。 |
| 并发冲突 | `409 SYNC_CONFLICT` | 保留双方版本，进入待处理冲突。 |
| 无法处理的 AI 结果 | `422 AI_RESULT_INVALID` | 原文进入待确认或回到手动表单。 |
| AI/语音配额 | `429 AI_QUOTA_EXCEEDED` | 显示何时恢复；手动记录持续可用。 |
| 外部依赖/网络 | `503 PROVIDER_UNAVAILABLE` / timeout | 可重试；语音保留 Android 临时音频，文本保留草稿。 |
| 内部异常 | `500 INTERNAL_ERROR` | 只显示 request ID 和安全提示；记录脱敏故障。 |

### 7.2 日志、审计与指标

- 每个入站请求生成/接受 `requestId`；操作链路在 Android、Web、API、worker、AI/ASR 调用中传播。
- 允许日志字段：时间、环境、模块/API、关联 ID、匿名用户 ID、错误码、堆栈、模型、耗时、尺寸、token、错误类型、重试次数。
- 禁止日志字段：密码、验证码、访问/刷新令牌、笔记正文、待办正文、账目备注、完整音频、完整请求/响应、完整 AI 输入/输出。
- `ai_diagnostics` 仅记录经过专用脱敏函数和长度限制的快照，默认 7 天过期；笔记正文禁止纳入。管理员查看每次写入 `audit_events`。
- 核心指标：认证成功/失败、同步成功/冲突/重试、未同步操作积压、AI 调用/配额/延迟/失败、建议确认/修改/拒绝、语音成功/失败/过期、回收站清理、账号删除任务、周回顾完成率。

### 7.3 配额与熔断

- AI 请求先执行个人日上限和平台总上限检查，再调用提供商；计数必须在服务端原子结算。
- 配额接近阈值、外部提供商错误激增或成本风险达到阈值时发管理员告警。
- 管理员有 AI 紧急关闭开关；关闭后 `/ai/organize` 和笔记 AI 建议立即返回明确降级响应，语音转写开关独立管理。
- 所有文本手动创建 API 与同步 API 不受 AI 开关影响。

---

## 8. 安全与部署前检查

### 8.1 最小权限

- API、worker、数据库迁移、管理员 Web 和运维查询使用不同服务账户/角色。
- PostgreSQL 用户数据表含 `user_id NOT NULL`，repository 必须带 actor scope；推荐使用数据库行级策略作为第二道防线，并在每个事务以 `SET LOCAL app.user_id` 设置当前用户范围。
- 管理员 API 不默认读取正文表字段；需要受控诊断查询的角色和操作独立授权。
- KMS 解密权限仅授予 API/worker 运行身份；开发、测试、生产密钥、数据库和日志项目严格分离。

### 8.2 上线前必须具体化的运维值

这些不是产品范围问题，但在 Alpha 上线前必须写入部署 runbook 和用户可见隐私说明：

1. OpenAI 文本整理模型与转写模型、账户的数据使用/保留控制和故障降级开关；
2. 邮件服务提供商、发送域名、验证码/邮件 token 到期时间；
3. KMS/秘密管理服务、密钥轮换职责和紧急失效步骤；
4. 备份频率、保留周期、恢复演练方法，以及账号永久删除后备份残留的最大清理周期；
5. AI 单账号每日上限、平台总上限、告警阈值、最大上传大小与录音时长；
6. 日志/审计/诊断数据的实际保留期限和访问授权名单；
7. HTTPS、数据库、邮件、OpenAI 网络可达性和域名/证书配置。

### 8.3 回滚策略

| 变更类型 | 回滚方式 |
| --- | --- |
| API/客户端功能 | Feature flag 关闭 AI、语音、收支变化提示或小组件详情；已有手动路径继续工作。 |
| 数据库迁移 | 使用 expand → backfill → switch → contract；发布前准备向后兼容读写，禁止不可逆破坏性迁移与应用版本同发。 |
| 模型/提示词调整 | schema version 化；新版本先灰度，失败即停用并回到手动/待确认收件箱。 |
| 同步协议 | 新旧客户端并行兼容至最低支持版本；cursor 和 operation payload 必须版本化。 |
| 密钥轮换 | 保留旧 key version 解密能力直至重加密完成；不能因轮换导致用户内容不可读。 |

---

## 9. 关键取舍记录

| 决策 | 选择 | 原因 / 代价 |
| --- | --- | --- |
| 服务形态 | **已确认 TQ1：Go 模块化单体 + PostgreSQL** | Alpha 速度与可靠性优先；按认证、同步、收件箱、待办、记账、笔记、回顾、AI/语音与后台领域组织，未来按真实瓶颈再拆。 |
| API | **已确认 TQ2：REST + OpenAPI** | Android/Web/后台共用版本化 `/api/v1` 契约、调试和幂等同步清晰；V1 不提前引入 GraphQL、RPC 或 gRPC。 |
| 数据访问 | **已确认 TQ3：`pgx` + SQL migrations + `sqlc`** | SQL 与事务边界可审查，类型映射在生成阶段校验；核心业务不以 ORM 为主。 |
| 同步 | 增量拉取 + 幂等操作 + 显式冲突 | 离线文本安全优先；代价是 V1 不做实时协作。 |
| AI | 单一 OpenAI 适配器 + provider port | 未来可替换但不承担多模型运营复杂度。 |
| ASR | 单一 OpenAI 转写适配器（`TranscriptionProvider`）+ 请求内流转 | 已确认采用 OpenAI，快速上线且无服务端音频存档；代价是 V1 不能离线转写或自动故障切换。 |
| 数据加密 | 服务端可解密的应用层信封加密 | 满足字段加密、统计和 AI；代价是并非 E2EE。 |
| 搜索 | HMAC token 检索 + 解密回验 | 支持关键词找回而不保存明文全文索引；代价是泄露 token 等值/频率模式。 |
| 财务计算 | 只计算已记录账目、单账户基础货币 | 保持可信和简单；代价是不做余额、多币种、自动流水。 |
| 音频留存 | Android 私有加密临时文件最长 24 小时 | 支持失败重试而不建立音频档案；代价是到期不可恢复。 |

## 10. 本 task 与后续任务的职责边界

本 task 的产出是：V1 产品范围、验收边界、跨端职责、不可妥协的安全/隐私规则、影响可行性的高层技术选择，以及 T01–T11 的依赖顺序。它不是一次性完成全部工程设计的任务。

后续在启动实施任务前或实施任务内，按“刚好足够、可验证”的粒度继续完成：

1. **工程基线任务**：锁定精确依赖版本、仓库结构、格式化/Lint/测试/CI、密钥注入、编码与提交规范，并把长期规则沉淀进 `.trellis/spec/`；
2. **各功能实施任务**：按 T01、T02…分别细化数据模型、migration、OpenAPI 请求/响应/错误字段、权限、幂等、同步行为、测试夹具和回滚方案；
3. **UX/UI 设计任务**：在产品已确认的 Android“快速记录/行动”和 Web“整理/数据工作台”分工之上，制作信息架构、用户流程、线框图、设计系统和具体页面交互；
4. **运维与发布任务**：确定云/地域、部署拓扑、CI/CD、监控告警、备份恢复、成本阈值和 runbook。

因此，本文件中已有的 API 路径、数据模型和界面职责仅用于表达全局边界与依赖；每项业务资源的最终字段合同和页面细节不在本 task 中冻结。

## 11. 实施前评审结论

本设计已为 PRD 中全部 V1 范围给出可实施的最小结构、API、同步、语音、数据保护和运维边界。下一步应按 [`implement.md`](./implement.md) 拆分并安排实现；在用户明确批准三个规划工件前，任务不得从 `planning` 进入 `in_progress`。
