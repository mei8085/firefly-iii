# 附件与交易记录的关联关系分析

## 1. 数据库层：多态关联设计

### 1.1 attachments 表结构

定义于 [createAttachmentsTable](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/database/migrations/2016_06_16_000002_create_main_tables.php#L134-L162)：

```php
$table->increments('id');
$table->timestamps();
$table->softDeletes();
$table->integer('user_id', false, true);
$table->integer('attachable_id', false, true);     // 多态外键
$table->string('attachable_type', 255);             // 多态类型
$table->string('md5', 128);
$table->string('filename', 1024);
$table->string('title', 1024)->nullable();
$table->text('description')->nullable();
$table->text('notes')->nullable();
$table->string('mime', 1024);
$table->integer('size', false, true);
$table->boolean('uploaded')->default(1);
$table->foreign('user_id')->references('id')->on('users')->onDelete('cascade');
```

核心设计：使用 Laravel 的 **Polymorphic Morph** 模式，`attachable_id` + `attachable_type` 两个字段组合指向上级实体。

### 1.2 合法的 attachable_type 列表

定义于 [firefly.php#L213-L223](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/config/firefly.php#L213-L223)：

```php
'valid_attachment_models' => [
    Account::class,
    Bill::class,
    Budget::class,
    Category::class,
    PiggyBank::class,
    Tag::class,
    Transaction::class,
    TransactionJournal::class,
    Recurrence::class,
],
```

共 **9 种**实体可以拥有附件。其中与交易直接相关的是 `Transaction` 和 `TransactionJournal`。

## 2. 模型层：MorphTo / MorphMany 关系声明

### 2.1 Attachment 模型侧（反向）

[Attachment::attachable](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Attachment.php#L89-L92)：

```php
public function attachable(): MorphTo
{
    return $this->morphTo();
}
```

### 2.2 各实体模型侧（正向）

所有拥有附件能力的模型都声明了 `attachments(): MorphMany`，示例如下：

| 模型 | 代码位置 | 关系声明 |
|---|---|---|
| TransactionJournal | [TransactionJournal::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/TransactionJournal.php#L117-L120) | `$this->morphMany(Attachment::class, 'attachable')` |
| Account | [Account::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Account.php#L98) | `$this->morphMany(Attachment::class, 'attachable')` |
| Bill | [Bill::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Bill.php#L106) | `$this->morphMany(Attachment::class, 'attachable')` |
| Budget | [Budget::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Budget.php#L74) | `$this->morphMany(Attachment::class, 'attachable')` |
| Category | [Category::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Category.php#L76) | `$this->morphMany(Attachment::class, 'attachable')` |
| PiggyBank | [PiggyBank::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/PiggyBank.php#L100) | `$this->morphMany(Attachment::class, 'attachable')` |
| Tag | [Tag::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Tag.php#L80) | `$this->morphMany(Attachment::class, 'attachable')` |
| Recurrence | [Recurrence::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Recurrence.php#L100) | `$this->morphMany(Attachment::class, 'attachable')` |

> **注意**：`Transaction` 模型 **没有** 声明 `attachments()` 关系方法，尽管它是合法的 `attachable_type`。

## 3. 关键不一致点：Transaction vs TransactionJournal

### 3.1 三层交易模型结构

```
TransactionGroup (交易组)
  └── TransactionJournal (交易日志，一种交易类型)
        └── Transaction (交易记录，一进一出的两条)
```

- 一个 **TransactionGroup** 可包含多个 **TransactionJournal**（拆分交易场景）
- 一个 **TransactionJournal** 恰好包含两个 **Transaction**（一借一贷）

### 3.2 附件归挂在 Journal 而非 Transaction

**数据库中的实际归属**：在 `AttachmentFactory::create` 中存在关键逻辑——

[AttachmentFactory::create](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Factory/AttachmentFactory.php#L44-L84)：

```php
public function create(array $data): ?Attachment
{
    $model = str_contains((string) $data['attachable_type'], 'FireflyIII')
        ? $data['attachable_type']
        : sprintf('FireflyIII\Models\%s', $data['attachable_type']);

    // ★ 如果传入 Transaction，自动转换到 TransactionJournal
    if (Transaction::class === $model) {
        $transaction = $this->user->transactions()->find((int) $data['attachable_id']);
        $data['attachable_id'] = $transaction->transaction_journal_id;
        $model = TransactionJournal::class;
    }

    $attachment = Attachment::create([
        'attachable_type' => $model,  // 最终存储的是 TransactionJournal
        'attachable_id'   => $data['attachable_id'],
        // ...
    ]);
}
```

**结论**：即使 API 传入 `attachable_type = Transaction`，后端也会将其**自动提升**为 `TransactionJournal`。数据库中不会存在 `attachable_type = Transaction` 的附件记录。

### 3.3 查询侧同样只关联 Journal

[TransactionGroupRepository::getAttachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php#L109-L136)：

```php
$set = Attachment::whereIn('attachable_id', $journals)
    ->where('attachable_type', TransactionJournal::class)  // 硬编码为 TransactionJournal
    ->where('uploaded', true)
    ->whereNull('deleted_at')
    ->get();
```

[AttachmentCollection::joinAttachmentTables](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Helpers/Collector/Extensions/AttachmentCollection.php#L510-L529)：

```php
$this->query
    ->leftJoin('attachments', 'attachments.attachable_id', '=', 'transaction_journals.id')
    ->where(static function (EloquentBuilder $q1): void {
        $q1->where('attachments.attachable_type', TransactionJournal::class);
        $q1->orWhereNull('attachments.attachable_type');
    });
```

[TransactionGroupEnrichment](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Support/JsonApi/Enrichments/TransactionGroupEnrichment.php#L169-L179)：

```php
$attachments = Attachment::query()
    ->whereIn('attachable_id', $this->journalIds)
    ->where('attachable_type', TransactionJournal::class)
    ->groupBy('attachable_id')
    ->get(['attachable_id', DB::raw('COUNT(id) as nr_of_attachments')]);
```

## 4. 上传入口与附件归属的协作方式

### 4.1 两套上传路径

| 路径 | 适用场景 | 附件归挂时机 | 归挂对象 |
|---|---|---|---|
| **Web UI（表单上传）** | Account / Bill / Budget / Category / PiggyBank / Tag / Recurrence 的创建和编辑 | 实体创建**之后**立即保存 | 通过 `saveAttachmentsForModel($model, $files)` 直接绑定到该模型 |
| **API（两步上传）** | Transaction（交易）及所有其他实体 | 交易创建**之后**，前端拿到 `transaction_journal_id` 后再上传 | 前端显式传入 `attachable_type=TransactionJournal` + `attachable_id=journal_id` |

### 4.2 Web UI 路径详解

Web UI 中所有非交易实体的附件上传遵循相同模式，以 Bill 为例：

[Bill\CreateController](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Controllers/Bill/CreateController.php#L112)：

```php
$recurrence = $this->repository->store($data);
$files = $request->hasFile('attachments') ? $request->file('attachments') : null;
$this->attachments->saveAttachmentsForModel($bill, $files);
```

调用链：`saveAttachmentsForModel()` → `processFile()` → 创建 Attachment 记录 + 写入磁盘文件。

**此路径下附件直接关联到传入的 Model 实例**，`attachable_type` 就是该模型的 FQCN。

### 4.3 API 路径详解（交易场景）

交易创建流程分两步：

**第一步**：创建交易（不涉及附件）

API: `POST /api/v1/transactions` → [Transaction\StoreController::store](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Transaction/StoreController.php#L85-L143)

返回结果包含每个 `transaction_journal_id`。

**第二步**：前端拿到 journal ID 后上传附件

API: `POST /api/v1/attachments` → [Attachment\StoreController::store](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/StoreController.php#L74-L92)

前端请求体：
```json
{
  "filename": "receipt.pdf",
  "attachable_type": "TransactionJournal",
  "attachable_id": 42
}
```

**第三步**：上传文件内容

API: `POST /api/v1/attachments/{id}/upload` → [Attachment\StoreController::upload](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/StoreController.php#L97-L121)

### 4.4 前端 JS 处理逻辑

[v1 CreateTransaction.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v1/src/components/transactions/CreateTransaction.vue#L561-L648)：

```javascript
collectAttachmentData(response) {
    let groupId = response.data.data.id;
    response.data.data.attributes.transactions = response.data.data.attributes.transactions.reverse();
    let attachments = $('input[name="attachments[]"]');
    for (const key in attachments) {
        // ★ 将文件与对应的 transaction_journal_id 关联
        toBeUploaded.push({
            journal: response.data.data.attributes.transactions[key].transaction_journal_id,
            file: attachments[key].files[fileKey]
        });
    }
}

uploadFiles(fileData, groupId, transactionData) {
    const data = {
        filename: fileData[key].name,
        attachable_type: 'TransactionJournal',   // ★ 硬编码为 TransactionJournal
        attachable_id: fileData[key].journal,
    };
    axios.post('./api/v1/attachments', data)
        .then(response => {
            axios.post('./api/v1/attachments/' + response.data.data.id + '/upload', fileData[key].content);
        });
}
```

[v2 process-attachments.js](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v2/src/pages/transactions/shared/process-attachments.js#L61-L114)：

```javascript
export function processAttachments(groupId, transactions) {
    transactions = transactions.reverse();
    let attachments = document.querySelectorAll('input[name="attachments[]"]');
    for (const key in attachments) {
        toBeUploaded.push({
            journal: transactions[key].transaction_journal_id,  // ★ 按 key 索引与 journal 对应
            file: attachments[key].files[fileKey]
        });
    }
}

// 上传时也是硬编码 TransactionJournal
poster.post(fileData[key].name, 'TransactionJournal', fileData[key].journal);
```

## 5. 不同交易类型下的不一致性分析

### 5.1 交易类型的层级关系

Firefly III 中交易类型（Withdrawal / Deposit / Transfer）通过以下层级表达：

```
TransactionType (枚举值)
  → TransactionJournal.transaction_type_id (外键指向 TransactionType)
    → TransactionJournal 属于 TransactionGroup
```

**所有交易类型共用相同的附件归挂逻辑**——附件始终挂在 `TransactionJournal` 上，与交易类型无关。

### 5.2 TransactionGroup 没有 attachments 关系

`TransactionGroup` 模型**没有**定义 `attachments()` 方法。API 获取交易组附件时，需遍历其下所有 Journal：

[Transaction\ListController::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Transaction/ListController.php#L70-L94)：

```php
foreach ($transactionGroup->transactionJournals as $transactionJournal) {
    $collection = $this->journalAPIRepository->getAttachments($transactionJournal)->merge($collection);
}
```

### 5.3 Recurrence（定期交易）的附件归属差异

Recurrence 的附件直接挂在 **Recurrence** 模型上（`attachable_type = Recurrence`），而不是挂在某个 Journal 上。这是因为 Recurrence 创建时尚未生成实际的 TransactionJournal，后者是在 Recurrence 被触发执行时才创建的。

**关键问题**：Recurrence 触发生成 TransactionJournal 时，**不会复制附件**到新生成的 Journal 上。

[GroupCloneService::cloneJournal](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Update/GroupCloneService.php#L67-L120) 中克隆 Journal 时会复制 notes、meta、categories、budgets、tags，但 **没有克隆 attachments**。

### 5.4 各实体 Observer 对附件的清理

当实体被删除时，对应的附件通过 Observer 自动清理：

| Observer | 代码位置 |
|---|---|
| DeletedTransactionJournalObserver | [L69-L71](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedTransactionJournalObserver.php#L69-L71) |
| DeletedAccountObserver | [L47-L49](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedAccountObserver.php#L47-L49) |
| DeletedCategoryObserver | [L43-L45](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedCategoryObserver.php#L43-L45) |
| DeletedRecurrenceObserver | [L43-L45](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedRecurrenceObserver.php#L43-L45) |
| DeletedTagObserver | [L43-L45](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedTagObserver.php#L43-L45) |
| PiggyBankObserver | [L55-L57](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/PiggyBankObserver.php#L55-L57) |

**注意**：`DeletedTransactionGroupObserver` **不会**清理附件，因为附件不属于 Group。附件通过 `DeletedTransactionJournalObserver` 随 Journal 删除而级联删除。

## 6. 验证层：IsValidAttachmentModel

[IsValidAttachmentModel](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Rules/IsValidAttachmentModel.php#L49-L87) 对 API 请求中的 `attachable_type` + `attachable_id` 进行验证：

```php
$result = match ($this->model) {
    Account::class            => $this->validateAccount((int) $value),
    Bill::class               => $this->validateBill((int) $value),
    Budget::class             => $this->validateBudget((int) $value),
    Category::class           => $this->validateCategory((int) $value),
    PiggyBank::class          => $this->validatePiggyBank((int) $value),
    Tag::class                => $this->validateTag((int) $value),
    Transaction::class        => $this->validateTransaction((int) $value),
    TransactionJournal::class => $this->validateJournal((int) $value),
    default                   => false
};
```

API StoreRequest ([StoreRequest::rules](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Requests/Models/Attachment/StoreRequest.php#L59-L73)) 同时验证 `attachable_type` 必须在 `valid_attachment_models` 列表内。

## 7. 不一致问题总结

### 7.1 Transaction 作为合法 attachable_type 但实际不存在

- 配置中 `Transaction::class` 是合法的 `attachable_type`
- `IsValidAttachmentModel` 能验证 Transaction
- 但 `AttachmentFactory::create` 会将 Transaction **自动转换** 为 TransactionJournal
- `Transaction` 模型没有声明 `attachments()` 关系方法
- **结果**：数据库中永远不会出现 `attachable_type = Transaction` 的记录，配置声明与实际行为不一致

### 7.2 Recurrence 的附件不会传递到生成的交易

- Recurrence 的附件归挂在 Recurrence 自身
- Recurrence 触发生成 TransactionJournal 时，附件不会自动复制
- 用户需要分别为 Recurrence 和生成的交易添加附件

### 7.3 克隆交易不复制附件

- [GroupCloneService](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Update/GroupCloneService.php#L67-L120) 克隆 Journal 时复制了 notes、meta、categories、budgets、tags，但遗漏了 attachments

### 7.4 Web UI 与 API 的附件上传时机差异

- Web UI 非交易实体：在实体创建的同一请求中同步上传附件
- Web UI 交易：前端先创建交易，再异步调用 API 上传附件
- API 交易：始终是两步操作（先 POST 交易 → 再 POST 附件 + 上传内容）

### 7.5 AttachmentCollection 的 LEFT JOIN 可能产生笛卡尔积

[AttachmentCollection::joinAttachmentTables](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Helpers/Collector/Extensions/AttachmentCollection.php#L510-L529) 使用 `leftJoin` 将 attachments 表连接到 transaction_journals，如果一个 Journal 有多个附件，查询结果会产生多行，需要后续在 PHP 层面进行聚合处理。

## 8. 完整调用链图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        附件上传入口总览                                   │
├──────────────────┬──────────────────────┬───────────────────────────────┤
│ 入口             │ 归挂目标              │ 调用路径                      │
├──────────────────┼──────────────────────┼───────────────────────────────┤
│ Web 创建/编辑    │ Account              │ Controller                    │
│ Account          │                      │ → saveAttachmentsForModel()   │
├──────────────────┼──────────────────────┼───────────────────────────────┤
│ Web 创建/编辑    │ Bill                 │ Controller                    │
│ Bill             │                      │ → saveAttachmentsForModel()   │
├──────────────────┼──────────────────────┼───────────────────────────────┤
│ Web 创建/编辑    │ Budget               │ Controller                    │
│ Budget           │                      │ → saveAttachmentsForModel()   │
├──────────────────┼──────────────────────┼───────────────────────────────┤
│ Web 创建/编辑    │ Category             │ Controller                    │
│ Category         │                      │ → saveAttachmentsForModel()   │
├──────────────────┼──────────────────────┼───────────────────────────────┤
│ Web 创建/编辑    │ PiggyBank            │ Controller                    │
│ PiggyBank        │                      │ → saveAttachmentsForModel()   │
├──────────────────┼──────────────────────┼───────────────────────────────┤
│ Web 创建/编辑    │ Tag                  │ Controller                    │
│ Tag              │                      │ → saveAttachmentsForModel()   │
├──────────────────┼──────────────────────┼───────────────────────────────┤
│ Web 创建/编辑    │ Recurrence           │ Controller                    │
│ Recurrence       │                      │ → saveAttachmentsForModel()   │
├──────────────────┼──────────────────────┼───────────────────────────────┤
│ Web 创建交易     │ TransactionJournal   │ JS collectAttachmentData()    │
│ (v1 Vue)         │                      │ → axios POST /api/v1/attach.. │
│                  │                      │ → axios POST /upload          │
├──────────────────┼──────────────────────┼───────────────────────────────┤
│ Web 创建交易     │ TransactionJournal   │ JS processAttachments()       │
│ (v2)             │                      │ → AttachmentPost.post()       │
│                  │                      │ → AttachmentPost.upload()     │
├──────────────────┼──────────────────────┼───────────────────────────────┤
│ API 直接创建     │ 任意合法模型          │ POST /api/v1/attachments      │
│ 附件             │ (含Transaction)      │ → AttachmentFactory::create() │
│                  │                      │ → Transaction 自动转为 Journal│
│                  │                      │ POST /api/v1/attachments/{id} │
│                  │                      │ /upload                       │
└──────────────────┴──────────────────────┴───────────────────────────────┘
```

## 9. 核心文件索引

| 文件 | 职责 |
|---|---|
| [Attachment.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Attachment.php) | 附件模型，定义 MorphTo 关系 |
| [AttachmentFactory.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Factory/AttachmentFactory.php) | 工厂类，含 Transaction→Journal 转换逻辑 |
| [AttachmentHelper.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Helpers/Attachments/AttachmentHelper.php) | Web UI 附件上传核心，含 saveAttachmentsForModel / saveAttachmentFromApi |
| [AttachmentRepository.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/Attachment/AttachmentRepository.php) | 附件仓库，store/update/destroy |
| [AttachmentController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Controllers/AttachmentController.php) | Web UI 附件管理（编辑/删除/下载/查看） |
| [Attachment\StoreController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/StoreController.php) | API 附件创建 + 上传 |
| [Attachment\StoreRequest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Requests/Models/Attachment/StoreRequest.php) | API 附件创建请求验证 |
| [IsValidAttachmentModel.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Rules/IsValidAttachmentModel.php) | 验证 attachable_type + attachable_id 的合法性 |
| [AttachmentCollection.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Helpers/Collector/Extensions/AttachmentCollection.php) | 交易查询时 LEFT JOIN 附件表 |
| [TransactionGroupRepository.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php) | 交易组仓库，getAttachments / countAttachments |
| [TransactionGroupEnrichment.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Support/JsonApi/Enrichments/TransactionGroupEnrichment.php) | 交易组数据富化，统计每个 Journal 的附件数量 |
| [GroupCloneService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Update/GroupCloneService.php) | 交易克隆服务（不复制附件） |
| [process-attachments.js](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v2/src/pages/transactions/shared/process-attachments.js) | v2 前端附件上传处理 |
| [CreateTransaction.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v1/src/components/transactions/CreateTransaction.vue) | v1 前端交易创建（含附件上传） |
| [firefly.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/config/firefly.php#L213-L223) | valid_attachment_models 配置 |
