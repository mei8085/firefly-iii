# 附件与交易记录的关联关系分析

## 0. 四条链路落库差异总览

| 链路 | Transaction 模型涉及角色 | 最终落库 attachable_type | 备注 |
|---|---|---|---|
| **创建**：Web UI 交易创建/编辑 | 不涉及，前端硬编码 TransactionJournal | `FireflyIII\Models\TransactionJournal` | 安全，无孤儿风险 |
| **创建**：API 附件创建（POST /api/v1/attachments） | AttachmentFactory 自动提升：Transaction→Journal | `FireflyIII\Models\TransactionJournal` | 即使传入 Transaction，也被转换 |
| **创建**：Web UI 非交易实体（Account/Bill 等） | 不涉及，直接 associate($model) | 各实体 FQCN | 安全 |
| **更新**：API 附件更新（PUT /api/v1/attachments） | **无提升逻辑**，原样存储 | 可能为 `FireflyIII\Models\Transaction` | **孤儿漏洞** |
| **更新**：API 交易更新（PUT /api/v1/transactions） | 不处理附件 | 不变 | 但删除 split 会级联删除附件 |
| **更新**：Web UI 交易编辑 | 前端硬编码 Journal，只能追加新附件 | `TransactionJournal` | 不处理已有附件变更 |
| **查询**：全局附件列表（Web + API） | 通过 user_id HasMany 返回**所有类型**，**包括** Transaction | 可见所有（含孤儿） | 孤儿可见 |
| **查询**：交易上下文（Show/Collector/Enrichment） | 只查 attachable_type=TransactionJournal | **不可见** Transaction 类型 | 孤儿不可见 |
| **删除**：交易组/Journal 删除 | Observer 遍历 journal->attachments()（MorphMany 只认 Journal 类型） | **不清理** Transaction 类型附件 | **孤儿残留** |
| **删除**：单条 Transaction 删除（无独立入口） | TransactionObserver 无删除处理 | N/A | Transaction 只能随 Journal 删除，无独立删除入口 |

---

## 1. 数据库层：多态关联设计

### 1.1 attachments 表结构

定义于 [createAttachmentsTable](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/database/migrations/2016_06_16_000002_create_main_tables.php#L134-L162)：

```php
$table->increments('id');
$table->timestamps();
$table->softDeletes();
$table->integer('user_id', false, true);
$table->integer('attachable_id', false, true);     // 多态外键
$table->string('attachable_type', 255);             // 多态类型（存 FQCN，如 FireflyIII\Models\TransactionJournal）
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

核心设计：使用 Laravel 的 **Polymorphic Morph** 模式，`attachable_id` + `attachable_type` 组合指向上级实体。`user_id` 单独建立外键，用于快速按用户检索所有附件。

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

共 **9 种**实体可以拥有附件。注意 `Transaction::class` 在列表中是合法的。

## 2. 模型层关系

### 2.1 Attachment 模型（反向）

[Attachment::attachable](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Attachment.php#L89-L92)：

```php
public function attachable(): MorphTo
{
    return $this->morphTo();
}
```

[Attachment::user](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Attachment.php#L110-L113)（用于全局查询）：

```php
public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}
```

### 2.2 User / UserGroup 模型（全局附件入口）

[User::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/User.php#L118-L121)：

```php
public function attachments(): HasMany
{
    return $this->hasMany(Attachment::class);
}
```

[UserGroup::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/UserGroup.php#L86-L89)：

```php
public function attachments(): HasMany
{
    return $this->hasMany(Attachment::class);
}
```

**关键点**：这两个关系是普通的 `HasMany`，仅通过 `user_id` / `user_group_id` 关联，**不区分 `attachable_type`**。这意味着它们返回**所有类型**的附件，包括可能存在的孤儿 Transaction 类型附件。

### 2.3 各实体模型（正向 MorphMany）

所有合法的 attachable_type 模型都声明了 `attachments(): MorphMany`，但 `Transaction` 和 `TransactionGroup` 除外：

| 模型 | 关系声明 | 代码位置 |
|---|---|---|
| TransactionJournal | `$this->morphMany(Attachment::class, 'attachable')` | [TransactionJournal::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/TransactionJournal.php#L117-L120) |
| Account | 同上 | [Account::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Account.php#L98) |
| Bill | 同上 | [Bill::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Bill.php#L106) |
| Budget | 同上 | [Budget::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Budget.php#L74) |
| Category | 同上 | [Category::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Category.php#L76) |
| PiggyBank | 同上 | [PiggyBank::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/PiggyBank.php#L100) |
| Tag | 同上 | [Tag::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Tag.php#L80) |
| Recurrence | 同上 | [Recurrence::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Recurrence.php#L100) |
| **Transaction** | **未定义** | 不存在 `attachments()` 方法 |
| **TransactionGroup** | **未定义** | 不存在 `attachments()` 方法 |

### 2.4 三层交易模型结构

```
TransactionGroup (交易组)
  └── transactionJournals(): HasMany (无 ORDER BY，按 ID 升序返回)
        └── transactions(): HasMany
              ├── source (amount < 0)
              └── destination (amount > 0)
