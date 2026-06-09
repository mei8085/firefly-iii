# Firefly III Webhook 投递机制深度解析

## 一、概述

Firefly III 的 Webhook 系统采用 **事件驱动 + 多层触发 + 异步队列 + 定时重试** 的架构，确保业务事件能够可靠地投递到外部系统。整个系统由以下核心组件协作完成：

- **事件触发层**：监听模型变更事件，生成 Webhook 消息
- **消息生成层**：根据触发器类型和响应格式生成消息内容
- **调度入口层**：四种触发方式共同保障消息投递的实时性与可靠性
- **投递队列层**：通过 Laravel Queue 异步发送，结合 Cronjob 实现重试
- **签名安全层**：使用 HMAC-SHA3-256 对请求进行签名

---

## 二、业务事件与触发条件

### 2.1 触发器类型（WebhookTrigger）

Webhook 触发器枚举共定义了 8 种触发场景：

| 枚举值 | 数值 | 触发场景 |
|--------|------|----------|
| `ANY` | 50 | 任意事件触发（通配） |
| `STORE_TRANSACTION` | 100 | 创建交易时触发 |
| `UPDATE_TRANSACTION` | 110 | 更新交易时触发 |
| `DESTROY_TRANSACTION` | 120 | 删除交易时触发 |
| `STORE_BUDGET` | 200 | 创建预算时触发 |
| `UPDATE_BUDGET` | 210 | 更新预算时触发 |
| `DESTROY_BUDGET` | 220 | 删除预算时触发 |
| `STORE_UPDATE_BUDGET_LIMIT` | 230 | 创建/更新/删除预算限额时触发 |

### 2.2 交易类事件的触发条件

交易类事件（创建、更新、删除交易）由三个监听器分别处理：

- 创建交易：监听 `CreatedSingleTransactionGroup` 和 `UserRequestedBatchProcessing` 事件
- 更新交易：监听 `UpdatedSingleTransactionGroup` 事件
- 删除交易：监听 `DestroyedSingleTransactionGroup` 事件

**触发控制方式：**

交易事件通过 `TransactionGroupEventFlags` 对象的 `fireWebhooks` 属性控制是否触发 Webhook，该属性默认为 `true`。

```php
// 判断逻辑
if ($event->flags->fireWebhooks) {
    $this->createWebhookMessages($event->objects->transactionGroups, WebhookTrigger::STORE_TRANSACTION);
}
```

**批量提交的特殊处理：**

对于批量提交的交易（`batchSubmission = true`），若系统启用了批量处理配置（`enable_batch_processing`），则单个交易创建事件不会触发 Webhook，而是在批量处理完成后统一触发。

### 2.3 预算类事件的触发条件

预算类事件由 `ProcessesBudgets` 监听器统一处理，监听以下三个事件：

- `CreatedBudget`（预算创建）→ 触发 `STORE_BUDGET`
- `UpdatedBudget`（预算更新）→ 触发 `UPDATE_BUDGET`
- `DestroyingBudget`（预算删除中）→ 触发 `DESTROY_BUDGET`

**触发控制方式：**

> **注意**：预算类监听器**不检查**事件的 `createWebhookMessages` 参数，只要事件触发就会生成 Webhook 消息。

虽然 `CreatedBudget` 和 `UpdatedBudget` 事件类都定义了 `createWebhookMessages` 布尔参数，但监听器的 `handle()` 方法并未使用该参数做条件判断，而是直接调用消息生成器。`DestroyingBudget` 事件则根本没有 `createWebhookMessages` 参数。

### 2.4 预算限额类事件的触发条件

预算限额类事件由 `ProcessesBudgetLimits` 监听器统一处理，监听以下三个事件：

- `CreatedBudgetLimit`（预算限额创建）
- `UpdatedBudgetLimit`（预算限额更新）
- `DestroyedBudgetLimit`（预算限额删除）

三个事件统一触发 `STORE_UPDATE_BUDGET_LIMIT` 触发器。

**触发控制方式：**

预算限额事件通过事件对象的 `createWebhookMessages` 属性控制是否触发 Webhook：

```php
// 判断逻辑
if ($event->createWebhookMessages) {
    $this->createWebhookMessages($event->user, $event->budget, WebhookTrigger::STORE_UPDATE_BUDGET_LIMIT);
}
```

### 2.5 全局开关

除了各事件自身的触发标志外，还有两个全局开关控制 Webhook 功能是否启用：

