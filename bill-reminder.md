# Firefly III 账单提醒与预计交易的协作链路

## 概述

Firefly III 中涉及两个独立但可协作的概念：

- **Bill（周期账单 / 订阅）**：追踪预期中的周期性支出，监控是否已按时支付
- **Recurrence（循环交易）**：自动按计划创建交易的自动化机制

二者通过 `bill_id` 关联——Recurrence 在创建交易时可指定关联某个 Bill，使生成的交易自动被标记为"已支付该账单"。

---

## 一、全局调度入口

所有定时任务通过统一的 Artisan 命令启动：

```
php artisan firefly-iii:cron
```

[Cron.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Console/Commands/Tools/Cron.php#L41-L144)

该命令支持以下与本文相关的选项：

| 选项 | 作用 |
|------|------|
| `--create-recurring` | 仅执行循环交易生成任务 |
| `--send-subscription-warnings` | 仅执行账单到期提醒任务 |
| `--force` | 强制执行，忽略时间间隔限制 |
| `--date=YYYY-MM-DD` | 模拟指定日期执行 |

不加任何选项时，所有定时任务（含上述两个）都会按序执行。

---

## 二、链路一：循环交易（Recurrence）驱动预计交易生成

### 2.1 调用链路总览

```
Cron::handle()
  → Cron::recurringCronJob()
    → RecurringCronjob::fire()           # 频率控制（≥12h 间隔）
      → RecurringCronjob::fireRecurring()
        → CreateRecurringTransactions::handle()   # 核心业务 Job
          → filterRecurrences()          # 过滤无效 Recurrence
          → handleRepetitions()          # 逐个处理重复规则
            → handleOccurrences()        # 逐个处理发生日期
              → handleOccurrence()       # 当天则创建交易
                → getTransactionData()
                  → RecurringRepository::getBillId()  # 从 rt_meta 读取 bill_id
                  → TransactionGroupRepository::store()
                    → TransactionGroupFactory::create()
                      → TransactionJournalFactory::createJournal()
                        → TransactionJournal::create()  # bill_id 写入 transaction_journals
          → event(TransactionGroupsRequestedReporting)  # 事后通知
```

### 2.2 各环节详解

#### 2.2.1 RecurringCronjob — 频率控制

[RecurringCronjob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/Cronjobs/RecurringCronjob.php#L38-L97)

- 读取 `FireflyConfig` 中 `last_rt_job` 配置，判断距上次执行是否超过 **43,200 秒（12 小时）**
- 12 小时内不会重复执行（除非 `--force`）
- 执行完毕后更新 `last_rt_job` 为当前时间戳

#### 2.2.2 CreateRecurringTransactions — 核心业务 Job

[CreateRecurringTransactions.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/CreateRecurringTransactions.php#L50-L464)

**步骤一：获取所有 Recurrence**

```php
$this->recurrences = $this->repository->getAll();  // 从 recurrences 表获取全部记录
```

[RecurringRepository::getAll()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/Recurring/RecurringRepository.php#L136-L144)

**步骤二：过滤 — `filterRecurrences()` → `validRecurrence()`**

[validRecurrence()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/CreateRecurringTransactions.php#L412-L464)

过滤条件（任一不满足即跳过）：

| 条件 | 说明 |
|------|------|
| `active` | Recurrence 必须处于激活状态 |
| `repetitions` 限制 | 若设置了最大执行次数，且已创建的 journal 数 ≥ 此值则跳过 |
| `repeat_until` 已过 | 若设置了终止日期且已过则跳过 |
| `first_date` 尚未到 | 若首次执行日期在未来则跳过 |
| `latest_date` = 今天 | 今天已执行过则跳过（除非 force） |

**步骤三：处理重复规则 — `handleRepetitions()`**

[handleRepetitions()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/CreateRecurringTransactions.php#L352-L378)

- 每个 Recurrence 可有多条 `RecurrenceRepetition`（重复规则）
- 调用 `repository->getOccurrencesInRange()` 计算从 `first_date` 到「今天+2天」范围内的所有发生日期
- 发生日期计算支持 5 种重复类型：`daily`、`weekly`、`monthly`、`ndom`（每月第N个星期X）、`yearly`

[getOccurrencesInRange()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/Recurring/RecurringRepository.php#L264-L291)

**步骤四：按日期创建交易 — `handleOccurrence()`**

[handleOccurrence()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/CreateRecurringTransactions.php#L261-L322)

仅当发生日期等于今天时才执行：

1. 检查当天是否已创建过 journal（`getJournalCount`）
2. 检查是否已有相同 `recurrence_id` + `recurrence_date` 的历史记录（`createdPreviously`）
3. 构建 `getTransactionData()` 数组，其中 **`bill_id` 来自 `RecurrenceTransactionMeta`**：

```php
'bill_id' => $this->repository->getBillId($transaction),
```

[getBillId()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/Recurring/RecurringRepository.php#L146-L158)

`getBillId()` 的实现非常直接：

```php
public function getBillId(RecurrenceTransaction $recurrence): int
{
    $meta = $recurrence->recurrenceTransactionMeta()
        ->where('name', 'bill_id')
        ->first();
    return null !== $meta ? (int) $meta->value : 0;
}
```

它从 `rt_meta` 表中查找 `name='bill_id'` 的记录，读取其 `value` 字段。

4. 调用 `groupRepository->store($array)` 创建 `TransactionGroup`
5. 更新 Recurrence 的 `latest_date` 为当天

**步骤五：交易创建时 bill_id 的落库链路**

当 `groupRepository->store($data)` 被调用后，bill_id 的传递路径如下：

[TransactionGroupRepository::store()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php#L349-L380)

→ 委托给 [TransactionGroupFactory::create()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Factory/TransactionGroupFactory.php#L57-L90)

→ 调用 [TransactionJournalFactory::create()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Factory/TransactionJournalFactory.php#L230-L369) 的 `createJournal()` 方法

在 `createJournal()` 中：

```php
// L247: 从 $row['bill_id'] 查找 Bill 对象
$bill = $this->billRepository->findBill((int) $row['bill_id'], $row['bill_name']);

// L248: 只有 WITHDRAWAL 类型交易才允许关联 bill
$billId = TransactionTypeEnum::WITHDRAWAL->value === $type->type && $bill instanceof Bill ? $bill->id : null;

// L330-L342: 创建 TransactionJournal 时写入 bill_id
$journal = TransactionJournal::create([
    ...
    'bill_id' => $billId,
    ...
]);
```

`bill_id` 最终保存在 `transaction_journals` 表的 `bill_id` 列中。[TransactionJournal 模型](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Models/TransactionJournal.php#L60-L72) 的 fillable 字段包含 `bill_id`。

**关键约束**：只有交易类型为 `WITHDRAWAL` 时，bill_id 才会被实际写入。Deposit 和 Transfer 类型的 bill_id 会被强制置为 null——这是因为账单本质上追踪的是支出。

**步骤六：事后事件通知**

```php
event(new TransactionGroupsRequestedReporting($userId, $journals));
```

[TransactionGroupsRequestedReporting](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Events/Model/TransactionGroup/TransactionGroupsRequestedReporting.php#L31-L42)

### 2.3 交易创建后的邮件通知

[MailsNewTransactionsReport](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Listeners/Model/TransactionGroup/MailsNewTransactionsReport.php#L37-L75)

- 监听 `TransactionGroupsRequestedReporting` 事件
- 检查用户偏好 `notification_transaction_creation`（默认关闭）
- 通过 `NotificationSender` 发送 `TransactionCreation` 通知

---

## 三、Recurrence 保存时 bill_id 的写入链路（Web + API 完整入口）

### 3.1 Recurrence 交易模板中 bill_id 的存储结构

Recurrence 的交易模板由三层模型构成：

```
Recurrence (recurrences 表)
 └─ RecurrenceTransaction (recurrences_transactions 表)
      └─ RecurrenceTransactionMeta (rt_meta 表)
           ├─ name='bill_id'       → value=账单ID
           ├─ name='budget_id'     → value=预算ID
           ├─ name='category_id'   → value=分类ID
           ├─ name='piggy_bank_id' → value=存钱罐ID
           └─ name='tags'          → value=JSON数组
```

[RecurrenceTransactionMeta 模型](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Models/RecurrenceTransactionMeta.php#L33-L56)

- 表名：`rt_meta`
- 关键字段：`rt_id`（关联 RecurrenceTransaction）、`name`、`value`（存储为 string）
- bill_id 存储方式：`name='bill_id'`, `value='123'`（string 形式的 ID）

### 3.2 保存入口全景图

Firefly III 有两套独立的入口创建/更新 Recurrence，但**最终汇聚到同一套业务逻辑**：

```
                ┌─────────────────────────────────┐
                │  HTTP 请求                      │
                └─────┬─────────────────────┬─────┘
                      │                     │
                ┌─────▼─────┐        ┌──────▼──────┐
                │ Web 表单   │        │ JSON API v1 │
                └─────┬─────┘        └──────┬──────┘
                      │                     │
    ┌─────────────────▼──────────┐  ┌──────▼──────────────────┐
    │ RecurrenceFormRequest      │  │ StoreRequest /          │
    │ ::getAll()                 │  │ UpdateRequest           │
    │ (app/Http/Requests)        │  │ ::getAll()              │
    └─────────────┬──────────────┘  │ (app/Api/V1/Requests)   │
                  │                 └──────┬──────────────────┘
                  │                        │
                  └───────────┬────────────┘
                              ▼
                ┌───────────────────────────┐
                │ RecurringRepository       │
                │ ::store() / ::update()    │
                └─────┬─────────────────┬───┘
                      │                 │
              ┌───────▼──────┐  ┌──────▼──────────┐
              │ Recurrence   │  │ Recurrence       │
              │ Factory      │  │ UpdateService    │
              │ ::create()   │  │ ::update()       │
              └───────┬──────┘  └──────┬──────────┘
                      │                 │
                      └────────┬────────┘
                               ▼
                   ┌───────────────────────────┐
                   │ RecurringTransactionTrait │
                   │ ::setBill()               │
                   │   → upsert rt_meta        │
                   └───────────────────────────┘
```

### 3.3 Web 表单保存入口

**路由定义**（来自 [routes/web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/routes/web.php#L913-L914)）：

| 动作 | 路由 |
|------|------|
| 创建 | `POST /recurring/store` → [CreateController::store()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Http/Controllers/Recurring/CreateController.php#L228-L269) |
| 更新 | `POST /recurring/update/{id}` → [EditController::update()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Http/Controllers/Recurring/EditController.php#L182-L215) |

**Web 创建入口**：

```php
// CreateController::store()
$data       = $request->getAll();
$recurrence = $this->repository->store($data);
```

**Web 更新入口**：

```php
// EditController::update()
$data       = $request->getAll();
$recurrence = $this->repository->update($recurrence, $data);
```

### 3.4 JSON API v1 保存入口

**路由定义**（来自 [routes/api.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/routes/api.php#L518-L527)）：

| 动作 | 路由 |
|------|------|
| 创建 | `POST /api/v1/recurrences` → [StoreController::store()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Api/V1/Controllers/Models/Recurrence/StoreController.php#L66-L86) |
| 更新 | `PUT /api/v1/recurrences/{recurrence}` → [UpdateController::update()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Api/V1/Controllers/Models/Recurrence/UpdateController.php#L64-L84) |

**API 创建入口**：

```php
// StoreController::store()
$data       = $request->getAll();
$recurrence = $this->repository->store($data);
```

**API 更新入口**：

```php
// UpdateController::update()
$data       = $request->getAll();
$recurrence = $this->repository->update($recurrence, $data);
```

### 3.5 请求数据提取（Web vs API 对比）

#### 3.5.1 Web 请求 — RecurrenceFormRequest

[RecurrenceFormRequest::getAll()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Http/Requests/RecurrenceFormRequest.php#L58-L150)

Web 表单中 bill_id 是顶级字段（因为 Web 表单每个 Recurrence 只有一个交易模板）：

```php
// L84: 直接从顶级表单字段 'bill_id' 提取
'transactions' => [[
    ...
    'bill_id' => $this->convertInteger('bill_id'),
    'bill_name' => null,
    ...
]],
```

验证规则：

```php
// L188: bill_id 必须存在于 bills 表且属于当前用户
'bill_id' => ['mustExist:bills,id', 'belongsToUser:bills,id', 'nullable'],
```

#### 3.5.2 API 请求 — StoreRequest / UpdateRequest

[StoreRequest::getAll()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Api/V1/Requests/Models/Recurrence/StoreRequest.php#L57-L73)
[UpdateRequest::getAll()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Api/V1/Requests/Models/Recurrence/UpdateRequest.php#L58-L80)

API 中 bill_id 是交易数组的嵌套字段（因为 API 支持多交易模板）：

```php
// StoreRequest::getAll()
return [
    'recurrence'   => $recurrence,
    'transactions' => $this->getTransactionData(),  // 从 'transactions' 数组提取
    'repetitions'  => $this->getRepetitionData(),
];
```

`getTransactionData()` 调用 `getSingleTransactionData()`，该方法由 **[GetRecurrenceData trait](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/Request/GetRecurrenceData.php#L32-L56)** 提供：

```php
protected function getSingleTransactionData(array $transaction): array
{
    $return     = [];
    $stringKeys = ['id'];
    // L36: bill_id 在 intKeys 数组中，会被转为 int
    $intKeys    = ['currency_id', 'foreign_currency_id', 'source_id', 
                   'destination_id', 'bill_id', 'piggy_bank_id', 
                   'bill_id', 'budget_id', 'category_id'];
    $keys       = ['amount', 'currency_code', 'foreign_amount', ...];

    foreach ($intKeys as $key) {
        if (array_key_exists($key, $transaction)) {
            $return[$key] = (int) $transaction[$key];
        }
    }
    // ...
    return $return;
}
```

API 请求体示例：

```json
{
    "type": "withdrawal",
    "title": "Monthly Rent",
    "first_date": "2026-07-01",
    "repetitions": [{"type": "monthly", "moment": "1", "skip": 0}],
    "transactions": [{
        "amount": 3000,
        "source_id": 1,
        "destination_name": "Landlord",
        "bill_id": 5
    }]
}
```

### 3.6 统一保存层 — RecurringRepository

Web 和 API 都通过 [RecurringRepository](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/Recurring/RecurringRepository.php) 作为统一入口：

```php
// L550-L557: store() 方法
public function store(array $data): Recurrence
{
    /** @var RecurrenceFactory $factory */
    $factory = app(RecurrenceFactory::class);
    $factory->setUser($this->user);
    return $factory->create($data);
}

// L585-L591: update() 方法
public function update(Recurrence $recurrence, array $data): Recurrence
{
    /** @var RecurrenceUpdateService $service */
    $service = app(RecurrenceUpdateService::class);
    return $service->update($recurrence, $data);
}
```

### 3.7 创建 Recurrence 时的 bill_id 写入

[RecurrenceFactory::create()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Factory/RecurrenceFactory.php#L60-L138)

流程：

1. 解析 `$data['recurrence']` 基本字段并创建 `Recurrence` 主记录
2. 调用 `$this->createRepetitions()` 创建重复规则
3. 调用 `$this->createTransactions()` 创建交易模板（含 bill_id）

`createTransactions()` 方法由 [RecurringTransactionTrait](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Services/Internal/Support/RecurringTransactionTrait.php#L94-L166) 提供：

```php
// L94-L166: createTransactions()
// L135-L145: 创建 RecurrenceTransaction 主记录（不含 bill_id）
$transaction = new RecurrenceTransaction([
    'recurrence_id'           => $recurrence->id,
    'transaction_currency_id' => $currency->id,
    'source_id'               => $source->id,
    ...
]);
$transaction->save();

// L150-L152: 如果 $array 包含 bill_id 键，调用 setBill()
if (array_key_exists('bill_id', $array)) {
    $this->setBill($transaction, (int) $array['bill_id']);
}
```

核心方法 [setBill()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Services/Internal/Support/RecurringTransactionTrait.php#L275-L295)：

```php
private function setBill(RecurrenceTransaction $transaction, int $billId): void
{
    $billFactory = app(BillFactory::class);
    $billFactory->setUser($transaction->recurrence->user);
    $bill = $billFactory->find($billId, null);  // 校验 billId 有效性

    if (null === $bill) {
        // bill 不存在，删除之前的关联（如果有）
        $transaction->recurrenceTransactionMeta()
            ->where('name', 'bill_id')->delete();
        return;
    }

    // bill 有效，创建或更新 rt_meta 记录
    $meta = $transaction->recurrenceTransactionMeta()
        ->where('name', 'bill_id')->first();

    if (null === $meta) {
        $meta        = new RecurrenceTransactionMeta();
        $meta->rt_id = $transaction->id;
        $meta->name  = 'bill_id';
    }
    $meta->value = $bill->id;  // 注意：存入的是 string 类型
    $meta->save();
}
```

### 3.8 更新 Recurrence 时的 bill_id 处理

[RecurrenceUpdateService::update()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Services/Internal/Update/RecurrenceUpdateService.php#L55-L109)

更新时 bill_id 的处理在 `updateTransactions()` → `updateCombination()` 中：

1. [updateTransactions()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Services/Internal/Update/RecurrenceUpdateService.php#L283-L326)：
   - 将已有的交易模板与提交的数据按 `id` 进行匹配（匹配成功则更新，未匹配则删除，多余的提交项则新增）
   
2. [updateCombination()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Services/Internal/Update/RecurrenceUpdateService.php#L161-L241)：
   - 首先更新 `RecurrenceTransaction` 主记录的字段（金额、账户等）
   - **L219-L221**：如果提交数据包含 `bill_id`，调用 `$this->setBill()` 更新关联

```php
if (array_key_exists('bill_id', $submitted)) {
    $this->setBill($transaction, (int) $submitted['bill_id']);
}
```

`setBill()` 方法同样来自 `RecurringTransactionTrait`，逻辑与创建时完全一致。

### 3.9 保存入口汇总表

| 维度 | Web 入口 | JSON API v1 入口 |
|------|----------|------------------|
| 创建路由 | `POST /recurring/store` | `POST /api/v1/recurrences` |
| 更新路由 | `POST /recurring/update/{id}` | `PUT /api/v1/recurrences/{recurrence}` |
| 创建控制器 | [Recurring\CreateController](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Http/Controllers/Recurring/CreateController.php) | [Models\Recurrence\StoreController](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Api/V1/Controllers/Models/Recurrence/StoreController.php) |
| 更新控制器 | [Recurring\EditController](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Http/Controllers/Recurring/EditController.php) | [Models\Recurrence\UpdateController](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Api/V1/Controllers/Models/Recurrence/UpdateController.php) |
| 请求类 | [RecurrenceFormRequest](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Http/Requests/RecurrenceFormRequest.php) | [StoreRequest](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Api/V1/Requests/Models/Recurrence/StoreRequest.php) / [UpdateRequest](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Api/V1/Requests/Models/Recurrence/UpdateRequest.php) |
| bill_id 位置 | 表单顶级字段 `bill_id` | 嵌套字段 `transactions[*].bill_id` |
| 数据提取 Trait | （RecurrenceFormRequest 自身实现） | [GetRecurrenceData](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/Request/GetRecurrenceData.php) |
| 统一 Repository | **[RecurringRepository::store()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/Recurring/RecurringRepository.php#L550-L557) / [update()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/Recurring/RecurringRepository.php#L585-L591)** | 相同 |
| 创建 Factory | [RecurrenceFactory](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Factory/RecurrenceFactory.php) | 相同 |
| 更新 Service | [RecurrenceUpdateService](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Services/Internal/Update/RecurrenceUpdateService.php) | 相同 |
| bill_id 写入 Trait | **[RecurringTransactionTrait::setBill()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Services/Internal/Support/RecurringTransactionTrait.php#L275-L295)** | 相同 |

---

## 四、链路二：账单到期提醒（Bill Warning）

### 4.1 调用链路总览

```
Cron::handle()
  → Cron::subscriptionWarningCronJob()
    → BillWarningCronjob::fire()         # 频率控制（≥12h 间隔）
      → BillWarningCronjob::fireWarnings()
        → WarnAboutBills::handle()       # 核心业务 Job
          ├── 遍历所有用户 → 所有活跃 Bill
          ├── getDates()
          │   └── Navigation::startOfPeriod($date, $bill->repeat_freq)  ← 按账单周期计算窗口
          │   └── Navigation::endOfPeriod($start, $bill->repeat_freq)   ← 计算窗口结束
          │   └── SubscriptionEnrichment::enrichSingle()
          │       ├── collectPaidDates()        # 从 transaction_journals 查询已付款（只查当前周期）
          │       └── collectPayDates()         # 用 BillDateCalculator 计算预期付款日
          │           └── BillDateCalculator::getPayDates()
          ├── 过期检查：needsOverdueAlert()
          │   └── event(SubscriptionsAreOverdueForPayment)
          │       └── NotifiesAboutOverdueSubscriptions
          │           └── NotificationSender → SubscriptionsOverdueReminder
          └── 到期提醒：end_date / extension_date → needsWarning()
              └── event(SubscriptionNeedsExtensionOrRenewal)
                  └── NotifiesAboutExtensionOrRenewal
                      └── NotificationSender → BillReminder
```

### 4.2 各环节详解

#### 4.2.1 BillWarningCronjob — 频率控制

[BillWarningCronjob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/Cronjobs/BillWarningCronjob.php#L38-L100)

- 读取 `FireflyConfig` 中 `last_bw_job` 配置
- 同样 12 小时内不重复执行（除非 `--force`）
- 执行完毕后更新 `last_bw_job`

#### 4.2.2 WarnAboutBills — 核心业务 Job

[WarnAboutBills.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/WarnAboutBills.php#L44-L199)

遍历所有用户的所有活跃 Bill，执行两类检查：

**检查 A：账单过期（Overdue）**

```php
$dates = $this->getDates($bill);
if ($this->needsOverdueAlert($dates)) {
    $overdue[] = ['bill' => $bill, 'dates' => $dates];
}
```

1. `getDates()` 使用 `SubscriptionEnrichment` 计算当前周期内的 `pay_dates`（应付款日）和 `paid_dates`（已付款日）
2. `needsOverdueAlert()` 判断逻辑：

```php
$count = count($pay_dates) - count($paid_dates);  // 未付款数量
$earliest = new Carbon($pay_dates[0]);
$diff = $earliest->diffInDays($this->date);
return $diff >= 6;  // 最早的应付款日距今 ≥ 6 天则判定过期
```

[needsOverdueAlert()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/WarnAboutBills.php#L158-L172)

**检查 B：到期/续期提醒（End Date / Extension Date）**

```php
if ($this->needsWarning($bill, 'end_date')) { ... }
if ($this->needsWarning($bill, 'extension_date')) { ... }
```

[needsWarning()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/WarnAboutBills.php#L174-L184)

判断逻辑：

```php
$diff = $this->getDiff($bill, $field);  // 今天与 end_date/extension_date 的天数差
$list = config('firefly.bill_reminder_periods');  // [90, 30, 14, 7, 0]
return in_array($diff, $list, true);
```

即：当到期日距今天数恰好为 **90 / 30 / 14 / 7 / 0 天** 时触发提醒。

配置来源：[firefly.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/config/firefly.php#L226)

```php
'bill_reminder_periods' => [90, 30, 14, 7, 0],
```

### 4.3 按账单周期计算查询日期窗口 — 核心算法

**这是账单提醒最关键的设计：每次只检查「当前账单周期」内的付款情况**，而非全量扫描历史。

#### 4.3.1 WarnAboutBills::getDates() — 日期窗口计算入口

[WarnAboutBills::getDates()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/WarnAboutBills.php#L117-L132)

```php
private function getDates(Bill $bill): array
{
    // 步骤 1: 取当前日期（Cron 传入的 date 或今天）
    $start = clone $this->date;
    
    // 步骤 2: 根据账单的 repeat_freq，找到这个日期所在周期的开始
    $start = Navigation::startOfPeriod($start, $bill->repeat_freq);
    
    // 步骤 3: 从周期开始日，计算同一周期的结束日
    $end = clone $start;
    $end = Navigation::endOfPeriod($end, $bill->repeat_freq);
    
    // 步骤 4: 用这个 [start, end] 窗口查询数据
    $enrichment = new SubscriptionEnrichment();
    $enrichment->setUser($bill->user);
    $enrichment->setStart($start);
    $enrichment->setEnd($end);
    
    $single = $enrichment->enrichSingle($bill);
    
    return [
        'pay_dates'  => $single->meta['pay_dates']  ?? [],
        'paid_dates' => $single->meta['paid_dates'] ?? [],
    ];
}
```

#### 4.3.2 Navigation::startOfPeriod() — 计算周期开始

[Navigation::startOfPeriod()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/Navigation.php#L651-L724)

根据 `repeat_freq` 映射到 Carbon 的日期方法：

```php
$functionMap = [
    'daily'     => 'startOfDay',      // 今天 00:00
    'weekly'    => 'startOfWeek',     // 本周一 00:00（参数 MONDAY）
    'monthly'   => 'startOfMonth',    // 本月 1 号 00:00
    'quarterly' => 'firstOfQuarter',  // 本季度第一天
    'yearly'    => 'startOfYear',     // 本年 1 月 1 日
    // ...
];
```

**half-year（半年）特殊处理**：

```php
if ('half-year' === $repeatFreq || '6M' === $repeatFreq) {
    $skipTo = $date->month > 7 ? 6 : 0;  // 1-6月→上半年(+0月)，7-12月→下半年(+6月)
    $date->startOfYear()->addMonths($skipTo);
    return $date;
}
```

#### 4.3.3 Navigation::endOfPeriod() — 计算周期结束

[Navigation::endOfPeriod()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/Navigation.php#L185-L285)

根据 `repeat_freq` 映射到加周期的方法，**对于周/月/季/年等周期，加完后还要再减 1 天**：

```php
$functionMap = [
    'daily'     => 'endOfDay',       // 今天 23:59:59.999999
    'weekly'    => 'addWeeks',       // +1 周
    'monthly'   => 'addMonths',      // +1 月
    'quarterly' => 'addQuarters',    // +1 季度
    'yearly'    => 'addYears',       // +1 年
    // ...
];

// 周/月/季/半年/年 需要额外减 1 天
$subDay = ['week', 'weekly', '1W', 'month', 'monthly', '1M', '3M', 
           'quarter', 'quarterly', '6M', 'half-year', 'half_year', 
           '1Y', 'year', 'yearly'];

// 伪代码逻辑：
if (in_array($repeatFreq, $subDay)) {
    $currentEnd->{$function}($modifier ?? 1);  // 加一个周期
    $currentEnd->subDay();                       // 减 1 天
    $currentEnd->endOfDay();                     // 到当天结束
}
```

#### 4.3.4 日期窗口示例

假设今天是 **2026-06-18**（周四），不同账单周期的查询窗口：

| Bill repeat_freq | startOfPeriod | endOfPeriod | 实际查询范围 |
|------------------|---------------|-------------|--------------|
| `daily` | 2026-06-18 00:00 | 2026-06-18 23:59:59 | 当天 |
| `weekly` | 2026-06-15（周一） | 2026-06-21 23:59:59（周日） | 本周 |
| `monthly` | 2026-06-01 | 2026-06-30 23:59:59 | 本月 |
| `quarterly` | 2026-04-01 | 2026-06-30 23:59:59 | Q2 |
| `half-year` | 2026-01-01 | 2026-06-30 23:59:59 | 上半年 |
| `yearly` | 2026-01-01 | 2026-12-31 23:59:59 | 全年 |

**算法验证**（以 monthly 为例）：
1. $date = 2026-06-18
2. startOfPeriod(monthly) → 2026-06-01
3. endOfPeriod(monthly, start=2026-06-01):
   - addMonths(1) → 2026-07-01
   - subDay() → 2026-06-30
   - endOfDay() → 2026-06-30 23:59:59.999999
4. 查询范围：2026-06-01 → 2026-06-30

#### 4.3.5 设计意图

这种按账单周期计算窗口的设计有以下优点：

1. **性能优化**：每个账单只查询其周期范围内的交易，而不是全表扫描
2. **业务准确**：账单是周期性的，只关心当前周期内是否已支付，历史周期的拖欠应该在之前的检查中已经处理
3. **避免重复提醒**：每个周期只检查一次该周期的付款状态，过期判定的 6 天窗口也是基于当前周期的最早应付款日计算

### 4.4 账单读取已付款记录的查询链路

#### 4.4.1 日期窗口传递到 SubscriptionEnrichment

日期窗口计算完成后，通过 `setStart()` 和 `setEnd()` 传递给 [SubscriptionEnrichment](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/JsonApi/Enrichments/SubscriptionEnrichment.php#L234-L349)：

```php
$enrichment->setStart($start);  // 周期开始
$enrichment->setEnd($end);      // 周期结束
```

#### 4.4.2 SubscriptionEnrichment::collectPaidDates() — 已付款记录查询

这是核心查询方法，**只查询 [start, end] 日期窗口内的记录**：

```php
// L248-L258: 准备日期范围（已按账单周期计算好）
$start = clone $this->start;
$searchStart = clone $start;
$start->subDay();  // 起始日期减 1 天作为上一周期的容错

$end = clone $this->end;
$searchEnd = clone $end;
$searchStart->startOfDay();
$searchEnd->endOfDay();

// L263-L288: 执行多表 JOIN 查询
$set = $this->user
    ->transactionJournals()           // 从 transaction_journals 表
    ->whereIn('bill_id', $this->subscriptionIds)  // 过滤：bill_id 在目标账单列表中
    ->leftJoin('transactions', 'transactions.transaction_journal_id', '=', 'transaction_journals.id')
    ->where('transactions.amount', '>', 0)  // 只看正向金额（支出交易的正向侧）
    ->before($searchEnd)                    // 日期范围：≤ 周期结束日
    ->after($searchStart)                   // 日期范围：≥ 周期开始日 - 1天
    ->get([
        'transaction_journals.id',
        'transaction_journals.date',        // 交易日期 = 付款日
        'transaction_journals.bill_id',     // 关联的账单 ID
        ...
    ]);
```

查询核心要点：

| 要素 | 说明 |
|------|------|
| 表 | `transaction_journals` JOIN `transactions` |
| 关联条件 | `transaction_journals.bill_id IN (账单ID列表)` |
| 金额过滤 | `transactions.amount > 0`（正向金额 = 支出方） |
| 日期范围 | `searchStart ≤ date ≤ searchEnd`（**已按账单周期动态计算**） |
| 容错 | `$start->subDay()` 使窗口向左扩展 1 天，容忍上一周期末的交易 |

**注意**：这里的日期范围不是固定值，而是**针对每个账单的 `repeat_freq` 动态计算**的。例如：
- 月付账单查询本月范围内的付款记录
- 周付账单查询本周范围内的付款记录

#### 4.4.3 预期付款日的计算 — BillDateCalculator

[BillDateCalculator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/Models/BillDateCalculator.php#L32-L173)

`getPayDates()` 方法核心逻辑：

1. 从 `earliest` 到 `latest` 逐日循环
2. 每轮调用 `nextDateMatch()` 计算下一个预期付款日
3. `nextDateMatch()` 通过 `Navigation::diffInPeriods()` 计算从账单起始日到当前日期跨了多少个周期
4. 跳过 `skip` 值对应的周期（如每 2 个月付一次则 skip=1）
5. 处理月末边界情况（如 30 号的账单在 2 月的处理）
6. 结果必须晚于 `lastPaid` 日期（排除已付款的周期）

**关键**：`getPayDates()` 接收 `$lastPaid` 参数——**只有在 lastPaid 之后的预期付款日才会被返回**，这确保了已付款的周期不会再出现在未付款列表中。

#### 4.4.4 已付款记录的查询方式汇总

| 场景 | 代码位置 | 查询方式 |
|------|----------|----------|
| Cron 过期提醒 | [SubscriptionEnrichment::collectPaidDates()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/JsonApi/Enrichments/SubscriptionEnrichment.php#L234-L349) | 按账单周期动态窗口，批量 JOIN 查询 `transaction_journals.bill_id` |
| 通用 API | [BillRepository::getPaidDatesInRange()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/Bill/BillRepository.php#L321-L349) | 调用方指定日期范围，通过 `Bill::transactionJournals()` 关系查询 |
| 模型关系 | [Bill::transactionJournals()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Models/Bill.php#L167-L170) | `HasMany` 关系，`transaction_journals.bill_id = bills.id` |

---

## 五、两条链路的协作机制

### 5.1 数据模型关系全景

```
┌───────────────────────────────────────────────────────────────┐
│                  bills 表 (Bill 模型)                          │
│  id, name, amount_min, amount_max, date, repeat_freq, skip,    │
│  active, end_date, extension_date, transaction_currency_id     │
│                                                               │
│  关系: transactionJournals() → HasMany                        │
│  repeat_freq 用于动态计算账单提醒查询窗口                       │
└──────────────────┬────────────────────────────────────────────┘
                   │
                   │ 1:N (通过 bill_id 外键)
                   ▼
┌───────────────────────────────────────────────────────────────┐
│            transaction_journals 表 (TransactionJournal)        │
│  id, user_id, bill_id, transaction_type_id, description,       │
│  date, amount, transaction_currency_id                         │
│                                                               │
│  ⬆ 这条记录由 Recurrence 链路通过 Cron 自动创建                 │
│     bill_id 字段 = Recurrence 交易模板中保存的 bill_id          │
│                                                               │
│  ⬇ 这条记录被 Bill 提醒链路读取，作为 paid_dates 的依据          │
│     查询窗口由 bill.repeat_freq 动态计算                        │
└───────────────────────────────────────────────────────────────┘
                   ▲
                   │
                   │  被 Bill 提醒链路读取，用于计算 paid_dates
                   │
┌───────────────────────────────────────────────────────────────┐
│            rt_meta 表 (RecurrenceTransactionMeta)              │
│  id, rt_id, name, value                                        │
│                                                               │
│  name='bill_id', value='123' ← 用户创建/更新 Recurrence 时写入  │
│                                                               │
│  ⬆ 被 Recurrence 链路读取，作为创建交易时的 bill_id 来源         │
└──────────────────┬────────────────────────────────────────────┘
                   │
                   │ N:1 (rt_id → recurrences_transactions.id)
                   ▼
┌───────────────────────────────────────────────────────────────┐
 │      recurrences_transactions 表 (RecurrenceTransaction)      │
│  id, recurrence_id, source_id, destination_id, amount, ...    │
└──────────────────┬────────────────────────────────────────────┘
                   │
                   │ N:1 (recurrence_id → recurrences.id)
                   ▼
┌───────────────────────────────────────────────────────────────┐
│              recurrences 表 (Recurrence)                       │
│  id, user_id, title, first_date, repeat_until, latest_date,   │
│  repetitions, active, apply_rules                              │
└───────────────────────────────────────────────────────────────┘
```

### 5.2 协作时序（单周期视角）

```
T0: 用户配置阶段
    ├── 创建 Bill: bills 表新增记录
    │    repeat_freq = 'monthly'（按月）
    │    date = 2026-06-01（账单起始日）
    └── 创建 Recurrence: recurrences + recurrences_transactions
                        + rt_meta(name='bill_id', value=Bill.id)

T1: Cron 执行循环交易任务 (Recurrence 链路)
    ├── RecurringCronjob 判断时间间隔
    ├── CreateRecurringTransactions Job
    │   ├── 获取所有活跃 Recurrence
    │   ├── 计算今天是否为发生日
    │   ├── 从 rt_meta 读取 bill_id=5
    │   ├── TransactionGroupRepository::store()
    │   │   └── TransactionJournalFactory::createJournal()
    │   │       └── TransactionJournal::create(['bill_id' => 5, ...])
    │   │           ↳ transaction_journals 表新增记录，bill_id=5
    │   └── 更新 Recurrence.latest_date = 今天
    └── 结束

T2: Cron 执行账单提醒任务 (Bill 链路)
    ├── BillWarningCronjob 判断时间间隔
    ├── WarnAboutBills Job
    │   ├── 遍历所有活跃 Bill (id=5)
    │   ├── getDates(bill #5):
    │   │   ├── start = today → 2026-06-18
    │   │   ├── start = Navigation::startOfPeriod(start, 'monthly')
    │   │   │         → 2026-06-01（本月初）
    │   │   ├── end = Navigation::endOfPeriod(start, 'monthly')
    │   │   │         → 2026-06-30（本月末）
    │   │   └── SubscriptionEnrichment
    │   │       ├── collectPaidDates(start=2026-06-01, end=2026-06-30)
    │   │       │   SELECT ... FROM transaction_journals
    │   │       │   WHERE bill_id = 5 
    │   │       │   AND date BETWEEN 2026-06-01 AND 2026-06-30
    │   │       │   → 找到 T1 创建的交易 → 该账单被标记为"已付"
    │   │       └── collectPayDates():
    │   │           BillDateCalculator 计算预期付款日
    │   │           → 只返回 lastPaid 之后的日期
    │   ├── needsOverdueAlert():
    │   │   pay_dates 数量 - paid_dates 数量 = 0 → 未过期，不提醒
    │   └── needsWarning():
    │       判断 end_date / extension_date
    └── 结束
```

### 5.3 bill_id 在两条链路中的流转

```
  用户表单/API (Web 或 JSON API)
      │
      ▼
  RecurrenceFormRequest / StoreRequest / UpdateRequest
  Web: transactions[0]['bill_id'] = 5 (顶级表单字段)
  API: transactions[*]['bill_id'] = 5 (嵌套字段)
      │
      ▼
  RecurringRepository::store() / update()
      │
      ▼
  RecurrenceFactory / RecurrenceUpdateService
    → RecurringTransactionTrait::setBill()
       → rt_meta 表:
         rt_id=?, name='bill_id', value='5' (string)
      │
      ▼
  ┌─────────────────────────────────────┐
  │  Recurrence 执行链路 (Cron)         │
  │                                     │
  │  CreateRecurringTransactions        │
  │    → RecurringRepository::getBillId()│
  │       读取 rt_meta bill_id='5'      │
  │    → TransactionGroupRepository     │
  │       → TransactionJournalFactory   │
  │          → transaction_journals     │
  │            bill_id = 5 (仅 WITHDRAWAL)│
  └──────────────┬──────────────────────┘
                 │
                 ▼
  ┌─────────────────────────────────────┐
  │  Bill 提醒链路 (Cron)               │
  │                                     │
  │  WarnAboutBills                     │
  │    → getDates()                     │
  │       Navigation::startOfPeriod(date, freq) │
  │       Navigation::endOfPeriod(start, freq) │
  │       → SubscriptionEnrichment      │
  │          → collectPaidDates()       │
  │            查询 transaction_journals│
  │            WHERE bill_id = 5        │
  │            AND date BETWEEN start AND end
  │            → 找到已付款记录         │
  │          → collectPayDates()        │
  │            BillDateCalculator       │
  │            基于 lastPaid 计算预期   │
  │    → 过期判断:                      │
  │       pay_dates - paid_dates = 0    │
  │       → 不触发过期提醒               │
  └─────────────────────────────────────┘
```

### 5.4 协作的关键保证

1. **bill_id 仅对 WITHDRAWAL 生效**：TransactionJournalFactory 强制非支出交易 bill_id 为 null，防止 Deposit/Transfer 误关联账单。

2. **按账单周期动态查询窗口**：每个账单的查询日期窗口由 `bill.repeat_freq` 动态计算，确保只检查当前周期的付款状态，性能更优且业务更准确。

3. **lastPaid 作为计算锚点**：BillDateCalculator 计算预期付款日时，以 `lastPaid`（最近一次已付款日期）为起点，保证已付款周期不会重复计算。

4. **两链路共用同一张表**：Recurrence 链路写入 `transaction_journals.bill_id`，Bill 链路读取同一字段——这是两条链路协作的唯一数据桥梁。

5. **Cron 执行顺序**：在 [Cron.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Console/Commands/Tools/Cron.php#L99-L129) 中，recurring 任务先执行，subscription warning 后执行。这确保了当天新创建的交易能被同一次 Cron 的账单提醒读取到。

6. **频率一致**：两个 Cronjob 都使用 12 小时最小间隔，保证二者执行节奏基本同步。

---

## 六、Bill 与 Recurrence 的协作关系（补充）

### 6.1 概念区分

```
┌─────────────────────────────────────────────────────────┐
│                     Bill（周期账单）                      │
│  作用：追踪预期支出是否按时支付                             │
│  关键字段：name, amount_min/max, repeat_freq, skip,       │
│           date, end_date, extension_date                  │
│  关联交易：transaction_journals.bill_id                   │
│  查询窗口：由 repeat_freq 动态计算当前周期                 │
└─────────────────────────────────────────────────────────┘
                         ▲
                         │ bill_id（可选关联）
                         │
┌─────────────────────────────────────────────────────────┐
│                  Recurrence（循环交易）                    │
│  作用：自动按周期创建交易                                  │
│  关键字段：title, first_date, repeat_until, latest_date   │
│  子模型：RecurrenceRepetition（重复规则）                  │
│          RecurrenceTransaction（交易模板）                 │
│            └→ RecurrenceTransactionMeta（含 bill_id）     │
└─────────────────────────────────────────────────────────┘
```

### 6.2 协作流程

1. 用户创建一个 Bill，定义预期支出（如"月租 ¥3000，每月 1 日"，`repeat_freq='monthly'`）
2. 用户创建一个 Recurrence，设置与 Bill 相同的周期，**并在交易模板中指定 `bill_id`**
3. Cron 执行时：
   - **Recurrence 链路**：按计划创建交易，交易自动关联到 Bill → Bill 被标记为该周期"已付"
   - **Bill Warning 链路**：按 Bill 的 `repeat_freq` 计算查询窗口（如本月），检查 `pay_dates` vs `paid_dates`，如果已付则不触发过期提醒

### 6.3 不使用 Recurrence 的情况

如果用户只创建 Bill 而不创建对应 Recurrence：
- Bill 仍然会计算预期付款日（`pay_dates`）
- 如果用户手动记录了关联该 Bill 的交易（`paid_dates`），过期检查会通过
- 如果没有手动记录，`needsOverdueAlert()` 会在 6 天后触发过期提醒

---

## 七、关键配置与偏好

### 7.1 系统配置

| 配置项 | 位置 | 值 | 说明 |
|--------|------|----|------|
| `bill_reminder_periods` | [config/firefly.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/config/firefly.php#L226) | `[90, 30, 14, 7, 0]` | 到期提醒触发天数 |
| `last_rt_job` | `configuration` 表 | 时间戳 | 循环交易 Job 上次执行时间 |
| `last_bw_job` | `configuration` 表 | 时间戳 | 账单提醒 Job 上次执行时间 |

### 7.2 用户偏好

| 偏好键 | 默认值 | 说明 |
|--------|--------|------|
| `notification_bill_reminder` | `true` | 是否接收账单提醒通知 |
| `notification_transaction_creation` | `false` | 是否接收循环交易创建报告 |
| `bill_overdue_{id}_{hash}` | `false` | 过期通知去重标记（已发送则设为 `true`） |

### 7.3 时间控制

| 参数 | 值 | 说明 |
|------|----|------|
| Cron 最小间隔 | 43,200 秒（12 小时） | 两次 Cron 执行间最小时间 |
| 过期判定天数 | 6 天 | 最早预期付款日距今 ≥ 6 天视为过期 |
| 发生日期搜索范围 | 今天 + 2 天 | 包含周末的缓冲 |
| 日期窗口容错 | -1 天 | collectPaidDates 查询向左扩展 1 天 |

---

## 八、Bill 模型关键字段

[Bill.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Models/Bill.php#L50-L212)

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 账单名称 |
| `amount_min` / `amount_max` | string | 预期金额范围 |
| `date` | Carbon | 账单起始日期 |
| `repeat_freq` | string | 重复频率（monthly/weekly/yearly 等）**→ 用于计算账单提醒查询窗口** |
| `skip` | int | 跳过周期数（0=每期，1=隔一期） |
| `active` | bool | 是否激活 |
| `end_date` | Carbon? | 账单终止日期 |
| `extension_date` | Carbon? | 账单续期日期 |
| `transaction_currency_id` | int | 货币 ID |

---

## 九、完整链路时序图

```
                        ┌──────────┐
                        │  Cron 命令 │
                        └────┬─────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              │              ▼
    ┌─────────────────┐     │     ┌─────────────────┐
    │ RecurringCronjob│     │     │BillWarningCronjob│
    └────────┬────────┘     │     └────────┬────────┘
             │              │              │
             ▼              │              ▼
  ┌──────────────────────┐  │  ┌──────────────────────┐
  │CreateRecurring       │  │  │WarnAboutBills        │
  │Transactions          │  │  │                      │
  │                      │  │  │  遍历 User → Bill    │
  │ 1.获取所有 Recurrence│  │  │                      │
  │ 2.过滤无效记录       │  │  │ ┌──────────────────┐ │
  │ 3.计算发生日期       │  │  │ │getDates()        │ │
  │ 4.当天则创建交易     │  │  │ │  startOfPeriod() │ │
  │   └→ rt_meta 读取   │  │  │ │  endOfPeriod()   │ │
  │      bill_id        │  │  │ │过期检查          │ │
  │   └→ TransactionGroup│ │  │ │Subscription      │ │
  │      Repository     │  │  │ │Enrichment        │ │
  │   └→ transaction_   │  │  │ │  → BillDate      │ │
  │      journals.bill_id│  │  │ │    Calculator    │ │
  │ 5.更新 latest_date   │  │  │ │  → pay_dates    │ │
  │                      │  │  │ │  → paid_dates   │ │
  └──────────┬───────────┘  │  │ └──────┬───────────┘ │
             │              │  │        │ ≥6天        │
             ▼              │  │        ▼              │
  ┌──────────────────────┐  │  │ ┌──────────────────┐ │
  │TransactionGroups     │  │  │ │SubscriptionsAre  │ │
  │RequestedReporting    │  │  │ │OverdueForPayment │ │
  │   (Event)            │  │  │ │   (Event)        │ │
  └──────────┬───────────┘  │  │ └──────┬───────────┘ │
             │              │  │        ▼              │
             ▼              │  │ ┌──────────────────┐ │
  ┌──────────────────────┐  │  │ │NotifiesAbout     │ │
  │MailsNewTransactions  │  │  │ │OverdueSubs       │ │
  │Report (Listener)     │  │  │ │  (Listener)      │ │
  │                      │  │  │ └──────┬───────────┘ │
  │ 检查偏好 → 发送通知   │  │  │        │              │
  └──────────────────────┘  │  │ ┌──────────────────┐ │
                            │  │ │到期/续期检查      │ │
                            │  │ │end_date          │ │
                            │  │ │extension_date    │ │
                            │  │ │  diff ∈ [90,30,  │ │
                            │  │ │   14,7,0]        │ │
                            │  │ └──────┬───────────┘ │
                            │  │        ▼              │
                            │  │ ┌──────────────────┐ │
                            │  │ │SubscriptionNeeds │ │
                            │  │ │ExtensionOrRenewal│ │
                            │  │ │   (Event)        │ │
                            │  │ └──────┬───────────┘ │
                            │  │        ▼              │
                            │  │ ┌──────────────────┐ │
                            │  │ │NotifiesAbout     │ │
                            │  │ │ExtensionOrRenewal│ │
                            │  │ │  (Listener)      │ │
                            │  │ └──────┬───────────┘ │
                            │  │        │              │
                            │  └────────┼──────────────┘
                            │           ▼
                            │  ┌──────────────────────┐
                            │  │ NotificationSender    │
                            │  │  → Mail / Slack /     │
                            │  │    Pushover           │
                            │  └──────────────────────┘
```

---

## 十、核心代码索引

### Recurrence 保存相关

| 文件 | 核心方法/类 | 说明 |
|------|------------|------|
| **Web 入口** | | |
| [CreateController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Http/Controllers/Recurring/CreateController.php) | `store()` | Web 创建 Recurrence 入口 |
| [EditController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Http/Controllers/Recurring/EditController.php) | `update()` | Web 更新 Recurrence 入口 |
| [RecurrenceFormRequest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Http/Requests/RecurrenceFormRequest.php) | `getAll()`, `rules()` | Web 请求数据提取（bill_id 是顶级字段） |
| **API 入口** | | |
| [StoreController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Api/V1/Controllers/Models/Recurrence/StoreController.php) | `store()` | API v1 创建 Recurrence 入口 |
| [UpdateController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Api/V1/Controllers/Models/Recurrence/UpdateController.php) | `update()` | API v1 更新 Recurrence 入口 |
| [StoreRequest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Api/V1/Requests/Models/Recurrence/StoreRequest.php) | `getAll()` | API 创建请求数据提取 |
| [UpdateRequest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Api/V1/Requests/Models/Recurrence/UpdateRequest.php) | `getAll()` | API 更新请求数据提取 |
| **共用 Trait** | | |
| [GetRecurrenceData.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/Request/GetRecurrenceData.php) | `getSingleTransactionData()` | API 请求提取 bill_id（intKeys 包含 bill_id） |
| **业务层** | | |
| [RecurringRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/Recurring/RecurringRepository.php) | `store()`, `update()` | Web/API 共用的统一保存入口 |
| [RecurrenceFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Factory/RecurrenceFactory.php) | `create()` | 创建 Recurrence 主流程 |
| [RecurrenceUpdateService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Services/Internal/Update/RecurrenceUpdateService.php) | `update()`, `updateCombination()` | 更新 Recurrence |
| [RecurringTransactionTrait.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Services/Internal/Support/RecurringTransactionTrait.php) | `setBill()`, `createTransactions()` | bill_id 写入 rt_meta |

### Recurrence 执行相关

| 文件 | 核心方法/类 | 说明 |
|------|------------|------|
| [CreateRecurringTransactions.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/CreateRecurringTransactions.php) | `handle()`, `getTransactionData()`, `handleOccurrence()` | 循环交易 Job |
| [RecurringRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/Recurring/RecurringRepository.php) | `getBillId()`, `getAll()`, `getOccurrencesInRange()` | bill_id 读取 + 发生日期计算 |
| [TransactionGroupRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php) | `store()` | 交易保存入口 |
| [TransactionJournalFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Factory/TransactionJournalFactory.php) | `createJournal()` | bill_id 写入 transaction_journals（仅 WITHDRAWAL） |

### Bill 提醒相关

| 文件 | 核心方法/类 | 说明 |
|------|------------|------|
| [WarnAboutBills.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/WarnAboutBills.php) | `handle()`, `getDates()`, `needsOverdueAlert()` | 账单提醒 Job |
| **日期窗口计算** | | |
| [Navigation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/Navigation.php) | `startOfPeriod()`, `endOfPeriod()` | 按账单 repeat_freq 计算查询窗口的起止日期 |
| **已付款查询** | | |
| [SubscriptionEnrichment.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/JsonApi/Enrichments/SubscriptionEnrichment.php) | `collectPaidDates()`, `collectPayDates()` | 按周期窗口查询已付款 + 计算预期付款日 |
| [BillDateCalculator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/Models/BillDateCalculator.php) | `getPayDates()`, `nextDateMatch()` | 日期计算引擎 |
| [BillRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/Bill/BillRepository.php) | `getPaidDatesInRange()` | 已付款日期通用查询 |
| [Bill.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Models/Bill.php) | `transactionJournals()` | Bill 与 TransactionJournal 的 HasMany 关系 |