```

- `TransactionGroup::transactionJournals()` 未定义 `orderBy()`，实际按主键 ID 升序返回（InnoDB 默认顺序）
- `TransactionJournal::transactionType` → 外键指向 `TransactionType`，值来自 [TransactionTypeEnum](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Enums/TransactionTypeEnum.php#L30-L39)：
  - `WITHDRAWAL = 'Withdrawal'`（支出/取款）
  - `DEPOSIT = 'Deposit'`（收入/存款）
  - `TRANSFER = 'Transfer'`（转账）
  - `OPENING_BALANCE`、`RECONCILIATION`、`LIABILITY_CREDIT` 等

---

## 3. 创建链路（Create）

### 3.1 Web UI 交易创建/编辑（v1 / v2）

**共同特征**：前端硬编码传入 `attachable_type = 'TransactionJournal'`，后端不会触发 Transaction→Journal 转换。

#### 拆分交易（Split）与附件的索引对应关系

##### 后端：Journal 创建顺序

[TransactionJournalFactory::create](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Factory/TransactionJournalFactory.php#L109-L151)：

```php
foreach ($transactions as $index => $row) {
    $journal = $this->createJournal(new NullArrayObject($row));
    $collection->push($journal);  // 按提交顺序 push：[Split0, Split1, Split2]
}
return $collection;
```

[TransactionGroupFactory::create](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Factory/TransactionGroupFactory.php#L81-L89)：

```php
$group->transactionJournals()->saveMany($collection);  // 保存顺序 = collection 顺序
```

**Journal 创建顺序 = 前端表单 splits 数组的提交顺序（Split0 → Split1 → Split2）**。

##### 后端：Journal 返回顺序

[TransactionGroupTransformer::transformJournals](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Transformers/TransactionGroupTransformer.php#L368-L378)：

```php
foreach ($transactionJournals as $journal) {   // $group->transactionJournals，无 ORDER BY
    $result[] = $this->transformJournal($journal);
}
```

Laravel HasMany 无 ORDER BY 时，按主键 ID 升序返回。由于 saveMany 按提交顺序保存，ID 分配也按提交顺序，所以 **API 返回顺序 = 创建顺序 = 提交顺序（[Split0, Split1, Split2]）**。

##### 前端：附件与 Journal 的匹配逻辑

**v1 CreateTransaction.vue** [L565-L594](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v1/src/components/transactions/CreateTransaction.vue#L565-L594)：

```javascript
collectAttachmentData(response) {
    response.data.data.attributes.transactions = response.data.data.attributes.transactions.reverse();
    // 反转后变成 [Split2, Split1, Split0]
    let attachments = $('input[name="attachments[]"]');  // DOM 顺序 = [Split0_input, Split1_input, Split2_input]
    for (const key in attachments) {
        toBeUploaded.push({
            journal: response.data.data.attributes.transactions[key].transaction_journal_id,
            // key=0 → transactions[0] = Split2
            // key=1 → transactions[1] = Split1
            // key=2 → transactions[2] = Split0
            file: attachments[key].files[fileKey]
        });
    }
}
```

**v2 process-attachments.js** [L61-L80](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v2/src/pages/transactions/shared/process-attachments.js#L61-L80)：

```javascript
export function processAttachments(groupId, transactions) {
    transactions = transactions.reverse();
    let attachments = document.querySelectorAll('input[name="attachments[]"]');
    for (const key in attachments) {
        toBeUploaded.push({
            journal: transactions[key].transaction_journal_id,
            file: attachments[key].files[fileKey]
        });
    }
}
```

**附件-Journal 对应关系**（反转后的交叉匹配）：

| 输入框索引 key | DOM 中的 split | `.reverse()` 后 transactions[key] | 结果 |
|---|---|---|---|
| 0 | Split 0 (第一个标签页) | Split 2 (最后创建的 journal) | Split0 的附件 → 最后一个 Journal |
| 1 | Split 1 (第二个标签页) | Split 1 (中间的 journal) | Split1 的附件 → 中间 Journal |
| 2 | Split 2 (最后一个标签页) | Split 0 (最先创建的 journal) | Split2 的附件 → 第一个 Journal |

**关键结论**：附件与 Journal 的对应是**反转的首尾交叉**。前端 Split 标签页 0 的文件挂到后端最后创建的 Journal 上，反之亦然。

**交易类型（Withdrawal/Deposit/Transfer）不影响此对应逻辑**——对应关系仅取决于：
1. 表单 splits 的 DOM 顺序（由 `index` 控制）
2. 后端保存顺序（= 提交数组顺序）
3. 前端 `.reverse()` 操作

每种交易类型的差异仅体现在 **Account 校验规则** 中（见 [TransactionJournalFactory::validateAccounts](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Factory/TransactionJournalFactory.php)），与附件归属无关。

#### 落库结果

v1/v2 前端都通过 axios POST：

```json
{
  "filename": "receipt.pdf",
  "attachable_type": "TransactionJournal",
  "attachable_id": <journal_id>
}
```

后端 [AttachmentFactory::create](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Factory/AttachmentFactory.php#L44-L84) 中 `$model === TransactionJournal::class`，不触发 if 分支，直接落库：

```
attachable_type = 'FireflyIII\Models\TransactionJournal'
attachable_id   = <journal_id>
```

### 3.2 API 直接创建附件（POST /api/v1/attachments）

调用链：[Attachment\StoreController::store](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/StoreController.php#L74-L92) → `AttachmentRepository::store` → `AttachmentFactory::create`

[AttachmentFactory::create](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Factory/AttachmentFactory.php#L44-L84) 的 Transaction→Journal 自动提升：

```php
$model = str_contains((string) $data['attachable_type'], 'FireflyIII')
    ? $data['attachable_type']
    : sprintf('FireflyIII\Models\%s', $data['attachable_type']);