- `config('firefly.feature_flags.webhooks')` — 功能特性开关
- `FireflyConfig::get('allow_webhooks', ...)` — 系统配置开关

任意一个为 `false` 时，Webhook 投递都不会执行。

### 2.6 消息生成与投递触发的两步流程

理解 Webhook 投递的关键是区分两个独立的步骤：

1. **消息生成**：由模型事件监听器（如 `ProcessesBudgets`、`ProcessesBudgetLimits`）响应业务事件，调用消息生成器创建 `WebhookMessage` 记录
2. **投递触发**：发布 `WebhookMessagesRequestSending` 事件，由 `SendsWebhookMessages` 监听器将待发送消息分发到队列

这两步是**解耦**的——生成消息的路径不一定会立即触发投递，反之亦然。

**各业务路径的两步流程对比：**

| 业务操作 | 消息生成 | 立即投递触发 | 代码位置 |
|---------|---------|-------------|---------|
| 交易创建 | ✅ `fireWebhooks=true` 时生成 | ✅ 立即发布 | [TransactionGroupRepository::store()](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php#L337-L372) |
| 交易更新 | ✅ `fireWebhooks=true` 时生成 | ✅ 立即发布 | UpdateController 等 |
| 交易删除 | ✅ `fireWebhooks=true` 时生成 | ✅ 立即发布 | [TransactionGroupDestroyService](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Services/Internal/Destroy/TransactionGroupDestroyService.php#L36-L54) |
| 预算创建 | ✅ 始终生成（监听器不检查参数） | ✅ 立即发布 | [BudgetRepository::store()](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Repositories/Budget/BudgetRepository.php#L544-L640) |
| 预算更新 | ✅ 始终生成（监听器不检查参数） | ✅ 立即发布 | [BudgetRepository::update()](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Repositories/Budget/BudgetRepository.php#L642-L692) |
| **预算删除** | ✅ 始终生成（DestroyingBudget 事件） | ❌ **不发布！** | [BudgetDestroyService](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Services/Internal/Destroy/BudgetDestroyService.php#L36-L85) |
| 预算限额创建 | ✅ `createWebhookMessages=true` 时生成 | ✅ 立即发布 | [BudgetLimitRepository::store()](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Repositories/Budget/BudgetLimitRepository.php#L328-L356) |
| 预算限额更新 | ✅ `createWebhookMessages=true` 时生成 | ✅ 立即发布 | [BudgetLimitRepository::update()](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Repositories/Budget/BudgetLimitRepository.php#L360-L410) |
| 预算限额删除 | ✅ `createWebhookMessages=true` 时生成 | ✅ 立即发布 | [BudgetLimitRepository::destroyBudgetLimit()](file:///d:/fz/0508-2/solo-dogfeeding/code/126-firefly-iii/app/Repositories/Budget/BudgetLimitRepository.php#L121-L133) |

> **重要发现**：预算删除（`BudgetDestroyService`）是唯一的例外——它只生成 Webhook 消息，但**不立即发布投递请求**。删除预算产生的 Webhook 消息需要依赖页面随机兜底、手动触发或定时任务来实际发送。

---

## 三、消息生成机制

### 3.1 消息生成器（StandardMessageGenerator）

消息生成器实现了 `MessageGeneratorInterface` 接口，是消息生成的核心组件。

**工作流程：**

1. **设置上下文**：通过 `setUser()`、`setTrigger()`、`setObjects()` 设置生成上下文
2. **筛选 Webhook**：调用 `getWebhooks()` 根据触发器筛选出匹配的活跃 Webhook
3. **生成消息**：遍历每个 Webhook 和每个对象，调用 `generateMessage()` 生成消息
4. **持久化消息**：通过 `WebhookMessageFactory` 创建 `WebhookMessage` 记录

### 3.2 Webhook 匹配规则

消息生成器通过以下逻辑匹配需要触发的 Webhook：

```sql
WHERE active = true 
  AND (webhook_triggers.title = :trigger_name OR webhook_triggers.title = 'ANY')
```

即：触发器精确匹配 **或** 设置了 `ANY` 触发器的活跃 Webhook 都会被选中。

### 3.3 响应内容类型（WebhookResponse）

响应类型枚举定义了 Webhook 消息中包含的数据内容：

| 枚举值 | 数值 | 说明 |
|--------|------|------|
| `TRANSACTIONS` | 200 | 返回交易详情 |
| `ACCOUNTS` | 210 | 返回关联账户信息 |
| `BUDGET` | 230 | 返回预算详情 |
| `RELEVANT` | 240 | 自动返回与事件相关的数据 |
| `NONE` | 220 | 不返回具体内容（仅通知） |

#### RELEVANT 响应的自动映射

当 Webhook 设置为 `RELEVANT` 响应时，系统会根据触发事件的对象类型自动选择实际的响应内容：

| 触发对象类型 | 实际响应类型 |
|-------------|-------------|
| `TransactionGroup`（交易组） | `TRANSACTIONS` |
| `Budget` / `BudgetLimit`（预算/预算限额） | `BUDGET` |

#### 触发器与响应的约束

配置文件中定义了触发器与响应之间的禁止组合：

- `ANY` 触发器不能与 `BUDGET`、`TRANSACTIONS`、`ACCOUNTS` 响应搭配
- 交易类触发器（`STORE_TRANSACTION`、`UPDATE_TRANSACTION`、`DESTROY_TRANSACTION`）不能与 `BUDGET` 响应搭配
- 预算类触发器（`STORE_BUDGET`、`UPDATE_BUDGET`、`DESTROY_BUDGET`、`STORE_UPDATE_BUDGET_LIMIT`）不能与 `TRANSACTIONS`、`ACCOUNTS` 响应搭配

### 3.4 消息工厂（WebhookMessageFactory）

`WebhookMessageFactory` 负责创建 `WebhookMessage` 模型实例并持久化到数据库：

```php
$webhookMessage = new WebhookMessage();
$webhookMessage->webhook()->associate($webhook);
$webhookMessage->sent    = false;  // 初始状态：未发送
$webhookMessage->errored = false;  // 初始状态：无错误
$webhookMessage->uuid    = $data['uuid'];  // 消息唯一标识
$webhookMessage->message = $data;  // 消息内容（JSON 存储）
$webhookMessage->save();
```

### 3.5 消息体结构

生成的 Webhook 消息包含以下字段：

```json
{
  "uuid": "a1b2c3d4-...（UUID v4）",
  "user_id": 0,
  "user_group_id": 0,
  "trigger": "STORE_TRANSACTION",
  "response": "TRANSACTIONS",
  "url": "https://example.com/webhook",
  "version": "v0",
  "content": { ... }
}
```

### 3.6 不同预算事件的消息内容差异

预算相关的三类事件（普通预算更新、预算删除、预算限额变更）虽然最终投递的都是预算类数据，但在触发时机、传入对象和消息内容上存在差异。

#### 普通预算更新（STORE_BUDGET / UPDATE_BUDGET）

- **触发对象**：`Budget` 模型实例
- **触发器**：`STORE_BUDGET` 或 `UPDATE_BUDGET`
- **content 数据源**：`BudgetTransformer` 对 Budget 模型进行转换
- **数据特点**：包含预算的基本信息（名称、活跃状态、币种等），通过 `BudgetEnrichment` 进行数据补充后输出

#### 预算删除（DESTROY_BUDGET）

- **触发时机**：删除前触发（`DestroyingBudget` 事件），确保消息生成时模型数据仍然存在
- **触发对象**：`Budget` 模型实例（删除前的完整数据）
- **触发器**：`DESTROY_BUDGET`
- **content 数据源**：与预算更新相同，使用 `BudgetTransformer` 转换 Budget 模型
- **数据特点**：消息中包含的是被删除预算的完整数据，接收方可以根据 `trigger = DESTROY_BUDGET` 判断这是删除通知

> **注意**：预算删除使用的是 `DestroyingBudget`（进行时）事件而非 `DestroyedBudget`（完成时）事件，目的是在预算被真正删除前生成包含完整数据的 Webhook 消息。

#### 预算限额变更（STORE_UPDATE_BUDGET_LIMIT）

- **触发场景**：创建、更新、删除预算限额时均触发
- **触发器**：统一为 `STORE_UPDATE_BUDGET_LIMIT`（三种操作共用同一个触发器，不做细分）
- **触发对象**：**`Budget` 模型实例**（注意：不是 `BudgetLimit` 对象，而是所属的预算对象）
- **content 数据源**：`BudgetTransformer` 对 Budget 模型进行转换
- **数据特点**：消息中的 `content` 是预算（Budget）的数据，而非预算限额（BudgetLimit）的数据；接收方只能通过 `trigger = STORE_UPDATE_BUDGET_LIMIT` 知道是预算限额发生了变化，但无法从 content 中直接获取限额详情

> **重要提示**：预算限额事件传入消息生成器的对象是 Budget 而非 BudgetLimit，这与直觉可能不同。如果 Webhook 的响应类型设置为 `BUDGET`，则 content 中输出的是预算信息；如果设置为 `RELEVANT`，也会自动映射到 `BUDGET` 响应，内容仍然是预算数据。目前无法通过 Webhook 直接获取预算限额的详细数据。

---

## 四、调度入口与投递触发

Webhook 消息生成后，需要通过某种机制触发实际的投递流程。`WebhookMessagesRequestSending` 事件是投递流程的统一入口，它有四种触发方式，共同保障消息能够被及时、可靠地投递。

### 4.1 四种调度入口的关系

四种入口互为补充，形成了"主动推送 + 兜底保障"的多层投递机制：

| 触发方式 | 实时性 | 可靠性 | 触发场景 |
|---------|--------|--------|---------|
| 业务保存后立即发布 | 最高 | 高 | 业务操作完成后立刻触发 |
| 页面随机兜底 | 低（随机） | 中 | 用户浏览页面时随机触发，作为轻量级兜底 |
| 手动触发接口 | 按需 | 高 | 用户通过 API 主动触发 |
| 定时任务 | 低（10分钟） | 最高 | 周期性兜底，保证消息最终被投递 |

### 4.2 业务保存后立即发布

**机制说明**：在业务操作（创建/更新/删除交易、预算、预算限额等）完成后，立即发布 `WebhookMessagesRequestSending` 事件，触发投递。

**触发位置**：散布在各个业务 Repository 或 Service 中：

- **交易创建**：`TransactionGroupRepository::store()` 方法，创建交易组后立即触发
- **交易克隆**：`GroupCloneService` 中，克隆交易组后触发
- **交易删除**：`TransactionGroupDestroyService` 中，删除交易组后触发
- **预算创建**：`BudgetRepository` 的创建方法中
- **预算更新**：`BudgetRepository` 的更新方法中
- **预算限额创建/更新/删除**：`BudgetLimitRepository` 的对应方法中

**代码模式**：

```php
// 业务操作完成后，直接发布事件
event(new CreatedSingleTransactionGroup($flags, $objects));
event(new WebhookMessagesRequestSending());
```

**特点**：
- 实时性最高，消息生成后几乎立即开始投递
- 依赖业务代码显式调用，可能存在遗漏
- 与监听器异步生成消息的模式配合：监听器生成消息，此处触发投递

### 4.3 页面随机兜底（Lottery 机制）

**机制说明**：用户访问 Web 页面时，以一定概率随机触发 `WebhookMessagesRequestSending` 事件。

**触发位置**：基础控制器 `Controller` 的构造函数中的全局中间件内。

**代码实现**：

```php
// lottery to send any remaining webhooks:
if (7 === random_int(1, 30)) {
    // trigger event to send them:
    event(new WebhookMessagesRequestSending());
}
```

**触发概率**：约 1/30（即约 3.3% 的页面请求会触发）

**触发前提**：用户已登录（`auth()->check()`）

**特点**：
- 作为轻量级兜底机制，弥补定时任务间隔较长的问题
- 无需额外的定时任务基础设施，利用正常的页面访问流量
- 触发频率与用户活跃度相关：用户越活跃，触发越频繁
- 不保证一定会触发，仅作为补充手段

### 4.4 手动触发接口

**机制说明**：提供 API 接口，允许用户或外部系统手动触发某个 Webhook 的投递。

**接口位置**：`api/v1/webhooks/{webhook}/trigger/{transactionGroup}`

**控制器**：`Webhook\ShowController::triggerTransaction()`

**工作流程**：

1. 接收 Webhook ID 和交易组 ID
2. 遍历该 Webhook 配置的所有触发器
3. 为每个触发器调用消息生成器，以指定的交易组为对象生成消息
4. 生成消息后，立即发布 `WebhookMessagesRequestSending` 事件触发投递

**代码关键片段**：

```php
// 遍历 webhook 的所有触发器
foreach ($webhook->webhookTriggers as $trigger) {
    $engine = app(MessageGeneratorInterface::class);
    $engine->setUser(auth()->user());
    $engine->setTrigger(WebhookTrigger::tryFrom((int) $trigger->key));
    $engine->setObjects(new Collection()->push($group));
    $engine->setWebhooks(new Collection()->push($webhook)); // 仅针对当前 webhook
    $engine->generateMessages();
}

// 触发投递
event(new WebhookMessagesRequestSending());
```

**特点**：
- 按需触发，可用于测试或手动重发
- 可以只针对特定 Webhook 生成消息（通过 `setWebhooks()` 指定）
- 响应状态码为 204（无内容）

### 4.5 定时任务（最可靠的兜底）

**机制说明**：通过 Cron 定时任务周期性地触发 `WebhookMessagesRequestSending` 事件，确保所有待发送的消息最终都能被处理。

**触发位置**：`WebhookCronjob` 定时任务

**执行频率**：每 10 分钟一次（通过 `last_webhook_job` 配置项控制，间隔需大于 600 秒）

**工作流程**：

1. 读取上次执行时间戳
2. 检查距上次执行是否超过 10 分钟
3. 若是则发布 `WebhookMessagesRequestSending` 事件
4. 更新上次执行时间

**触发方式**：通过系统 Cron 调用 `cron` 命令，或访问 `/cron` 路由触发

**特点**：
- 最可靠的兜底机制，不依赖用户活动或业务代码
- 间隔较长（10 分钟），实时性最差
- 与前三种方式形成互补：立即发布保证实时性，定时任务保证最终可达性

---

## 五、投递队列与重试机制

### 5.1 整体架构

Webhook 投递采用 **多层触发 + 队列异步发送** 的混合模式。`WebhookMessagesRequestSending` 事件是投递流程的统一入口，由四种方式触发后进入统一的队列发送流程：

```
                              ┌─────────────────────────────┐
                              │   调度入口（四种触发方式）   │
                              ├─────────────────────────────┤
                              │  1. 业务保存后立即发布       │
                              │  2. 页面随机兜底（1/30概率） │
                              │  3. 手动触发 API 接口       │
                              │  4. 定时任务（每10分钟）    │
                              └───────────────┬─────────────┘
                                              ↓
                          WebhookMessagesRequestSending 事件
                                              ↓
                              SendsWebhookMessages 监听器
                                              ↓
                        筛选待发送消息（sent=false 且 尝试次数≤2，每次最多5条）
                                              ↓
                        SendWebhookMessage Job（Laravel Queue 异步执行）
                                              ↓
                            StandardWebhookSender 执行发送
                                              ↓
                        ┌─────────────────────┴─────────────────────┐
                        ↓                                           ↓
                  成功 → sent=true                          失败 → sent=false, errored=true
                                                                   记录 WebhookAttempt
```

### 5.2 定时任务触发（WebhookCronjob）

`WebhookCronjob` 负责周期性地触发 Webhook 发送流程。

**执行频率：** 每 10 分钟一次（通过 `last_webhook_job` 配置记录上次执行时间戳，间隔需大于 600 秒）

**核心判断逻辑：**

```php
$lastTime = (int) FireflyConfig::get('last_webhook_job', 0)->data;
$diff = now()->getTimestamp() - $lastTime;
if ($diff > 600) {
    $this->fireWebhookMessages();
}
```

**触发方式：** 通过发布 `WebhookMessagesRequestSending` 事件来触发后续流程。

### 5.3 消息筛选规则

`SendsWebhookMessages` 监听器处理 `WebhookMessagesRequestSending` 事件时，按以下规则筛选待发送消息：

**筛选条件（同时满足）：**

1. **`sent = false`** — 消息处于未发送状态（发送失败后会被重置为此状态）
2. **`webhookAttempts()->count() <= 2`** — 历史尝试记录不超过 2 条

**数量限制：**

- 每次 Cronjob 执行最多处理 `5` 条消息（`splice(0, 5)`）

### 5.4 分发逻辑

筛选出消息后，按以下流程分发到队列：

```php
foreach ($messages as $message) {
    if (false === $message->sent) {
        $message->sent = true;       // 先标记为已发送（防止重复分发）
        $message->save();
        SendWebhookMessage::dispatch($message)->afterResponse();
    }
}
```

> **重要说明**：在分发前就将 `sent` 标记为 `true`，这是一种防止重复分发的乐观锁策略。如果后续实际发送失败，发送器会将 `sent` 重新设回 `false` 并标记 `errored = true`。

### 5.5 队列任务（SendWebhookMessage）

`SendWebhookMessage` 是实现了 `ShouldQueue` 接口的 Laravel Queue Job，负责异步执行发送：

```php
public function handle(): void
{
    $sender = app(WebhookSenderInterface::class);
    $sender->setMessage($this->message);
    $sender->send();
}
```

Job 本身不包含业务逻辑，只是委托给 `WebhookSenderInterface` 完成实际发送。

### 5.6 重试策略

**最大发送次数：3 次**（首次发送 + 2 次重试）

推导过程：
- 筛选条件为 `webhookAttempts()->count() <= 2`
- 当尝试记录数为 0 时：首次发送（第 1 次）
- 当尝试记录数为 1 时：第 1 次重试（第 2 次发送）
- 当尝试记录数为 2 时：第 2 次重试（第 3 次发送）
- 当尝试记录数达到 3 时：不再被选中，消息停留在失败状态

**重试间隔：** 约 10 分钟（由 Cronjob 执行频率决定）

**失败记录：** 每次发送失败都会创建一条 `WebhookAttempt` 记录，包含：
- `status_code`：HTTP 状态码（0 表示连接错误等网络异常）
- `logs`：错误信息和堆栈跟踪

### 5.7 过期清理

每次执行发送时，会自动清理已发送超过 14 天的消息记录：

```php
WebhookMessage::where('sent', true)
    ->where('created_at', '<', now()->subDays(14))
    ->delete();
```

---

## 六、签名机制

### 6.1 职责边界

- **发送端**：Firefly III 负责生成签名并放入请求头
- **接收端**：外部系统负责验证签名的有效性

本文档仅说明发送端的签名生成逻辑，并给出接收端验证的参考方法。

### 6.2 发送端签名生成

发送端使用 `Sha3SignatureGenerator` 生成签名，算法为 **HMAC-SHA3-256**。

**签名 payload 构造：**

```
payload = timestamp + "." + json_body
```

- `timestamp`：当前 Unix 时间戳（秒级，字符串形式）
- `json_body`：完整的请求体 JSON 字符串

**签名计算：**

使用 Webhook 配置的 `secret` 作为密钥，对 payload 进行 HMAC-SHA3-256 运算。

**签名头格式：**

```
Signature: t=1717986918,v1=abc123def456...
```

格式说明：
- `t=` 前缀：时间戳
- `v1=` 前缀：v1 版本的签名值（当前仅 v1 版本）
- 多部分之间用逗号分隔

### 6.3 接收端验证参考

接收端验证签名的一般步骤：

1. 从 `Signature` 请求头中解析出时间戳 `t` 和签名值 `v1`
2. 将时间戳与原始请求体拼接：`payload = t + "." + raw_body`
3. 使用约定的 secret，以 HMAC-SHA3-256 算法计算签名
4. 使用恒定时间比较算法，比对计算出的签名与 `v1` 值
5. 可选：校验时间戳的时效性（如 5 分钟内有效），防止重放攻击

### 6.4 其他请求头

发送请求时还会携带以下头信息：

| Header | 值/格式 | 说明 |
|--------|---------|------|
| `Content-Type` | `application/json` | 请求体类型 |
| `Accept` | `application/json` | 期望的响应类型 |
| `User-Agent` | `FireflyIII/{version}` | 客户端标识 |
| `connect_timeout` | `3.14` | 连接超时（秒） |
| `timeout` | `10` | 请求总超时（秒） |

---

## 七、发送执行流程

### 7.1 完整流程

`StandardWebhookSender` 的 `send()` 方法是实际执行发送的核心，完整流程如下：

```
1. 设置 sent = true（与分发时的乐观锁呼应，再次确认）
2. 验证 Webhook URL 的合法性（IsValidWebhookUrl 规则）
3. 调用签名生成器生成签名
   └─ 失败：记录 WebhookAttempt，sent=false，errored=true，直接返回
4. 将消息内容序列化为 JSON
   └─ 失败：记录 WebhookAttempt，sent=false，errored=true，直接返回
5. 使用 GuzzleHttp 发送 POST 请求
   ├─ 成功：sent = true，记录响应日志
   └─ 失败（ConnectException / RequestException）：
        ├─ 提取状态码和响应体
        ├─ 记录 WebhookAttempt
        ├─ sent = false，errored = true
        └─ 返回
```

### 7.2 异常处理

发送过程中可能遇到三类异常：

| 异常类型 | 触发场景 | 处理方式 |
|---------|---------|---------|
| `FireflyException` | 签名生成失败（如 Webhook 已被删除） | 记录尝试，标记错误，返回 |
| `JsonException` | 消息内容 JSON 序列化失败 | 记录尝试，标记错误，返回 |
| `ConnectException` / `RequestException` | 网络连接错误或 HTTP 错误响应 | 记录状态码和响应体，标记错误 |

---

## 八、数据模型

### 8.1 核心模型关系

```
Webhook (1) ────→ (N) WebhookMessage (1) ────→ (N) WebhookAttempt
    │
    ├── (N) WebhookTrigger（多对多关联表：webhook_webhook_trigger）
    ├── (N) WebhookResponse（多对多关联表：webhook_webhook_response）
    └── (N) WebhookDelivery（多对多关联表：webhook_webhook_delivery）
```

### 8.2 Webhook 模型

Webhook 模型代表一个 Webhook 配置，主要字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `active` | boolean | 是否启用 |
| `trigger` | integer | 触发器类型（枚举值） |
| `response` | integer | 响应类型（枚举值） |
| `delivery` | integer | 投递方式（JSON = 300） |
| `url` | string | Webhook 接收地址 |
| `secret` | string | 签名密钥 |
| `title` | string | 配置标题 |
| `user_id` | integer | 所属用户 |

### 8.3 WebhookMessage 模型

WebhookMessage 模型代表一条待发送或已发送的消息，主要字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `webhook_id` | integer | 关联的 Webhook 配置 |
| `sent` | boolean | 是否已发送 |
| `errored` | boolean | 是否发送出错 |
| `uuid` | string | 消息唯一标识 |
| `message` | json | 消息内容 |
| `logs` | json | 日志信息 |

### 8.4 WebhookAttempt 模型

WebhookAttempt 模型记录每次发送尝试的结果，主要字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `webhook_message_id` | integer | 关联的消息 |
| `status_code` | integer | HTTP 响应状态码（0 表示网络异常） |
| `logs` | text | 日志或错误详情 |

---

## 九、服务容器绑定

所有 Webhook 相关服务通过接口绑定到服务容器，便于扩展和测试：

```php
$this->app->bind(MessageGeneratorInterface::class, StandardMessageGenerator::class);
$this->app->bind(SignatureGeneratorInterface::class, Sha3SignatureGenerator::class);
$this->app->bind(WebhookSenderInterface::class, StandardWebhookSender::class);
```

---

## 十、设计决策分析

### 10.1 为什么用多层触发而不是单一触发方式？

四种触发方式互为补充，形成了"实时 + 兜底"的多层保障机制：

- **业务保存后立即发布**：保证实时性，让绝大多数消息能在事件发生后立即投递
- **页面随机兜底**：轻量级补充，利用用户访问流量偶尔触发，不依赖定时任务基础设施
- **手动触发接口**：满足测试和手动重发的需求
- **定时任务**：最可靠的兜底，确保所有消息最终都能被处理

### 10.2 为什么用 Cronjob + 队列的混合投递模式？

在投递执行层面，采用 Cronjob 触发 + 队列异步发送的混合模式，原因有三：

- **削峰填谷**：大量交易同时创建时，避免瞬间产生大量队列任务
- **重试天然支持**：利用定时任务的周期性，无需额外配置重试队列
- **流量可控**：每次只处理 5 条，对外部接收系统的压力小且可预测

### 10.3 为什么先标记 sent=true 再发送？

这是一种 **乐观锁** 策略：
- 防止多次触发重复分发同一条消息（在消息入队前就标记）
- 如果发送失败，发送器负责将 sent 重新设回 false
- 潜在风险：极端情况下（进程在标记后、分发前崩溃）可能出现消息"失踪"——标记为已发送但实际未入队

### 10.4 为什么重试次数是 3 次？

- 平衡投递可靠性与系统资源消耗
- 配合约 10 分钟的重试间隔，总重试窗口约 20 分钟
- 持续失败的 Webhook 往往意味着接收端故障或配置错误，继续重试收益有限

### 10.5 为什么用 HMAC-SHA3-256？

SHA-3（Keccak）是最新的 SHA 标准，相比 SHA-2 在密码学安全性上更强，且对量子计算攻击有更好的抗性。HMAC 结构确保了签名的可验证性和不可伪造性。
