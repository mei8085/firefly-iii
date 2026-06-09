# Firefly III Webhook 投递机制深度解析

## 一、概述

Firefly III 的 Webhook 系统采用 **事件驱动 + 异步队列 + 定时重试** 的三层架构，确保业务事件能够可靠地投递到外部系统。整个系统由以下核心组件协作完成：

- **事件触发层**：监听模型变更事件，生成 Webhook 消息
- **消息生成层**：根据触发器类型和响应格式生成消息内容
- **投递队列层**：通过 Laravel Queue 异步发送，结合 Cronjob 实现重试
- **签名安全层**：使用 HMAC-SHA3-256 对请求进行签名校验

---

## 二、业务事件与触发器

### 2.1 触发器类型（WebhookTrigger）

Webhook 触发器定义在 [WebhookTrigger.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Enums/WebhookTrigger.php) 中，共 8 种触发器：

| 枚举值 | 数值 | 触发场景 |
|--------|------|----------|
| `ANY` | 50 | 任意事件触发 |
| `STORE_TRANSACTION` | 100 | 创建交易时触发 |
| `UPDATE_TRANSACTION` | 110 | 更新交易时触发 |
| `DESTROY_TRANSACTION` | 120 | 删除交易时触发 |
| `STORE_BUDGET` | 200 | 创建预算时触发 |
| `UPDATE_BUDGET` | 210 | 更新预算时触发 |
| `DESTROY_BUDGET` | 220 | 删除预算时触发 |
| `STORE_UPDATE_BUDGET_LIMIT` | 230 | 创建/更新预算限额时触发 |

### 2.2 事件监听器

业务事件通过三个监听器捕获并触发 Webhook 消息生成：

#### 交易组事件监听器
- [ProcessesNewTransactionGroup.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesNewTransactionGroup.php)
  - 监听 `CreatedSingleTransactionGroup`、`UserRequestedBatchProcessing` 事件
  - 触发 `STORE_TRANSACTION` 触发器
  
- [ProcessesUpdatedTransactionGroup.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesUpdatedTransactionGroup.php)
  - 监听 `UpdatedSingleTransactionGroup` 事件
  - 触发 `UPDATE_TRANSACTION` 触发器

- [ProcessesDestroyedTransactionGroup.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesDestroyedTransactionGroup.php)
  - 监听 `DestroyedSingleTransactionGroup` 事件
  - 触发 `DESTROY_TRANSACTION` 触发器

#### 预算事件监听器
- [ProcessesBudgets.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Listeners/Model/Budget/ProcessesBudgets.php)
  - 监听 `CreatedBudget`、`UpdatedBudget`、`DestroyingBudget` 事件
  - 分别触发 `STORE_BUDGET`、`UPDATE_BUDGET`、`DESTROY_BUDGET` 触发器

#### 预算限额事件监听器
- [ProcessesBudgetLimits.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Listeners/Model/BudgetLimit/ProcessesBudgetLimits.php)
  - 监听 `CreatedBudgetLimit`、`UpdatedBudgetLimit`、`DestroyedBudgetLimit` 事件
  - 统一触发 `STORE_UPDATE_BUDGET_LIMIT` 触发器

### 2.3 触发条件控制

每个事件都带有 `flags` 参数，其中 `fireWebhooks` 标志位控制是否触发 Webhook：

```php
// ProcessesNewTransactionGroup.php 中的判断逻辑
if ($event->flags->fireWebhooks) {
    $this->createWebhookMessages($event->objects->transactionGroups, WebhookTrigger::STORE_TRANSACTION);
}
```

此外，全局功能开关也会影响 Webhook 是否启用：
- `config('firefly.feature_flags.webhooks')`
- `FireflyConfig::get('allow_webhooks', ...)`

---

## 三、消息生成机制

### 3.1 消息生成器（StandardMessageGenerator）

[StandardMessageGenerator.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Generator/Webhook/StandardMessageGenerator.php) 是消息生成的核心组件，实现了 `MessageGeneratorInterface` 接口。

**工作流程：**

1. **设置上下文**：通过 `setUser()`、`setTrigger()`、`setObjects()` 设置生成上下文
2. **筛选 Webhook**：调用 `getWebhooks()` 根据触发器筛选出匹配的活跃 Webhook
3. **生成消息**：遍历每个 Webhook 和每个对象，调用 `generateMessage()` 生成消息
4. **持久化消息**：通过 `WebhookMessageFactory` 创建 `WebhookMessage` 记录

### 3.2 Webhook 匹配规则

在 `getWebhooks()` 方法中，通过以下 SQL 逻辑匹配 Webhook：

```sql
WHERE active = true 
  AND (webhook_triggers.title = :trigger_name OR webhook_triggers.title = 'ANY')
```