if (Transaction::class === $model) {
    $transaction = $this->user->transactions()->find((int) $data['attachable_id']);
    $data['attachable_id'] = $transaction->transaction_journal_id;
    $model = TransactionJournal::class;
}
```

| API 传入 attachable_type | 落库 attachable_type | 备注 |
|---|---|---|
| `'TransactionJournal'` | `FireflyIII\Models\TransactionJournal` | 原样存储 |
| `'Transaction'` | `FireflyIII\Models\TransactionJournal` | attachable_id 也被改为 journal ID |
| `'FireflyIII\Models\Transaction'` | `FireflyIII\Models\TransactionJournal` | 同上，命名空间形式也会被转换 |
| 其他合法模型 | 各实体 FQCN | 原样存储 |

### 3.3 Web UI 非交易实体（Account/Bill 等）

调用链：各实体 CreateController → [AttachmentHelper::saveAttachmentsForModel](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Helpers/Attachments/AttachmentHelper.php#L263-L310)

```php
$attachment = new Attachment();
$attachment->user()->associate($user);
$attachment->attachable()->associate($model);  // Eloquent MorphTo::associate 设置 type + id
$attachment->save();
```

通过 `associate()` 取模型真实的 `getMorphClass()`，直接写入。由于这些场景不会传入 Transaction 实例，**不会产生 Transaction 类型的记录**。

---

## 4. 更新链路（Update）

### 4.1 API 附件更新（PUT /api/v1/attachments）

调用链：[Attachment\UpdateController::update](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/UpdateController.php#L69-L86) → [AttachmentRepository::update](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/Attachment/AttachmentRepository.php#L132-L155)

```php
public function update(Attachment $attachment, array $data): Attachment
{
    if (array_key_exists('title', $data)) {
        $attachment->title = $data['title'];
    }
    if (array_key_exists('filename', $data) && ...) {
        $attachment->filename = $data['filename'];
    }
    // ★ 归属迁移：无 Transaction→Journal 提升逻辑！
    if (array_key_exists('attachable_type', $data) && array_key_exists('attachable_id', $data)) {
        $attachment->attachable_id   = (int) $data['attachable_id'];
        $attachment->attachable_type = sprintf('FireflyIII\Models\%s', $data['attachable_type']);
    }
    $attachment->save();
    ...
}
```

**更新入口 vs 创建入口的关键差异**：

| 维度 | AttachmentFactory::create（创建） | AttachmentRepository::update（更新） |
|---|---|---|
| Transaction→Journal 提升 | **有**，自动转换 | **无**，原样存储 |
| 传入 `attachable_type=Transaction` | 落库为 `TransactionJournal` | 落库为 `FireflyIII\Models\Transaction` |
| 传入 `attachable_id=123`（Transaction ID） | 被替换为 `transaction_journal_id` | **直接存为 123**（Transaction ID） |

**孤儿漏洞**：通过 API `PUT /api/v1/attachments/123` 迁移归属到 `attachable_type=Transaction` 时，产生的记录：
- `attachable_type = 'FireflyIII\Models\Transaction'`
- `attachable_id` = Transaction ID（而非 Journal ID）
- Transaction 模型没有 `attachments()` 关系，无法通过 `$transaction->attachments` 查出
- 所有交易上下文查询都硬编码 `TransactionJournal::class`，无法查出
- 只有全局附件列表和 `GET /api/v1/attachments/{id}` 单条查询可以访问

验证层 `IsValidAttachmentModel` 只验证模型 ID 是否存在且属于当前用户，**不阻止这种落库**。

### 4.2 API 交易更新（PUT /api/v1/transactions）

调用链：[Transaction\UpdateController::update](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Transaction/UpdateController.php#L76-L131) → `TransactionGroupRepository::update` → [GroupUpdateService::update](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Update/GroupUpdateService.php#L48-L115)

**GroupUpdateService 完全不涉及附件操作**（无 attachment/attachable 代码）。但存在级联删除风险：

```php
// 找出需要删除的 journal（原有但提交中不包含的）
$result = array_diff($existing, $updated);
foreach ($result as $deletedId) {
    $journal = $transactionGroup->transactionJournals()->find((int) $deletedId);
    $service = app(JournalDestroyService::class);
    $service->destroy($journal);  // ★ 触发 DeletedTransactionJournalObserver
}
```

Observer 删除 journal 时会**级联删除该 journal 的附件**（仅 MorphMany 关联的 Journal 类型附件）。见下方删除链路详解。

### 4.3 Web UI 交易编辑

后端 [Transaction\EditController](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Controllers/Transaction/EditController.php) **不处理附件上传**，只渲染视图。附件上传由前端 JS 异步完成，逻辑同 3.1 交易创建，调用同一个 `collectAttachmentData()` / `processAttachments()`。

**限制**：
- 只能**追加**新附件
- 无法通过交易编辑表单**删除**或**迁移**已有附件
- 已有附件需到 `/attachments` 全局管理页面操作
- 新上传的附件仍走前端 `.reverse()` 索引匹配逻辑

### 4.4 Web UI 附件编辑页面

[AttachmentController::update](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Controllers/AttachmentController.php#L166-L183) 使用 [AttachmentFormRequest](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Requests/AttachmentFormRequest.php#L45-L48)：

```php
public function getAttachmentData(): array
{
    return ['title' => $this->convertString('title'), 'notes' => $this->convertString('notes')];
}
```

**Web UI 不允许修改归属**（`attachable_type` / `attachable_id` 不在允许修改的字段中）。只有 API 更新入口能迁移归属。

---

## 5. 查询链路（Query）

### 5.1 全局附件列表（可见所有类型）

#### Web UI `/attachments` 页面

[AttachmentController::index](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Controllers/AttachmentController.php#L151-L161)：

```php
$set = $this->repository->get()->reverse();
```

调用 [AttachmentRepository::get](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/Attachment/AttachmentRepository.php#L77-L80)：

```php
public function get(): Collection
{
    return $this->user->attachments()->get();  // User::attachments() = HasMany，不分 attachable_type
}
```

#### API `GET /api/v1/attachments`

[Attachment\ShowController::index](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/ShowController.php#L122-L150)：

```php
$collection = $this->repository->get();  // 同样调用 $user->attachments()
```

**全局查询特征**：
- 通过 `user_id` HasMany 关联，**不区分 attachable_type**
- 返回**所有类型**的附件：Account、Bill、Budget、...、TransactionJournal、**Transaction（含孤儿）**
- 孤儿记录（attachable_type=Transaction）**可见**

[AttachmentTransformer](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Transformers/AttachmentTransformer.php#L48-L68) 返回时将 `attachable_type` 去掉命名空间前缀显示：

```php
'attachable_type' => str_replace('FireflyIII\Models\\', '', $attachment->attachable_type),
// 'FireflyIII\Models\Transaction' → 'Transaction'
```

### 5.2 交易上下文查询（仅可见 TransactionJournal 类型）

所有交易上下文的附件查询都硬编码 `attachable_type = TransactionJournal::class`，**Transaction 类型记录不可见**。

#### 5.2.1 交易 Show 页面（Web UI）

[Transaction\ShowController::show](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Controllers/Transaction/ShowController.php#L145) → [TransactionGroupRepository::getAttachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php#L106-L136)：

```php
$journals = $group->transactionJournals->pluck('id')->toArray();
$set = Attachment::whereIn('attachable_id', $journals)
    ->where('attachable_type', TransactionJournal::class)   // ★ 硬编码
    ->where('uploaded', true)
    ->whereNull('deleted_at')
    ->get();
