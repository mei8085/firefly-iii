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

核心设计：使用 Laravel 的 **Polymorphic Morph** 模式，`attachable_id` + `attachable_type` 两个字段组合指向上级实体。注意 `attachable_type` 存储的是模型 FQCN（如 `FireflyIII\Models\TransactionJournal`），不是短名。

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

**关键事实**：`Transaction` 模型 **没有** 声明 `attachments()` 关系方法，尽管它是合法的 `attachable_type`。`TransactionGroup` 也没有声明。

## 3. 三层交易模型与附件归属

### 3.1 模型层级

```
TransactionGroup (交易组，可包含多笔拆分交易)
  └── TransactionJournal (交易日志，一笔交易，含类型信息)
        └── Transaction (交易记录，一借一贷两条)
```

- 一个 **TransactionGroup** 可包含多个 **TransactionJournal**（拆分交易场景）
- 一个 **TransactionJournal** 恰好包含两个 **Transaction**（一借一贷）
- `TransactionGroup` 和 `Transaction` 都**没有** `attachments()` 关系

### 3.2 Transaction 与 TransactionJournal 的实际落库关系

**结论先行**：在所有入口下，附件的 `attachable_type` 最终都存储为 `TransactionJournal`，不会存储为 `Transaction`。但这一结论的原因因入口而异，需要区分分析：

#### 入口 A：Web UI 交易创建（v1 / v2）

前端始终硬编码传入 `attachable_type: 'TransactionJournal'`，后端 `AttachmentFactory::create` 不会触发 Transaction→Journal 转换，直接写入 `TransactionJournal`。