即：触发器精确匹配 **或** 设置了 `ANY` 触发器的 Webhook 都会被选中。

### 3.3 响应内容类型（WebhookResponse）

响应类型定义在 [WebhookResponse.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Enums/WebhookResponse.php) 中：

| 枚举值 | 数值 | 说明 |
|--------|------|------|
| `TRANSACTIONS` | 200 | 返回交易详情 |
| `ACCOUNTS` | 210 | 返回关联账户信息 |
| `BUDGET` | 230 | 返回预算详情 |
| `RELEVANT` | 240 | 自动返回相关数据 |
| `NONE` | 220 | 不返回内容 |

#### RELEVANT 响应的自动映射

当 Webhook 设置为 `RELEVANT` 响应时，系统会根据触发事件的对象类型自动选择响应内容，映射规则在 `getRelevantResponse()` 方法中：

| 触发对象类型 | 实际响应类型 |
|-------------|-------------|
| `TransactionGroup` | `TRANSACTIONS` |
| `Budget` / `BudgetLimit` | `BUDGET` |

#### 触发器与响应的约束关系

[config/webhooks.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/config/webhooks.php) 中定义了触发器与响应之间的约束规则：

**禁止的响应组合（forbidden_responses）：**
- `ANY` 触发器不能使用 `BUDGET`、`TRANSACTIONS`、`ACCOUNTS` 响应
- 交易类触发器不能使用 `BUDGET` 响应
- 预算类触发器不能使用 `TRANSACTIONS`、`ACCOUNTS` 响应

**关联响应（force_relevant_response）：**
定义了哪些触发器会影响其他类型的数据，用于 `RELEVANT` 响应的上下文判断。

### 3.4 消息工厂（WebhookMessageFactory）

[WebhookMessageFactory.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Factory/WebhookMessageFactory.php) 负责创建 `WebhookMessage` 模型实例：

```php
$webhookMessage = new WebhookMessage();
$webhookMessage->webhook()->associate($webhook);
$webhookMessage->sent    = false;  // 初始状态：未发送
$webhookMessage->errored = false;  // 初始状态：无错误
$webhookMessage->uuid    = $data['uuid'];  // 唯一标识
$webhookMessage->message = $data;  // 消息内容（JSON）
```

### 3.5 消息体结构

生成的 Webhook 消息包含以下字段：

```json
{
  "uuid": "唯一标识符（UUID v4）",
  "user_id": 0,
  "user_group_id": 0,
  "trigger": "STORE_TRANSACTION",
  "response": "TRANSACTIONS",
  "url": "https://example.com/webhook",
  "version": "v0",
  "content": { ... }
}
```

---

## 四、投递队列与重试机制

### 4.1 整体架构

Webhook 投递采用 **定时任务触发 + 队列异步发送** 的混合模式：

```
业务事件发生
    ↓
生成 WebhookMessage（sent=false, errored=false）
    ↓
WebhookCronjob（每10分钟） ──→ 触发 WebhookMessagesRequestSending 事件
    ↓
SendsWebhookMessages 监听器
    ↓
筛选待发送消息（最多5条，尝试次数≤2）
    ↓
SendWebhookMessage Job（异步队列）
    ↓
StandardWebhookSender 发送
    ↓
成功：sent=true
失败：sent=false, errored=true，记录 WebhookAttempt
```

### 4.2 定时任务触发（WebhookCronjob）

[WebhookCronjob.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Support/Cronjobs/WebhookCronjob.php) 负责周期性地触发 Webhook 发送：

**执行频率：** 每 10 分钟一次（通过 `last_webhook_job` 配置记录上次执行时间，间隔 600 秒）

**核心逻辑：**
```php
$diff = now()->getTimestamp() - $lastTime;
if ($diff > 600) {
    $this->fireWebhookMessages();  // 触发发送
}
```

**触发方式：** 通过 `event(new WebhookMessagesRequestSending())` 发布事件

### 4.3 消息分发（SendsWebhookMessages）

[SendsWebhookMessages.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Listeners/Model/Webhook/SendsWebhookMessages.php) 监听器处理 `WebhookMessagesRequestSending` 事件：

**消息筛选规则：**
1. `sent = false` — 未发送的消息
2. `webhookAttempts()->count() <= 2` — 尝试次数不超过 3 次（0、1、2 共 3 次）
3. `splice(0, 5)` — 每次最多处理 5 条消息

**发送流程：**
```php
foreach ($messages as $message) {
    $message->sent = true;       // 先标记为已发送（防重复）
    $message->save();
    SendWebhookMessage::dispatch($message)->afterResponse();  // 分发到队列
}
```