```

结果按 `attachable_id`（journal ID）分组返回数组。页面在 [show.twig#L500-L506](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/views/transactions/show.twig#L500-L506) 中按 journal 渲染附件列表：

```twig
{% if attachments[journal.transaction_journal_id]|length > 0 %}
    {% include 'list.attachments' with {attachments: attachments[journal.transaction_journal_id]} %}
{% endif %}
```

#### 5.2.2 API 交易 Show/List（富化统计）

[TransactionGroupEnrichment](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Support/JsonApi/Enrichments/TransactionGroupEnrichment.php#L169-L179)：

```php
$attachments = Attachment::query()
    ->whereIn('attachable_id', $this->journalIds)
    ->where('attachable_type', TransactionJournal::class)   // ★ 硬编码
    ->groupBy('attachable_id')
    ->get(['attachable_id', DB::raw('COUNT(id) as nr_of_attachments')]);
```

#### 5.2.3 交易列表 Collector LEFT JOIN

[AttachmentCollection::joinAttachmentTables](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Helpers/Collector/Extensions/AttachmentCollection.php#L510-L529)：

```php
$this->query
    ->leftJoin('attachments', 'attachments.attachable_id', '=', 'transaction_journals.id')
    ->where(static function (EloquentBuilder $q1): void {
        $q1->where('attachments.attachable_type', TransactionJournal::class);  // ★ 硬编码
        $q1->orWhereNull('attachments.attachable_type');
    });