证据：
- [v1 CreateTransaction.vue#L633](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v1/src/components/transactions/CreateTransaction.vue#L633)：`attachable_type: 'TransactionJournal'`
- [v2 process-attachments.js#L31](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v2/src/pages/transactions/shared/process-attachments.js#L31)：`poster.post(fileData[key].name, 'TransactionJournal', fileData[key].journal)`

#### 入口 B：API 直接创建附件 `POST /api/v1/attachments`

调用链：`StoreController::store` → `AttachmentRepository::store` → `AttachmentFactory::create`

[AttachmentFactory::create](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Factory/AttachmentFactory.php#L44-L84) 中存在 **Transaction→Journal 自动提升逻辑**：

```php
$model = str_contains((string) $data['attachable_type'], 'FireflyIII')
    ? $data['attachable_type']
    : sprintf('FireflyIII\Models\%s', $data['attachable_type']);

if (Transaction::class === $model) {
    $transaction = $this->user->transactions()->find((int) $data['attachable_id']);
    $data['attachable_id'] = $transaction->transaction_journal_id;
    $model = TransactionJournal::class;
}

$attachment = Attachment::create([
    'attachable_type' => $model,  // 最终存储 TransactionJournal
    'attachable_id'   => $data['attachable_id'],
]);
```

如果 API 用户传入 `attachable_type=Transaction` + `attachable_id=123`，后端会：
1. 查找 Transaction#123
2. 取其 `transaction_journal_id`
3. 将 `attachable_type` 改为 `TransactionJournal::class`，`attachable_id` 改为 journal ID
4. 写入数据库

**结果**：数据库中**不会出现** `attachable_type = FireflyIII\Models\Transaction` 的记录。

#### 入口 C：Web UI 非交易实体（Account/Bill 等）

调用链：Controller → `AttachmentHelper::saveAttachmentsForModel`

[AttachmentHelper::processFile](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Helpers/Attachments/AttachmentHelper.php#L263-L310)：

```php
$attachment = new Attachment();
$attachment->user()->associate($user);
$attachment->attachable()->associate($model);  // 直接用 Morph 关联
$attachment->save();
```

此路径通过 Eloquent 的 `associate()` 方法设置 `attachable_type` 和 `attachable_id`，取的是模型的 FQCN。由于不会传入 `Transaction` 实例（交易走的是前端 API 路径），因此此处也不会产生 `Transaction` 类型的附件记录。

#### 入口 D：附件更新 `PUT /api/v1/attachments/{id}`

[AttachmentRepository::update](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/Attachment/AttachmentRepository.php#L132-L155)：

```php
if (array_key_exists('attachable_type', $data) && array_key_exists('attachable_id', $data)) {
    $attachment->attachable_id   = (int) $data['attachable_id'];
    $attachment->attachable_type = sprintf('FireflyIII\Models\%s', $data['attachable_type']);
}
```

**关键差异**：`AttachmentRepository::update` **没有** AttachmentFactory 中 Transaction→Journal 的自动提升逻辑！如果通过 API 更新附件时传入 `attachable_type=Transaction`，它会被原样存储为 `FireflyIII\Models\Transaction`，**不会**被转换为 TransactionJournal。

这是创建入口与更新入口之间的**本质差异**。

#### 落库关系总结

| 入口 | 传入 Transaction 类型时 | 最终落库 attachable_type |
|---|---|---|
| API 创建附件 (AttachmentFactory) | 自动提升为 Journal | `TransactionJournal` |
| Web UI 交易创建/编辑 | 前端硬编码 Journal | `TransactionJournal` |
| Web UI 非交易实体 | 不涉及 Transaction | 各实体 FQCN |
| **API 更新附件 (AttachmentRepository)** | **不做转换，原样存储** | **可能为 `Transaction`** |

## 4. 三类交易入口的附件处理

### 4.1 交易创建入口

#### Web UI v1

[CreateTransaction.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v1/src/components/transactions/CreateTransaction.vue#L565-L648)

流程：
1. 前端提交交易数据 → `POST /api/v1/transactions`
2. 后端返回交易组，包含每个 split 的 `transaction_journal_id`
3. 前端 `collectAttachmentData()` 将文件与 journal ID 对应（按索引 key 匹配）
4. 前端异步调用 `POST /api/v1/attachments`，`attachable_type: 'TransactionJournal'`
5. 再调用 `POST /api/v1/attachments/{id}/upload` 上传文件内容

#### Web UI v2

[edit.js#L125-L190](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v2/src/pages/transactions/edit.js#L125-L190) 和 [process-attachments.js](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v2/src/pages/transactions/shared/process-attachments.js#L61-L114)

流程与 v1 相同，但使用 Alpine.js + 模块化 API 客户端（`AttachmentPost`）。

**关键细节**：`processAttachments()` 接收的 `transactions` 数组来自后端 API 响应中的 `group.attributes.transactions`，每个元素都有 `transaction_journal_id`。前端文件输入框 `input[name="attachments[]"]` 与 transactions 数组**按索引 key 一一对应**。

#### API 创建交易

[Transaction\StoreController::store](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Transaction/StoreController.php#L85-L143)

后端只负责创建交易，**不处理附件**。API 用户需单独调用附件 API 创建并上传附件。

### 4.2 交易编辑入口

#### Web UI v1

[EditTransaction.vue#L874-L992](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v1/src/components/transactions/EditTransaction.vue#L874-L992)

编辑页的附件上传逻辑与创建页**完全一致**：
1. 用户在表单中选择新附件文件
2. 提交交易更新 → `PUT /api/v1/transactions/{id}`
3. 后端返回更新后的交易组（含 `transaction_journal_id`）
4. 前端 `collectAttachmentData()` 按索引匹配，调用附件 API 上传

**编辑时只能添加新附件，不能通过交易编辑表单删除已有附件。** 已有附件的删除需通过附件管理页面单独操作。

#### Web UI v2

[edit.js#L157-L167](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v2/src/pages/transactions/edit.js#L157-L167)：

```javascript
putter.put(submission, {id: this.groupProperties.id}).then((response) => {
    const group = response.data.data;
    this.groupProperties.id = parseInt(group.id);
    const attachmentCount = processAttachments(this.groupProperties.id, group.attributes.transactions);
    // ...
});
```

与创建逻辑复用同一个 `processAttachments()` 函数，同样硬编码 `TransactionJournal`。

#### Web UI EditController（后端）

[Transaction\EditController::edit](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Controllers/Transaction/EditController.php#L77-L142)

后端 EditController **不处理附件上传**，只渲染编辑视图。附件上传全部由前端 JS 异步完成。Controller 中没有 `AttachmentHelper` 或 `saveAttachmentsForModel` 调用。

### 4.3 API 更新交易入口

[Transaction\UpdateController::update](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Transaction/UpdateController.php#L76-L131)

调用链：`UpdateController::update` → `TransactionGroupRepository::update` → `GroupUpdateService::update`

[GroupUpdateService::update](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Update/GroupUpdateService.php#L48-L115) 的核心逻辑：

1. 如果只有单笔交易（1 group = 1 journal），调用 `JournalUpdateService` 更新
2. 如果是拆分交易，遍历提交的 transactions 数组：
   - 有 `transaction_journal_id` 的 → 调用 `JournalUpdateService` 更新
   - 没有 journal ID 的 → 通过 `TransactionJournalFactory` 创建新 journal
   - 原有但未出现在提交数据中的 journal → 通过 `JournalDestroyService` **删除**

[JournalUpdateService](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Update/JournalUpdateService.php) **不处理附件**——整个类中没有 attachment/attachable 相关代码。

**API 更新交易时不涉及任何附件操作**——既不添加、不删除、也不迁移附件。

### 4.4 拆分交易更新时的附件丢失风险

当 API 更新拆分交易时，[GroupUpdateService::update](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Update/GroupUpdateService.php#L84-L108) 可能删除原有的 journal：

```php
$result = array_diff($existing, $updated);
foreach ($result as $deletedId) {
    $journal = $transactionGroup->transactionJournals()->find((int) $deletedId);
    $service = app(JournalDestroyService::class);
    $service->destroy($journal);
}
```

[JournalDestroyService::destroy](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Destroy/JournalDestroyService.php#L35-L49) 调用 `$journal->delete()`，触发 Observer：

[DeletedTransactionJournalObserver::deleting](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedTransactionJournalObserver.php#L38-L81)：

```php
foreach ($transactionJournal->attachments()->get() as $attachment) {
    $repository->destroy($attachment);
}
```

**结果**：如果拆分交易更新导致某个 Journal 被删除，该 Journal 上的所有附件也会被级联删除，且无法恢复。

## 5. 附件更新与归属迁移入口

### 5.1 API 更新附件 `PUT /api/v1/attachments/{id}`

[Attachment\UpdateController::update](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/UpdateController.php#L69-L86)

调用链：`UpdateController::update` → `AttachmentRepository::update`

[AttachmentRepository::update](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/Attachment/AttachmentRepository.php#L132-L155)：

```php
public function update(Attachment $attachment, array $data): Attachment
{
    // 更新 title
    if (array_key_exists('title', $data)) {
        $attachment->title = $data['title'];
    }
    // 更新 filename（仅显示名，不影响磁盘文件）
    if (array_key_exists('filename', $data) && '' !== (string) $data['filename'] && $data['filename'] !== $attachment->filename) {
        $attachment->filename = $data['filename'];
    }
    // ★ 迁移归属（move attachment）
    if (array_key_exists('attachable_type', $data) && array_key_exists('attachable_id', $data)) {
        $attachment->attachable_id   = (int) $data['attachable_id'];
        $attachment->attachable_type = sprintf('FireflyIII\Models\%s', $data['attachable_type']);
    }

    $attachment->save();
    $attachment->refresh();
    if (array_key_exists('notes', $data)) {
        $this->updateNote($attachment, (string) $data['notes']);
    }
    return $attachment;
}
```

**此方法可以将附件从一个实体迁移到另一个实体**，包括跨类型迁移（如从 Bill 迁移到 TransactionJournal）。

### 5.2 更新入口与创建入口的关键差异

| 维度 | 创建入口（AttachmentFactory::create） | 更新入口（AttachmentRepository::update） |
|---|---|---|
| Transaction→Journal 自动提升 | **有**：自动转换 `attachable_type=Transaction` → `TransactionJournal` | **没有**：原样存储，可写入 `FireflyIII\Models\Transaction` |
| attachable_type 格式化 | 自动补全命名空间（`FireflyIII\Models\`） | 同样补全：`sprintf('FireflyIII\Models\%s', ...)` |
| 验证 | StoreRequest 校验 `attachable_type` 必须在 `valid_attachment_models` 内 + `IsValidAttachmentModel` 校验 ID 有效性 | UpdateRequest 校验 `attachable_type` 必须在 `valid_attachment_models` 内 + `IsValidAttachmentModel` 校验 ID 有效性 |
| 磁盘文件处理 | 创建后需单独调用 upload 接口写入 | 更新只改元数据，**不涉及磁盘文件重命名或移动** |

### 5.3 更新入口的 Transaction 落库漏洞

由于 `AttachmentRepository::update` 缺少 Transaction→Journal 的自动提升逻辑，以下场景可以导致数据库中出现 `attachable_type = FireflyIII\Models\Transaction` 的记录：

```
PUT /api/v1/attachments/123
{
  "attachable_type": "Transaction",
  "attachable_id": 456
}
```

验证层 `IsValidAttachmentModel` 会通过（因为 `Transaction` 是合法模型），`AttachmentRepository::update` 会将其存为 `FireflyIII\Models\Transaction`。

由于 `Transaction` 模型没有 `attachments()` 关系方法，这样的附件将成为**孤儿记录**：
- 无法通过 `$transaction->attachments` 查到
- 无法通过交易编辑页面管理
- 只有通过 `GET /api/v1/attachments/{id}` 单独访问才能看到
- 在 [TransactionGroupRepository::getAttachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php#L109-L136) 中查不到（硬编码 `attachable_type = TransactionJournal::class`）

### 5.4 Web UI 附件编辑页面

[AttachmentController::update](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Controllers/AttachmentController.php#L166-L183)

Web UI 的附件编辑只允许修改 `title` 和 `notes`，**不允许修改 `attachable_type` 和 `attachable_id`**。

证据：[AttachmentFormRequest::getAttachmentData](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Requests/AttachmentFormRequest.php#L45-L48)：

```php
public function getAttachmentData(): array
{
    return ['title' => $this->convertString('title'), 'notes' => $this->convertString('notes')];
}
```

**归属迁移只能通过 API 完成**，Web UI 不提供此功能。

## 6. 查询侧对 Transaction 类型的排除

所有查询附件的代码都硬编码为 `TransactionJournal::class`，不会查到 `Transaction` 类型的附件：

[TransactionGroupRepository::getAttachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php#L109-L136)：
```php
$set = Attachment::whereIn('attachable_id', $journals)
    ->where('attachable_type', TransactionJournal::class)
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

[JournalAPIRepository::getAttachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/Journal/JournalAPIRepository.php#L61-L74)：
```php
$set = $journal->attachments;  // 通过 MorphMany 关系加载
```

**结论**：如果通过 API 更新入口写入了 `attachable_type = Transaction` 的记录，它在上述所有查询中都会被过滤掉，成为隐形孤儿。

## 7. 附件删除与级联清理

### 7.1 附件自身的删除

API: `DELETE /api/v1/attachments/{id}` → [Attachment\DestroyController::destroy](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/DestroyController.php#L67-L79)

Web UI: `POST /attachments/destroy/{attachment}` → [AttachmentController::destroy](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Controllers/AttachmentController.php#L79-L89)

两者都调用 `AttachmentRepository::destroy`，删除数据库记录和磁盘文件。

### 7.2 实体删除时级联清理附件

通过 Observer 模式实现，当实体被删除时自动清理其附件：

| Observer | 代码位置 |
|---|---|
| DeletedTransactionJournalObserver | [L69-L71](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedTransactionJournalObserver.php#L69-L71) |
| DeletedAccountObserver | [L47-L49](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedAccountObserver.php#L47-L49) |
| DeletedCategoryObserver | [L43-L45](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedCategoryObserver.php#L43-L45) |
| DeletedRecurrenceObserver | [L43-L45](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedRecurrenceObserver.php#L43-L45) |
| DeletedTagObserver | [L43-L45](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedTagObserver.php#L43-L45) |
| PiggyBankObserver | [L55-L57](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/PiggyBankObserver.php#L55-L57) |
| BillDestroyService | [L42-L43](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Destroy/BillDestroyService.php#L42-L43) |
| BudgetDestroyService | [L51-L52](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Destroy/BudgetDestroyService.php#L51-L52) |

**注意**：
- `DeletedTransactionGroupObserver` **不会**清理附件——Group 没有附件关系
- `DeletedTransactionJournalObserver` **会**清理附件——Journal 有 `attachments()` 关系
- 如果附件的 `attachable_type` 是 `Transaction`，删除该 Transaction 时 **不会**触发附件清理（因为 Transaction 没有 `attachments()` 关系），这也是孤儿问题的表现之一

## 8. 验证层：IsValidAttachmentModel

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

其中 `validateTransaction` 调用 [JournalAPIRepository::findTransaction](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/Journal/JournalAPIRepository.php#L47-L54)，验证 Transaction ID 是否存在且属于当前用户。

**StoreRequest** ([StoreRequest::rules](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Requests/Models/Attachment/StoreRequest.php#L59-L73))：
- `attachable_type` 必须 `in:Account,Bill,Budget,...` 列表
- `attachable_id` 必须通过 `IsValidAttachmentModel` 验证

**UpdateRequest** ([UpdateRequest::rules](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Requests/Models/Attachment/UpdateRequest.php#L61-L75))：
- `attachable_type` 必须 `in:Account,Bill,Budget,...` 列表（但非 required，可选更新）
- `attachable_id` 必须通过 `IsValidAttachmentModel` 验证（但非 required，可选更新）
- **两个条件必须同时出现**才会触发归属迁移（见 `AttachmentRepository::update` 中的 `array_key_exists` 检查）

## 9. 不一致问题总结

### 9.1 附件更新入口缺少 Transaction→Journal 自动提升

- 创建入口 `AttachmentFactory::create` 有 Transaction→Journal 转换
- 更新入口 `AttachmentRepository::update` 没有此转换
- 通过 API 更新附件可以将 `attachable_type` 设为 `Transaction`，产生孤儿记录
- 查询侧全部硬编码 `TransactionJournal::class`，孤儿记录无法被正常检索
- 删除 Transaction 时不会清理挂在其上的附件（无 Observer）

### 9.2 交易创建/编辑/API 更新三类入口均不处理附件

| 入口 | 附件操作 | 说明 |
|---|---|---|
| 交易创建（Web UI） | 前端异步上传到 Journal | 后端不感知附件 |
| 交易编辑（Web UI） | 前端异步上传到 Journal | 后端不感知附件，只能追加不能删除 |
| API 创建交易 | 不处理 | 用户需单独调用附件 API |
| API 更新交易 | 不处理 | `GroupUpdateService` / `JournalUpdateService` 无附件代码 |

### 9.3 拆分交易更新可能级联删除附件

API 更新拆分交易时，如果减少了 split 数量，多余的 Journal 会被删除，其上的附件也通过 Observer 级联删除，不可恢复。

### 9.4 Recurrence 的附件不会传递到生成的交易

- Recurrence 的附件归挂在 Recurrence 自身
- Recurrence 触发生成 TransactionJournal 时，附件不会自动复制

### 9.5 克隆交易不复制附件

[GroupCloneService::cloneJournal](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Update/GroupCloneService.php#L67-L120) 克隆 Journal 时复制了 notes、meta、categories、budgets、tags，但遗漏了 attachments。

### 9.6 Web UI 与 API 的附件管理能力差异

- Web UI 附件编辑页面只能修改 title/notes，**不能迁移归属**
- API 附件更新可以修改 `attachable_type`/`attachable_id`，**可以迁移归属**
- 但 API 迁移归属缺少 Transaction→Journal 的保护逻辑

## 10. 完整调用链图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          附件操作入口总览                                      │
├────────────────────┬──────────────────┬───────────────────────────────────────┤
│ 入口               │ 归挂目标          │ 调用路径                               │
├────────────────────┼──────────────────┼───────────────────────────────────────┤
│ Web 创建/编辑       │ Account          │ Controller                            │
│ Account/Bill/等    │ Bill 等          │ → saveAttachmentsForModel()           │
│                    │                  │ → AttachmentHelper::processFile()     │
│                    │                  │ ★ attachable()->associate($model)     │
├────────────────────┼──────────────────┼───────────────────────────────────────┤
│ Web 创建交易       │ TransactionJournal│ 前端 JS → POST /api/v1/attachments    │
│ (v1 / v2)         │                  │ → AttachmentFactory::create()         │
│                    │                  │ ★ 前端硬编码 TransactionJournal        │
│                    │                  │ → POST /api/v1/attachments/{id}/upload│
├────────────────────┼──────────────────┼───────────────────────────────────────┤
│ Web 编辑交易       │ TransactionJournal│ 同创建交易流程                          │
│ (v1 / v2)         │                  │ ★ 只能追加，不能删除已有附件              │
├────────────────────┼──────────────────┼───────────────────────────────────────┤
│ API 创建交易       │ 不处理附件        │ Transaction\StoreController           │
│                    │                  │ 用户需单独调用附件 API                   │
├────────────────────┼──────────────────┼───────────────────────────────────────┤
│ API 更新交易       │ 不处理附件        │ GroupUpdateService                    │
│                    │                  │ JournalUpdateService                  │
│                    │                  │ ★ 可能删除 Journal → 级联删除附件       │
├────────────────────┼──────────────────┼───────────────────────────────────────┤
│ API 创建附件       │ Transaction→Journal│ POST /api/v1/attachments             │
│                    │ 自动提升          │ → AttachmentFactory::create()         │
│                    │ 其他类型原样存储   │ ★ Transaction 自动转为 Journal         │
├────────────────────┼──────────────────┼───────────────────────────────────────┤
│ API 更新附件       │ 原样存储          │ PUT /api/v1/attachments/{id}          │
│ (归属迁移)         │ ★ 无自动提升     │ → AttachmentRepository::update()      │
│                    │                  │ ★ Transaction 可能原样落库             │
├────────────────────┼──────────────────┼───────────────────────────────────────┤
│ Web 编辑附件       │ 不可迁移归属      │ AttachmentController::update          │
│                    │                  │ → AttachmentRepository::update()      │
│                    │                  │ ★ AttachmentFormRequest 只含 title/notes│
├────────────────────┼──────────────────┼───────────────────────────────────────┤
│ API/Web 删除附件   │ 删除记录+磁盘文件 │ AttachmentRepository::destroy()       │
│                    │                  │ → 删除 at-{id}.data                   │
└────────────────────┴──────────────────┴───────────────────────────────────────┘
```

## 11. 核心文件索引

| 文件 | 职责 |
|---|---|
| [Attachment.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Attachment.php) | 附件模型，定义 MorphTo 关系 |
| [AttachmentFactory.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Factory/AttachmentFactory.php) | 创建工厂，含 Transaction→Journal 自动提升逻辑 |
| [AttachmentHelper.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Helpers/Attachments/AttachmentHelper.php) | Web UI 附件上传核心，saveAttachmentsForModel / saveAttachmentFromApi |
| [AttachmentRepository.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/Attachment/AttachmentRepository.php) | 附件仓库，store/update/destroy；update 中**缺少** Transaction→Journal 提升 |
| [AttachmentController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Controllers/AttachmentController.php) | Web UI 附件管理（编辑/删除/下载/查看） |
| [Attachment\StoreController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/StoreController.php) | API 附件创建 + 上传 |
| [Attachment\UpdateController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/UpdateController.php) | API 附件更新（含归属迁移） |
| [Attachment\DestroyController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/DestroyController.php) | API 附件删除 |
| [Attachment\StoreRequest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Requests/Models/Attachment/StoreRequest.php) | API 附件创建请求验证 |
| [Attachment\UpdateRequest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Requests/Models/Attachment/UpdateRequest.php) | API 附件更新请求验证 |
| [AttachmentFormRequest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Requests/AttachmentFormRequest.php) | Web UI 附件编辑请求（仅 title/notes） |
| [IsValidAttachmentModel.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Rules/IsValidAttachmentModel.php) | 验证 attachable_type + attachable_id 的合法性 |
| [Transaction\UpdateController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Transaction/UpdateController.php) | API 交易更新（不处理附件） |
| [GroupUpdateService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Update/GroupUpdateService.php) | 交易组更新服务（不处理附件，可能删除 Journal → 级联删除附件） |
| [JournalUpdateService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Update/JournalUpdateService.php) | Journal 更新服务（不处理附件） |
| [JournalDestroyService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Destroy/JournalDestroyService.php) | Journal 删除服务，触发 Observer |
| [DeletedTransactionJournalObserver.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedTransactionJournalObserver.php) | Journal 删除时级联删除附件 |
| [AttachmentCollection.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Helpers/Collector/Extensions/AttachmentCollection.php) | 交易查询时 LEFT JOIN 附件表 |
| [TransactionGroupRepository.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php) | 交易组仓库，getAttachments / countAttachments |
| [TransactionGroupEnrichment.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Support/JsonApi/Enrichments/TransactionGroupEnrichment.php) | 交易组数据富化，统计每个 Journal 的附件数量 |
| [GroupCloneService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Update/GroupCloneService.php) | 交易克隆服务（不复制附件） |
| [EditTransaction.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v1/src/components/transactions/EditTransaction.vue) | v1 前端交易编辑（含附件上传） |
| [process-attachments.js](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v2/src/pages/transactions/shared/process-attachments.js) | v2 前端附件上传处理（创建/编辑共用） |
| [edit.js](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v2/src/pages/transactions/edit.js) | v2 前端交易编辑入口 |
| [firefly.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/config/firefly.php#L213-L223) | valid_attachment_models 配置 |