> **注意**：这里先将 `sent` 设为 `true` 是为了防止重复分发。如果发送失败，会在 Sender 中将 `sent` 重新设回 `false` 并标记 `errored=true`。

**清理机制：**
```php
WebhookMessage::where('sent', true)
    ->where('created_at', '<', now()->subDays(14))
    ->delete();
```
已发送超过 14 天的消息会被自动清理。

### 4.4 队列任务（SendWebhookMessage）

[SendWebhookMessage.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Jobs/SendWebhookMessage.php) 是 Laravel Queue Job，实现了 `ShouldQueue` 接口：

```php
public function handle(): void
{
    $sender = app(WebhookSenderInterface::class);
    $sender->setMessage($this->message);
    $sender->send();
}
```

该 Job 本身不包含复杂逻辑，只是委托给 `WebhookSenderInterface` 完成实际发送。

### 4.5 重试策略

**重试次数：** 最多 3 次（首次 + 2 次重试）

**重试间隔：** 由 Cronjob 的执行频率决定，约每 10 分钟重试一次

**重试条件：**
- `sent = false` （发送失败后被重置）
- `webhookAttempts 记录数 ≤ 2`

**失败记录：** 每次发送失败都会创建一条 [WebhookAttempt](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Models/WebhookAttempt.php) 记录，包含：
- `status_code`：HTTP 状态码（0 表示连接错误）
- `logs`：错误日志和堆栈信息

---

## 五、签名校验机制

### 5.1 签名生成器（Sha3SignatureGenerator）

[Sha3SignatureGenerator.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Helpers/Webhook/Sha3SignatureGenerator.php) 实现了 `SignatureGeneratorInterface` 接口，使用 HMAC-SHA3-256 算法生成签名。

### 5.2 签名生成过程

**签名 payload 构造：**

```
payload = timestamp + "." + json_body
```

其中：
- `timestamp`：Unix 时间戳（秒级）
- `json_body`：消息内容的 JSON 字符串

**签名计算：**

```php
$signature = hash_hmac('sha3-256', $payload, $webhook->secret);
```

**签名头格式：**

```
Signature: t=1717986918,v1=abc123def456...
```

格式说明：
- `t=` 前缀：时间戳
- `v1=` 前缀：v1 版本的签名值
- 多组签名之间用逗号分隔

### 5.3 请求头信息

[StandardWebhookSender.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Services/Webhook/StandardWebhookSender.php) 中定义的请求头：

| Header | 值 | 说明 |
|--------|----|------|
| `Content-Type` | `application/json` | 请求体类型 |
| `Accept` | `application/json` | 期望的响应类型 |
| `Signature` | `t=...,v1=...` | 签名 |
| `User-Agent` | `FireflyIII/{version}` | 客户端标识 |
| `connect_timeout` | `3.14` | 连接超时（秒） |
| `timeout` | `10` | 请求总超时（秒） |

### 5.4 验签方法（接收方）

接收方验证签名的步骤：

1. 从 `Signature` 头中解析出 `t`（时间戳）和 `v1`（签名值）
2. 将时间戳与请求体拼接：`payload = t + "." + body`
3. 使用相同的 secret 和 HMAC-SHA3-256 计算签名
4. 比较计算出的签名与 `v1` 值是否一致
5. 可选：校验时间戳防止重放攻击（如 5 分钟内有效）

---

## 六、发送执行流程（StandardWebhookSender）

### 6.1 发送流程详解

[StandardWebhookSender.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Services/Webhook/StandardWebhookSender.php) 的 `send()` 方法是实际执行发送的核心。

**完整流程：**

```
1. 标记消息为 sent=true（预标记）
2. 验证 Webhook URL 合法性（IsValidWebhookUrl 规则）
3. 生成签名（可能失败：FireflyException）
   ├─ 失败：记录 WebhookAttempt，sent=false, errored=true，返回
4. 序列化消息体为 JSON（可能失败：JsonException）
   ├─ 失败：记录 WebhookAttempt，sent=false, errored=true，返回
5. 发送 HTTP POST 请求（GuzzleHttp Client）
   ├─ 成功：sent=true，记录日志
   └─ 失败（ConnectException/RequestException）：
        ├─ 记录 WebhookAttempt（status_code + logs）
        ├─ sent=false, errored=true
        └─ 返回
```

### 6.2 异常处理

发送过程中可能遇到三类异常：

| 异常类型 | 触发场景 | 处理方式 |
|---------|---------|---------|
| `FireflyException` | 签名生成失败（如 Webhook 被删除） | 记录尝试，标记错误，返回 |
| `JsonException` | 消息内容 JSON 序列化失败 | 记录尝试，标记错误，返回 |
| `ConnectException` / `RequestException` | 网络连接错误或 HTTP 错误响应 | 记录状态码和响应体，标记错误 |