```

#### 5.2.4 API 交易附件子端点

[Transaction\ListController::attachments](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Transaction/ListController.php#L70-L94)：

```php
foreach ($transactionGroup->transactionJournals as $transactionJournal) {
    // $journalAPIRepository->getAttachments() 内部调用 $journal->attachments
    // 即 TransactionJournal::attachments() = MorphMany，只认 attachable_type=Journal
    $collection = $this->journalAPIRepository->getAttachments($transactionJournal)->merge($collection);
}
```

### 5.3 查询可见性对比矩阵

| 查询入口 | 底层关系/条件 | Journal 类型附件 | Transaction 类型附件（孤儿） | Account 等其他类型 |
|---|---|---|---|---|
| Web `/attachments` 全局列表 | `user->attachments()` HasMany | ✅ 可见 | ✅ 可见 | ✅ 可见 |
| API `GET /api/v1/attachments` 全局列表 | `user->attachments()` HasMany | ✅ 可见 | ✅ 可见 | ✅ 可见 |
| API `GET /api/v1/attachments/{id}` 单条 | 路由绑定按 ID | ✅ 可见 | ✅ 可见 | ✅ 可见 |
| 交易 Show 页面附件 | `WHERE attachable_type = Journal` | ✅ 可见 | ❌ 不可见 | ❌ 不涉及 |
| API 交易附件子端点 | `$journal->attachments()` MorphMany | ✅ 可见 | ❌ 不可见 | ❌ 不涉及 |
| 交易列表富化统计 | `WHERE attachable_type = Journal GROUP BY` | ✅ 计数 | ❌ 不计入 | ❌ 不涉及 |
| Collector LEFT JOIN hasAttachment | `LEFT JOIN ... WHERE type=Journal` | ✅ 正确标记 | ❌ 误判为无附件 | ❌ 不涉及 |

---

## 6. 删除链路（Delete）

### 6.1 附件自身删除

两个入口都调用 `AttachmentRepository::destroy`，逻辑一致：

- API: [Attachment\DestroyController::destroy](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/DestroyController.php#L67-L79)
- Web UI: [AttachmentController::destroy](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Controllers/AttachmentController.php#L79-L89)

[AttachmentRepository::destroy](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/Attachment/AttachmentRepository.php#L52-L67)：

```php
Storage::disk('upload')->delete($path);  // 删除磁盘文件 at-{id}.data
$attachment->delete();                   // 软删除（SoftDeletes）数据库记录
```

### 6.2 交易组删除 → 级联清理

[DeletedTransactionGroupObserver::deleting](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedTransactionGroupObserver.php#L34-L40)：

```php
foreach ($transactionGroup->transactionJournals()->get() as $journal) {
    $journal->delete();  // 触发 DeletedTransactionJournalObserver
}
```

### 6.3 Journal 删除 → 级联清理（核心逻辑）

[DeletedTransactionJournalObserver::deleting](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedTransactionJournalObserver.php#L38-L81)：

```php
public function deleting(TransactionJournal $transactionJournal): void
{
    // 1. 删除子 Transaction（禁用事件避免循环）
    TransactionJournal::withoutEvents(static function () use ($transactionJournal): void {
        foreach ($transactionJournal->transactions()->get() as $transaction) {
            $transaction->delete();  // ★ 不会触发 TransactionObserver，且不会清理 Transaction 类型附件
        }
    });

    // 2. 删除关联表（categories/budgets/tags/links 等）
    DB::table('budget_transaction_journal')->where(...)->delete();
    DB::table('category_transaction_journal')->where(...)->delete();
    DB::table('tag_transaction_journal')->where(...)->delete();

    // 3. ★ 删除附件：通过 MorphMany 关系
    foreach ($transactionJournal->attachments()->get() as $attachment) {
        $repository->destroy($attachment);
        // $journal->attachments() = MorphMany，WHERE attachable_type=Journal AND attachable_id=journal.id
        // 不会匹配 attachable_type=Transaction 的记录！
    }
}
```

### 6.4 Transaction 删除 → 无独立入口

- Transaction **没有独立的删除入口**：只能通过 `withoutEvents` 在 Journal 删除时被级联删除
- [TransactionObserver](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/TransactionObserver.php) 只有 `created` 和 `updated` 事件，**无 `deleting` 处理**
- Transaction 模型没有 `attachments()` 关系，即使假设有独立删除入口也无法触发 MorphMany 级联清理

### 6.5 孤儿残留链路

**场景**：通过 API `PUT /api/v1/attachments/{id}` 将附件迁移到 `attachable_type=Transaction`，然后删除该 Transaction 所属的 Journal。

| 步骤 | attachable_type | 结果 |
|---|---|---|
| 1. 创建附件 → 挂到 Journal#10 | TransactionJournal | 正常可见 |
| 2. API 更新 → `attachable_type=Transaction`，`attachable_id=99`（Journal#10 下的某个 Transaction） | Transaction | 全局列表仍可见，交易上下文消失 |
| 3. 删除 Journal#10 | — | Observer 走 `$journal->attachments()` → **不匹配** Transaction 类型记录 → **孤儿残留** |
| 4. 磁盘文件 | — | `at-{id}.data` 未被删除，永久占用存储 |

### 6.6 各 Observer 对附件的清理覆盖

| Observer | 清理方式 | 清理 Journal 类型 | 清理 Transaction 类型 | 其他类型 |
|---|---|---|---|---|
| DeletedTransactionJournalObserver | `$journal->attachments()` MorphMany | ✅ 匹配 | ❌ **不匹配** | N/A |
| DeletedAccountObserver | `$account->attachments()` MorphMany | N/A | N/A | ✅ 覆盖 |
| DeletedCategoryObserver | `$category->attachments()` MorphMany | N/A | N/A | ✅ 覆盖 |
| DeletedRecurrenceObserver | `$recurrence->attachments()` MorphMany | N/A | N/A | ✅ 覆盖 |
| DeletedTagObserver | `$tag->attachments()` MorphMany | N/A | N/A | ✅ 覆盖 |
| PiggyBankObserver | `$piggyBank->attachments()` MorphMany | N/A | N/A | ✅ 覆盖 |
| BillDestroyService | `$bill->attachments()` MorphMany | N/A | N/A | ✅ 覆盖 |
| BudgetDestroyService | `$budget->attachments()` MorphMany | N/A | N/A | ✅ 覆盖 |
| **DeletedTransactionGroupObserver** | 循环调用 `$journal->delete()` | ✅（间接） | ❌（间接不匹配） | N/A |
| **无 DeletedTransactionObserver** | 不存在 | N/A | ❌ 无清理逻辑 | N/A |

---

## 7. 交易类型（Withdrawal / Deposit / Transfer）与附件

### 7.1 交易类型确定位置

[TransactionJournalFactory::createJournal](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Factory/TransactionJournalFactory.php#L230-L342) 中创建 Journal 时确定类型：

```php
$type = $this->typeRepository->findTransactionType(null, $row['type']);
// $row['type'] = 'withdrawal' / 'deposit' / 'transfer' / ...

// 仅 Withdrawal 类型可关联 Bill
$billId = TransactionTypeEnum::WITHDRAWAL->value === $type->type && $bill instanceof Bill ? $bill->id : null;

$journal = TransactionJournal::create([
    'transaction_type_id' => $type->id,  // 外键到 transaction_types 表
    'bill_id' => $billId,                // 仅 Withdrawal
    // ...
]);
```

TransactionType 模型提供辅助方法 [L64-L79](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/TransactionType.php#L64-L79)：
- `isWithdrawal()` → `TransactionTypeEnum::WITHDRAWAL`
- `isDeposit()` → `TransactionTypeEnum::DEPOSIT`
- `isTransfer()` → `TransactionTypeEnum::TRANSFER`
- `isOpeningBalance()` → `TransactionTypeEnum::OPENING_BALANCE`

### 7.2 Source / Destination Transaction 的确定

每种交易类型的 source（负金额）和 destination（正金额）语义：

| 类型 | source（amount < 0） | destination（amount > 0） |
|---|---|---|
| Withdrawal | 资产账户（扣钱） | 支出类别/对方 |
| Deposit | 收入类别/对方 | 资产账户（收钱） |
| Transfer | 源资产账户 | 目标资产账户 |

[TransactionGroupTransformer::getSourceTransaction](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Transformers/TransactionGroupTransformer.php#L226-L235)：

```php
$result = $journal->transactions->first(static fn (Transaction $transaction): bool => (float) $transaction->amount < 0);
```

[TransactionGroupTransformer::getDestinationTransaction](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Transformers/TransactionGroupTransformer.php#L183-L192)：

```php
$result = $journal->transactions->first(static fn (Transaction $transaction): bool => (float) $transaction->amount > 0);
```

**附件归挂在 Journal 上，与 source/destination Transaction 无关**。`attachable_id` 指向 `journal.id`，无论哪条 Transaction。

### 7.3 不同交易类型的附件-Journal 对应：完全一致

| 交易类型 | journal 创建顺序 | 前端 `.reverse()` 逻辑 | 附件归挂对象 |
|---|---|---|---|
| Withdrawal | 按提交数组顺序 | 反转交叉匹配 | TransactionJournal |
| Deposit | 按提交数组顺序 | 反转交叉匹配 | TransactionJournal |
| Transfer | 按提交数组顺序 | 反转交叉匹配 | TransactionJournal |
| Opening Balance | 按提交数组顺序 | 反转交叉匹配 | TransactionJournal |

**交易类型完全不影响附件归挂逻辑**，影响的是：
1. source/destination 账户的校验规则（validateAccounts）
2. Bill 可否关联（仅 Withdrawal）
3. 默认币种选择（getCurrencyByAccount）
4. 前端表单字段展示（source_name/destination_name 显示的语义）

### 7.4 在 Show 页面按 Journal 展示附件

[show.twig#L500-L506](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/views/transactions/show.twig#L500-L506) 遍历每个 split（= 每个 Journal）展示对应附件：

```twig
{% for journal in groupArray.transactions %}
    {# 每个 split 的区块 #}
    ...
    {% if attachments[journal.transaction_journal_id]|length > 0 %}
        <div class="box">
            <h3>{{ 'attachments'|_ }}</h3>
            {% include 'list.attachments' with {attachments: attachments[journal.transaction_journal_id]} %}
        </div>
    {% endif %}
{% endfor %}
```

由于 `$this->repository->getAttachments()` 返回的是 `[journalId => [attachments]]` 的键值数组，按 journal ID 精确查找对应附件。此逻辑对所有交易类型通用。

---

## 8. 验证层

[IsValidAttachmentModel](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Rules/IsValidAttachmentModel.php#L49-L87) 对所有 API 请求中的 `attachable_type` + `attachable_id` 进行校验：

```php
$result = match ($this->model) {
    Account::class            => $this->validateAccount((int) $value),
    Bill::class               => $this->validateBill((int) $value),
    Budget::class             => $this->validateBudget((int) $value),
    Category::class           => $this->validateCategory((int) $value),
    PiggyBank::class          => $this->validatePiggyBank((int) $value),
    Tag::class                => $this->validateTag((int) $value),
    Transaction::class        => $this->validateTransaction((int) $value),   // 通过 JournalAPIRepository::findTransaction
    TransactionJournal::class => $this->validateJournal((int) $value),
    default                   => false
};
```

[Attachment\StoreRequest](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Requests/Models/Attachment/StoreRequest.php#L59-L73)：
- `attachable_type` 必须在合法列表中（in:Account,Bill,Budget,...）
- `attachable_id` 必须通过 `IsValidAttachmentModel`

[Attachment\UpdateRequest](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Requests/Models/Attachment/UpdateRequest.php#L61-L75)：
- 同上，但两个字段均非 required（可选更新）
- 必须同时出现才触发归属迁移（`array_key_exists` 同时为 true）

---

## 9. 完整调用链图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                       四条链路 Transaction vs TransactionJournal 总览                 │
├──────────────┬──────────────────────────────┬───────────────────────────┬──────────┤
│ 链路         │ 入口                          │ Transaction 涉及方式       │ 落库结果  │
├──────────────┼──────────────────────────────┼───────────────────────────┼──────────┤
│              │ Web 交易创建/编辑 (v1/v2)     │ 前端硬编码 Journal         │ Journal  │
│   创建       │ API POST /api/v1/attachments  │ Factory 自动提升           │ Journal  │
│              │ Web 非交易实体创建             │ associate($model)          │ 实体类型  │
├──────────────┼──────────────────────────────┼───────────────────────────┼──────────┤
│              │ API PUT /api/v1/attachments   │ 无提升，原样存储           │ 可能为    │
│              │                              │                           │ Transaction│
│   更新       │ API PUT /api/v1/transactions  │ 不处理附件                 │ 不变      │
│              │                              │ 删除 split → Observer      │ 级联删除   │
│              │ Web 交易编辑                  │ 前端硬编码 Journal（追加） │ Journal  │
│              │ Web 附件编辑                  │ 不能改归属                 │ 不变      │
├──────────────┼──────────────────────────────┼───────────────────────────┼──────────┤
│   查询       │ 全局列表 (Web + API)          │ user->attachments()        │ 全部可见  │
│              │ 交易 Show / Collector /       │ WHERE attachable_type =    │ 仅        │
│              │ Enrichment / 子端点            │ TransactionJournal         │ Journal   │
├──────────────┼──────────────────────────────┼───────────────────────────┼──────────┤
│              │ 删除 Journal → Observer       │ $journal->attachments()    │ 删 Journal│
│              │                              │ MorphMany                  │ 不删      │
│   删除       │ 删除 Group → 级联删 Journals  │ 同上                       │ 同上      │
│              │ 删除 Transaction              │ 无独立入口 + 无 Observer   │ 孤儿残留  │
└──────────────┴──────────────────────────────┴───────────────────────────┴──────────┘
```

---

## 10. 不一致问题总结

### 10.1 创建 vs 更新入口的 Transaction→Journal 保护不一致

- **创建入口** `AttachmentFactory::create`：有 Transaction→Journal 自动提升
- **更新入口** `AttachmentRepository::update`：无此保护
- **风险**：API 更新可以产生 `attachable_type=Transaction` 的孤儿记录

### 10.2 查询侧全局 vs 交易上下文的可见性不一致

- **全局附件列表**：通过 `user->attachments()` HasMany，**所有类型可见**（含孤儿）
- **交易上下文**：全部硬编码 `attachable_type = TransactionJournal::class`，孤儿不可见
- **结果**：全局列表显示的附件数量可能大于交易详情页中显示的数量

### 10.3 删除链路的级联清理不完整

- `DeletedTransactionJournalObserver` 通过 `$journal->attachments()`（MorphMany）清理附件
- MorphMany 自动附加 `WHERE attachable_type = Journal` 条件
- **不匹配** `attachable_type = Transaction` 的记录
- 孤儿永久残留，磁盘文件 `at-{id}.data` 也无法被清理

### 10.4 Transaction 合法声明 vs 实际无关系

- `Transaction::class` 在 `valid_attachment_models` 中声明为合法
- 但 `Transaction` 模型**没有** `attachments()` 关系方法
- 属于"合法但不可操作"的矛盾状态

### 10.5 前端 Split 索引反转匹配

- 表单 Split0（第一个标签页）的附件 → 最后创建的 Journal
- 表单 SplitN（最后一个标签页）的附件 → 最先创建的 Journal
- 对用户来说是"反转交叉"的，虽然功能正确但不易于调试

### 10.6 Recurrence 附件不传递 / 克隆交易不复制附件

- Recurrence 触发生成交易时不会复制附件
- `GroupCloneService::cloneJournal` 复制 notes/meta/categories/tags 但遗漏 attachments

---

## 11. 核心文件索引

| 文件 | 职责 |
|---|---|
| [Attachment.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/Attachment.php) | 附件模型（MorphTo + user BelongsTo） |
| [AttachmentFactory.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Factory/AttachmentFactory.php) | 创建工厂（含 Transaction→Journal 自动提升） |
| [AttachmentHelper.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Helpers/Attachments/AttachmentHelper.php) | Web UI 上传核心（saveAttachmentsForModel / saveAttachmentFromApi） |
| [AttachmentRepository.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/Attachment/AttachmentRepository.php) | 仓库：store/update/destroy/get；**update 无 Transaction→Journal 提升**；**get 通过 user_id 返回所有类型** |
| [User.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/User.php#L118-L121) | `attachments(): HasMany` = 全局附件入口（不区分 attachable_type） |
| [UserGroup.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Models/UserGroup.php#L86-L89) | 同上 |
| [AttachmentController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Controllers/AttachmentController.php) | Web UI：index（全局列表）/ edit（仅 title+notes）/ destroy / download / view |
| [Attachment\StoreController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/StoreController.php) | API 创建 + 上传文件 |
| [Attachment\ShowController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/ShowController.php) | API 全局列表（`repository->get()` = user->attachments）+ show + download |
| [Attachment\UpdateController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/UpdateController.php) | API 更新（含归属迁移入口） |
| [Attachment\DestroyController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Api/V1/Controllers/Models/Attachment/DestroyController.php) | API 删除 |
| [AttachmentTransformer.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Transformers/AttachmentTransformer.php) | 序列化：去掉命名空间前缀显示 attachable_type |
| [AttachmentFormRequest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Http/Requests/AttachmentFormRequest.php) | Web UI 表单请求：仅允许修改 title/notes |
| [IsValidAttachmentModel.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Rules/IsValidAttachmentModel.php) | 校验 attachable_type+id 合法性，含 validateTransaction |
| [TransactionJournalFactory.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Factory/TransactionJournalFactory.php#L109-L151) | Journal 创建顺序 = 提交数组顺序 |
| [TransactionGroupFactory.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Factory/TransactionGroupFactory.php#L81-L89) | saveMany 按 collection 顺序保存 Journals |
| [TransactionGroupTransformer.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Transformers/TransactionGroupTransformer.php#L368-L378) | 返回 Journal 顺序 = ID 升序（= 创建顺序） |
| [GroupUpdateService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Update/GroupUpdateService.php) | 交易更新：不处理附件，但删 split → 级联删附件 |
| [JournalUpdateService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Services/Internal/Update/JournalUpdateService.php) | Journal 更新：完全不涉及附件 |
| [TransactionGroupRepository.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php#L106-L136) | getAttachments：WHERE attachable_type = TransactionJournal |
| [TransactionGroupEnrichment.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Support/JsonApi/Enrichments/TransactionGroupEnrichment.php#L169-L179) | 附件计数统计：WHERE attachable_type = TransactionJournal |
| [AttachmentCollection.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Helpers/Collector/Extensions/AttachmentCollection.php#L510-L529) | LEFT JOIN：WHERE attachable_type = TransactionJournal |
| [DeletedTransactionGroupObserver.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedTransactionGroupObserver.php) | 删 Group → 循环删 Journals → 触发 Journal Observer |
| [DeletedTransactionJournalObserver.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/DeletedTransactionJournalObserver.php#L69-L72) | 删附件：通过 $journal->attachments()（MorphMany，仅 Journal 类型） |
| [TransactionObserver.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Handlers/Observer/TransactionObserver.php) | 只有 created/updated 主币种重算，无 deleting |
| [TransactionTypeEnum.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/app/Enums/TransactionTypeEnum.php) | WITHDRAWAL / DEPOSIT / TRANSFER 等枚举 |
| [CreateTransaction.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v1/src/components/transactions/CreateTransaction.vue#L565-L594) | v1：`.reverse()` + 按 key 索引匹配 |
| [EditTransaction.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v1/src/components/transactions/EditTransaction.vue#L874-L885) | v1 编辑：同创建逻辑 |
| [process-attachments.js](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/assets/v2/src/pages/transactions/shared/process-attachments.js#L61-L80) | v2：`.reverse()` + 按 key 索引匹配 |
| [attachments.blade.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/views/v2/partials/form/transaction/attachments.blade.php) | v2 表单输入：`name="attachments[]"` + `:data-index="index"` |
| [show.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/resources/views/transactions/show.twig#L500-L506) | Show 页面按 journal ID 展示附件 |
| [firefly.php](file:///d:/fz/0508-2/solo-dogfeeding/code/127-firefly-iii/config/firefly.php#L213-L223) | valid_attachment_models 配置（Transaction 在列表中为合法） |