### 6.3 URL 安全校验

使用 `IsValidWebhookUrl` 规则对 Webhook URL 进行验证，防止 SSRF 等安全问题。

---

## 七、数据模型与表结构

### 7.1 核心模型关系

```
Webhook (1) ────→ (N) WebhookMessage (1) ────→ (N) WebhookAttempt
    │
    ├── (N) WebhookTrigger（多对多）
    ├── (N) WebhookResponse（多对多）
    └── (N) WebhookDelivery（多对多）
```

### 7.2 Webhook 模型

[Webhook.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Models/Webhook.php) 主要字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `active` | boolean | 是否启用 |
| `trigger` | integer | 触发器类型（枚举值） |
| `response` | integer | 响应类型（枚举值） |
| `delivery` | integer | 投递方式（JSON=300） |
| `url` | string | Webhook 地址 |
| `secret` | string | 签名密钥 |
| `title` | string | 标题 |
| `user_id` | integer | 所属用户 |

### 7.3 WebhookMessage 模型

[WebhookMessage.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Models/WebhookMessage.php) 主要字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `webhook_id` | integer | 关联的 Webhook |
| `sent` | boolean 是否已发送 |
| `errored` | boolean | 是否出错 |
| `uuid` | string | 消息唯一标识 |
| `message` | json | 消息内容 |
| `logs` | json | 日志信息 |

### 7.4 WebhookAttempt 模型

[WebhookAttempt.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Models/WebhookAttempt.php) 主要字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `webhook_message_id` | integer | 关联的消息 |
| `status_code` | integer | HTTP 状态码 |
| `logs` | text | 日志/错误信息 |

---

## 八、服务容器绑定

所有 Webhook 相关服务在 [FireflyServiceProvider.php](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Providers/FireflyServiceProvider.php) 中注册：

```php
$this->app->bind(MessageGeneratorInterface::class, StandardMessageGenerator::class);
$this->app->bind(SignatureGeneratorInterface::class, Sha3SignatureGenerator::class);
$this->app->bind(WebhookSenderInterface::class, StandardWebhookSender::class);
```

这种基于接口的设计使得各组件可以独立替换，便于扩展和测试。

---

## 九、关键设计决策分析

### 9.1 为什么用 Cronjob 而不是直接队列？

- **削峰填谷**：大量交易同时创建时，避免瞬间产生大量队列任务
- **重试友好**：天然支持定时重试，无需额外的重试队列配置
- **可控性强**：每次只处理 5 条，对外部系统压力小

### 9.2 为什么先标记 sent=true 再发送？

这是一种 **乐观锁** 策略：
- 防止 Cronjob 重复分发同一条消息
- 如果发送失败，再将 `sent` 设回 `false`
- 代价：极端情况下（进程崩溃）可能出现消息标记为已发送但实际未发送的情况

### 9.3 为什么用 HMAC-SHA3-256 而不是 SHA256？

SHA-3（Keccak）是最新的哈希算法标准，相比 SHA-2 有更好的安全性和抗量子计算攻击能力。

### 9.4 为什么重试次数限制为 3 次？

- 平衡投递可靠性与资源消耗
- 配合约 10 分钟的重试间隔，总重试窗口约 20 分钟
- 持续失败的 Webhook 可能意味着接收端已不可用，继续重试意义不大

---

## 十、完整时序图

```
用户操作
   │
   ▼
业务模型变更（TransactionGroup/Budget 等）
   │
   ▼
触发模型事件（CreatedSingleTransactionGroup 等）
   │
   ▼
事件监听器（ProcessesNewTransactionGroup 等）
   │  fireWebhooks = true?
   ▼
StandardMessageGenerator
   │  1. 筛选匹配的 Webhook
   │  2. 生成消息内容
   │  3. WebhookMessageFactory 创建记录
   ▼
webhook_messages 表（sent=false, errored=false）
   │
   ▼
WebhookCronjob（每10分钟）
   │
   ▼
WebhookMessagesRequestSending 事件
   │
   ▼
SendsWebhookMessages 监听器
   │  筛选：sent=false 且 尝试次数≤2
   │  每次最多 5 条
   ▼
SendWebhookMessage Job（Laravel Queue）
   │
   ▼
StandardWebhookSender
   │  1. 验证 URL
   │  2. Sha3SignatureGenerator 生成签名
   │  3. GuzzleHttp 发送 POST 请求
   ▼
成功？───────┬────── 是 ──────► sent=true
   │        │
   否       │
   │        │
   ▼        │
记录 WebhookAttempt
sent=false, errored=true
   │
   └── 下次 Cronjob 重试（最多 3 次）
```
