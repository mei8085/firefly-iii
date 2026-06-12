# Firefly III 账户对账（Reconciliation）代码深度分析报告

## 一、概述

Firefly III 作为个人财务管理系统，账户对账功能用于将用户本地记录的账户余额与银行对账单进行核对。其核心设计理念是：**在不修改任何历史交易数据的前提下，通过标记已对账状态 + 生成补偿调节交易的方式，完成余额修正**。本报告深入分析代码实现细节。

---

## 二、核心架构与路由入口

### 2.1 路由定义

对账功能的全部路由注册在 `routes/web.php`：

| 路由 | 方法 | 控制器方法 | 作用 |
|------|------|-----------|------|
| `accounts/reconcile/{account}/index/{start?}/{end?}` | GET | `Account\ReconcileController@reconcile` | 进入对账页面 |
| `accounts/reconcile/{account}/submit/{start?}/{end?}` | POST | `Account\ReconcileController@submit` | 提交对账结果 |
| `accounts/reconcile/{account}/overview/{start?}/{end?}` | GET | `Json\ReconcileController@overview` | AJAX 获取对账概览（差额计算） |
| `accounts/reconcile/{account}/transactions/{start?}/{end?}` | GET | `Json\ReconcileController@transactions` | AJAX 获取对账期间交易列表 |
| `transactions/unreconcile/{tj}` | POST | `Transaction\EditController@unreconcile` | 反对账（撤销对账标记） |

### 2.2 关键枚举定义

- **交易类型枚举**：`app/Enums/TransactionTypeEnum.php` 定义了 `RECONCILIATION = 'Reconciliation'`，即补偿调节交易有独立的类型标识
- **账户类型枚举**：`app/Enums/AccountTypeEnum.php` 定义了 `RECONCILIATION = 'Reconciliation account'`，即每个资产账户对应一个专门的"对账调节账户"

---

## 三、对账流程全链路分析

### 3.1 阶段一：进入对账页面（计算起止余额）

**入口方法**：`app/Http/Controllers/Account/ReconcileController.php` 中的 `reconcile()`

流程说明：

1. **前置校验**：仅允许 `Asset account`（资产账户）进行对账，其他类型账户直接跳转
2. **日期范围处理**：默认取当前会计期间，自动校正日期顺序，开始日期设为当天 `00:00:00`，结束日期设为 `23:59:59`
3. **余额计算**：调用 `Steam::accountsBalancesOptimized()` 分别计算起始余额和结束余额

#### 余额计算核心逻辑

`app/Support/Steam.php` 中的 `accountsBalancesOptimized()` 是 Firefly III 计算账户余额的核心方法，其本质是一条 **SQL 聚合查询**：

```sql
SELECT transactions.account_id, transaction_currencies.code,
       SUM(transactions.amount) as sum_of_amount
FROM transactions
LEFT JOIN transaction_journals ON transaction_journals.id = transactions.transaction_journal_id
LEFT JOIN transaction_currencies ON transaction_currencies.id = transactions.transaction_currency_id
WHERE transaction_journals.date [<= or <] :date           -- 注意这里的 inclusive 参数
  AND transaction_journals.deleted_at IS NULL
GROUP BY transactions.account_id, transaction_currencies.code
```

**关键设计**：
- `inclusive = false` 用于起始余额：只统计**严格早于**开始日期的交易（即期初余额）
- `inclusive = true`（默认）用于结束余额：统计**早于等于**结束日期的交易
- 计算完全基于 `transactions.amount` 的历史累加，**从未修改过任何历史交易的金额**

### 3.2 阶段二：获取对账期间交易列表

**入口方法**：`app/Http/Controllers/Json/ReconcileController.php` 中的 `transactions()`

- 查询范围会向前后各扩展 3 天（`selectionStart.subDays(3)` / `selectionEnd.addDays(3)`），确保边界交易可见
- 调用 `processTransactions()` 对交易金额做**显示用的符号调整**：
  - `Deposit`（存入）、转入该账户的 `Transfer`（转账）、`Opening Balance`（初始余额）→ 显示为正数
  - 其余 → 保持原符号（支出显示为负数）
  - ⚠️ **重要**：此调整仅用于前端展示便于对账，**不影响数据库存储**

### 3.3 阶段三：差额计算与概览展示

**入口方法**：`app/Http/Controllers/Json/ReconcileController.php` 中的 `overview()`

#### 差额计算公式（第 131 行）

```php
$difference = bcadd(bcadd(bcsub($startBalance, $endBalance), $clearedAmount), $amount);
```

展开为数学公式：

```
差额(Difference) = (期初余额 - 期末余额) + 已对账交易金额 + 本次选中交易金额
```

#### 各项含义解析

| 变量 | 含义 | 计算方式 |
|------|------|---------|
| `$startBalance` | 用户输入/系统计算的期初余额 | 来自页面输入 |
| `$endBalance` | 用户输入/系统计算的期末余额 | 来自页面输入 |
| `$clearedAmount` | 该期间内**已被标记对账**的交易对该账户的净影响 | 遍历 `clearedIds`，按源/目标账户确定符号 |
| `$amount` | 用户**本次勾选**的交易对该账户的净影响 | 遍历 `selectedIds`，按源/目标账户确定符号 |
| `$diffCompare = bccomp($difference, '0')` | 差额方向 | `>0` 钱少了，`<0` 钱多了，`=0` 正好 |

#### 交易金额的符号确定规则

`app/Http/Controllers/Json/ReconcileController.php` 中的 `processJournal()`：

```php
// 若对账账户是交易的 source → 该交易使账户金额减少（加正数）
if ($account->id === $journal['source_account_id']) {
    $toAdd = $journal['amount'];  // 正号，因为 transactions 表中 source 侧存负数，但此处含义是"减少"
}
// 若对账账户是交易的 destination → 该交易使账户金额增加（加负数抵消）
if ($account->id === $journal['destination_account_id']) {
    $toAdd = bcmul((string) $journal['amount'], '-1');  // destination 侧存正数，此处取反表示"增加"
}
```

### 3.4 阶段四：提交对账（核心操作）

**入口方法**：`app/Http/Controllers/Account/ReconcileController.php` 中的 `submit()`

提交分为**两个独立且顺序执行的步骤**：

#### 步骤 1：标记选中交易为已对账（保留历史交易的关键）

```php
foreach ($data['journals'] as $journalId) {
    $this->repository->reconcileById((int) $journalId);
}
```

深入到 `app/Repositories/Journal/JournalRepository.php` 中的 `reconcileById()`：

```php
public function reconcileById(int $journalId): void
{
    $journal = $this->user->transactionJournals()->find($journalId);
    $journal?->transactions()->update(['reconciled' => true]);  // ★ 仅更新 reconciled 布尔字段！
}
```

**关键结论**：
- ✅ **绝不修改历史交易的金额、日期、描述、对手账户等任何业务字段**
- 仅在 `transactions` 表上更新一个布尔列 `reconciled = true`
- 这是 Firefly III "保留历史交易"设计的核心体现

#### 步骤 2：条件性生成补偿调节交易（差额不为零时）

```php
if ('create' === $data['reconcile']) {       // 用户在前端选择了"创建对账交易"
    $result = $this->createReconciliation(
        $account, $start, $end, $data['difference']
    );
}
```

用户选项来自前端页面 `resources/views/accounts/reconcile/overview.twig`：
- `diffCompare == 0`（差额为零）：隐藏单选框，默认提交 `reconcile=nothing`
- `diffCompare != 0`（有差额）：提供两个单选选项
  - `value="create"`：创建补偿交易
  - `value="nothing"`：不创建（仅标记对账，差额保留）

---

## 四、补偿交易（Reconciliation Transaction）生成机制

### 4.1 生成入口与对手账户获取

**方法**：`app/Http/Controllers/Account/ReconcileController.php` 中的 `createReconciliation()`

第一步调用 `getReconciliation()` 获取或创建**对账调节账户**：

`app/Repositories/Account/AccountRepository.php` 中的 `getReconciliation()`：

```php
public function getReconciliation(Account $account): ?Account
{
    // 命名规则："(多语言的'对账账户') - {账户名} ({币种代码})"
    $name = trans('firefly.reconciliation_account_name', [
        'name'     => $account->name,
        'currency' => $currency->code
    ]);

    $type = AccountType::where('type', AccountTypeEnum::RECONCILIATION->value)->first();

    // 查找现有对账账户，不存在则创建
    $current = $this->user->accounts()
        ->where('account_type_id', $type->id)
        ->where('name', $name)
        ->first();

    if (null !== $current) return $current;

    // 通过 AccountFactory 创建一个类型为 RECONCILIATION 的新账户
    return $factory->create($data);
}
```

**设计意图**：每个资产账户 + 币种组合都对应一个独立的"对账调节账户"，作为补偿交易的对手方，避免污染正常的收入/支出/转账分类。

### 4.2 补偿交易方向的判定

```php
$source      = $reconciliation;   // 默认：对账账户 → 资产账户（钱从对账账户"调进来"）
$destination = $account;

if (1 === bccomp($difference, '0')) {   // difference > 0：实际钱比账面少
    $source      = $account;            // 反向：资产账户 → 对账账户（钱"调出去"消失）
    $destination = $reconciliation;
}
```

| 差额状态 | 含义 | 交易方向 | 效果 |
|---------|------|---------|------|
| `difference > 0` | 银行实际余额 < 账面余额（钱少了） | 资产账户 → 对账账户 | 资产账户余额减少，与银行对齐 |
| `difference < 0` | 银行实际余额 > 账面余额（钱多了） | 对账账户 → 资产账户 | 资产账户余额增加，与银行对齐 |
| `difference = 0` | 无差异 | 不生成补偿交易 | — |

### 4.3 补偿交易构造与提交

补偿交易通过 `app/Factory/TransactionGroupFactory.php` 创建，数据结构如下：

```php
$submission = [
    'user'        => auth()->user(),
    'user_group'  => auth()->user()->userGroup,
    'group_title' => null,
    'transactions' => [[
        'type'                => 'reconciliation',       // ★ 独立交易类型
        'date'                => $end,                   // ★ 日期 = 对账期间结束日
        'currency_id'         => $currency->id,
        'amount'              => $difference,            // ★ 金额 = 差额（注意绝对值方向由 source/dest 决定）
        'description'         => '对账交易：{开始日} 至 {结束日}',  // i18n 翻译
        'source_id'           => $source->id,            // 源账户
        'destination_id'      => $destination->id,       // 目标账户
        'reconciled'          => true,                   // ★ 补偿交易自身直接标记已对账
    ]],
];
```

### 4.4 补偿交易的落地存储

通过 `app/Factory/TransactionJournalFactory.php` 中的 `createJournal()` 落地：

1. **对账安全检查**（第 311-313 行）：`reconciliationSanityCheck()` 确保 source/destination 中至少有一个是对账调节账户，缺失则自动补齐
2. **创建 TransactionJournal**：`transaction_type_id` 对应 `Reconciliation` 类型
3. **创建两条 Transaction**（复式记账）：
   - 源账户侧：负金额，通过 `TransactionFactory::createNegative()`
   - 目标账户侧：正金额，通过 `TransactionFactory::createPositive()`
   - 两条记录的 `reconciled` 字段**都被设为 `true`**（第 352、370 行）
   - 见 `app/Factory/TransactionFactory.php` 中的 `create()`：`'reconciled' => $this->reconciled`

---

## 五、数据库表结构视角

理解两张核心表的变化：

### 5.1 transactions 表字段变化

| 字段 | 对账标记操作 | 补偿交易生成 |
|------|-------------|-------------|
| `id` | 不变 | 新增两条记录 |
| `transaction_journal_id` | 不变 | 指向新的 journal |
| `account_id` | 不变 | 分别指向资产账户和对账调节账户 |
| `amount` | **不变** ✅ | 一负一正，金额 = 差额 |
| `reconciled` | `false` → `true` ✨ | 直接写入 `true` ✨ |
| 其他字段 | **全部不变** ✅ | 写入相应元数据 |

### 5.2 transaction_journals 表变化

| 字段 | 对账标记操作 | 补偿交易生成 |
|------|-------------|-------------|
| 已有 journals | 无任何 UPDATE | — |
| 新增 journal | — | `transaction_type_id` = Reconciliation，`date` = 对账期末 |

---

## 六、反对账（撤销对账）

**入口**：`app/Http/Controllers/Transaction/EditController.php` 中的 `unreconcile()` → `app/Repositories/Journal/JournalRepository.php` 中的 `unreconcileById()`

```php
public function unreconcileById(int $journalId): void
{
    $journal = $this->user->transactionJournals()->find($journalId);
    $journal?->transactions()->update(['reconciled' => false]);  // 同样只更新布尔字段
}
```

⚠️ **注意**：反对账操作**不会自动删除**之前生成的补偿交易（Reconciliation Transaction）。补偿交易作为独立的交易记录保留在系统中，用户需手动删除——这符合"交易不可变、历史可追溯"的审计原则。

---

## 七、完整流程时序图

```
用户进入对账页面
    │
    ▼
Account\ReconcileController@reconcile()
    │  ├─ 校验账户类型为 Asset
    │  ├─ Steam::accountsBalancesOptimized() 计算期初余额 (inclusive=false)
    │  └─ Steam::accountsBalancesOptimized() 计算期末余额 (inclusive=true)
    │
用户点击"开始对账"
    │
    ▼
Json\ReconcileController@transactions()          ◄── AJAX
    │  └─ 返回期间交易列表（±3 天边界）
    │
用户勾选交易，前端显示差额
    │
    ▼
Json\ReconcileController@overview()              ◄── AJAX
    │  ├─ processJournal() 计算选中交易净影响
    │  ├─ 公式: diff = (期初 - 期末) + 已对账金额 + 选中金额
    │  └─ 返回差额 + 提交表单 HTML
    │
用户确认提交 (diff ≠ 0 时选择"创建补偿交易")
    │
    ▼
Account\ReconcileController@submit()             ◄── POST
    │
    ├─ STEP 1: 标记交易（历史保留核心）
    │      └─ JournalRepository::reconcileById()
    │              └─ UPDATE transactions SET reconciled=true
    │                 WHERE transaction_journal_id = :id
    │                 ⚡ 仅更新 reconciled 字段，其余不变！
    │
    └─ STEP 2: 生成补偿交易（iff reconcile='create' 且 diff≠0）
           ├─ AccountRepository::getReconciliation()
           │      └─ 查找或创建 "Reconciliation account" 类型账户
           ├─ 根据 diff 符号确定 source/destination
           ├─ TransactionGroupFactory::create()
           │      └─ TransactionJournalFactory::createJournal()
           │              ├─ 检查账户合理性 reconciliationSanityCheck()
           │              ├─ INSERT transaction_journal (type=Reconciliation)
           │              ├─ TransactionFactory::createNegative()  (源账户, reconciled=true)
           │              └─ TransactionFactory::createPositive()  (目标账户, reconciled=true)
           └─ 完成！余额通过新增的两条 Transaction 记录被修正
```

---

## 八、关键设计亮点总结

### ✅ 8.1 历史交易零修改

Firefly III 严格遵循**会计不可变性原则**：
- 对账操作**绝不**修改历史交易的 `amount`（金额）、`date`（日期）、`description`（描述）等任何业务字段
- 仅在 `transactions` 表上切换一个布尔标记 `reconciled`
- 所有对账单据（银行流水）可追溯、可审计、可复核

### ✅ 8.2 补偿交易机制（余额修正的核心）

当存在差额时：
1. **不调整历史交易金额**，而是**新增**一条独立的 `Reconciliation` 类型交易
2. 对手方使用专门的 `Reconciliation account`（对账调节账户），与收入/支出账户隔离
3. 补偿交易日期 = 对账期末日，保证余额计算在该日期后与银行对齐
4. 补偿交易自身自动标记为 `reconciled = true`，不会在下次对账时重复出现

### ✅ 8.3 余额的纯聚合计算

余额 = `SUM(transactions.amount)` 动态聚合：
- 新增补偿交易 → 两条新 Transaction（一正一负）→ 资产账户侧的金额方向与差额一致 → 汇总后余额自然被修正
- 无需在账户表中缓存或回写"余额"字段，余额永远是交易的**函数**，避免数据不一致

### ✅ 8.4 复式记账保证平衡

每笔补偿交易：
```
资产账户 (Δ)  ←  →  对账调节账户 (Δ)
    -X                    +X        (差额为正：钱少了，从资产账户流出到对账账户)
    +X                    -X        (差额为负：钱多了，从对账账户流入到资产账户)
```

无论哪种方向，两笔 Transaction 的 SUM 永远为零，系统整体借贷平衡保持不变。

---

## 九、回答用户核心问题

### Q1：如何在保留历史交易的前提下完成余额修正？

**答**：
1. **历史交易层面**：仅对用户勾选的交易执行 `UPDATE transactions SET reconciled = true WHERE transaction_journal_id IN (...)`。**不触碰金额、日期、对手方等任何业务字段**，历史数据 100% 保留。
2. **余额修正层面**：通过**新增**而非修改方式完成。若对账差额不为零，系统创建一条类型为 `Reconciliation` 的全新交易（包含两条 Transaction 记录），日期记为对账期末日。该交易的金额方向自动匹配差额方向，资产账户侧的新增 Transaction 会在后续余额聚合时生效，从而在不动旧数据的前提下，让期末之后的余额与银行实际一致。

### Q2：期间是否会生成补偿交易？

**答**：**条件性生成**，取决于差额和用户选择：
- 差额 = 0：**不生成**。提交参数自动带 `reconcile=nothing`
- 差额 ≠ 0：**由用户在前端二选一**
  - 选 `create`：生成补偿交易（推荐选项，前端文本会明确告知"创建正/负对账调节交易，金额 XXX"）
  - 选 `nothing`：不生成，仅标记对账，差额保留（可能影响后续报表准确性）

补偿交易具有以下特征：
- 类型标识：`TransactionTypeEnum::RECONCILIATION`
- 对手账户：专用的 `Reconciliation account`（按资产账户+币种自动创建/获取）
- 日期：对账期间的结束日 `$end->endOfDay()`
- 对账标记：自身立即 `reconciled=true`，避免下次重复对账
- 可追溯：作为独立交易存在，可在交易列表中筛选查看，可单独删除但不影响原交易

---

## 十、提交阶段事务控制与回滚机制深度分析

### 10.1 submit() 方法的执行顺序与无事务特征

重新审视 [ReconcileController::submit()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Controllers/Account/ReconcileController.php#L169-L204) 的完整控制流：

```php
public function submit(ReconciliationStoreRequest $request, Account $account, Carbon $start, Carbon $end): RedirectResponse
{
    Log::debug('In ReconcileController::submit()');
    $data = $request->getAll();

    // ═══════════════════════════════════════════════════════════
    // STEP 1: 逐条标记交易（每条 UPDATE 立即生效，无法回滚）
    // ═══════════════════════════════════════════════════════════
    foreach ($data['journals'] as $journalId) {
        $this->repository->reconcileById((int) $journalId);
    }
    Log::debug('Reconciled all transactions.');

    // 日期校正
    if ($end->lt($start)) {
        [$start, $end] = [$end, $start];
    }

    // ═══════════════════════════════════════════════════════════
    // STEP 2: 创建补偿交易（可能失败）
    // ═══════════════════════════════════════════════════════════
    $result = '';
    if ('create' === $data['reconcile']) {
        $result = $this->createReconciliation($account, $start, $end, $data['difference']);
    }

    // 结果反馈（无论 STEP 2 是否成功，STEP 1 的修改都已持久化）
    if ('' === $result) {
        session()->flash('success', (string) trans('firefly.reconciliation_stored'));
    }
    if ('' !== $result) {
        session()->flash('error', (string) trans('firefly.reconciliation_error', ['error' => $result]));
    }

    return redirect(route('accounts.show', [$account->id]));
}
```

**代码层验证无事务**：

1. 无数据库事务包装：`ReconcileController` 继承的基础 [Controller.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Controllers/Controller.php) 中**没有**任何 `DB::transaction()`、`DB::beginTransaction()`、`DB::commit()`、`DB::rollback()` 调用。
2. [ReconcileController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Controllers/Account/ReconcileController.php) 自身也**不包含**任何事务相关代码。
3. 每条 `reconcileById()` 在 MySQL InnoDB 中默认以 **auto-commit** 模式执行：一条 `UPDATE` 语句立即提交并持久化。

### 10.2 createReconciliation() 的异常捕获与错误返回

查看 [createReconciliation()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Controllers/Account/ReconcileController.php#L211-L270) 的错误处理：

```php
private function createReconciliation(Account $account, Carbon $start, Carbon $end, string $difference): string
{
    // ...省略准备代码...

    try {
        $factory->create($submission);      // 可能抛出多种异常
    } catch (FireflyException $e) {        // ⚠️ 只捕获 FireflyException！
        return $e->getMessage();            // 返回错误字符串
    }

    return '';
}
```

⚠️ **关键细节**：方法签名标注了 `@throws DuplicateTransactionException`，但 `try-catch` **仅捕获 `FireflyException`**。而 [DuplicateTransactionException](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Exceptions/DuplicateTransactionException.php#L32) 继承自原生 `\Exception`，**不是** `FireflyException` 的子类。

### 10.3 重复交易异常（DuplicateTransactionException）激活条件深度分析

这是一个容易被误解的点：**对账流程中的补偿交易创建，默认不会触发重复交易检查**。

#### 10.3.1 触发开关：`errorOnHash` 属性

[TransactionJournalFactory](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Factory/TransactionJournalFactory.php#L75) 中有一个关键开关：

```php
private bool $errorOnHash = false;   // 默认关闭！
```

这个开关由 `setErrorOnHash()` 方法设置，在 [TransactionGroupFactory::create()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Factory/TransactionGroupFactory.php#L62) 第 62 行被赋值：

```php
$this->journalFactory->setErrorOnHash($data['error_if_duplicate_hash'] ?? false);
```

而对账补偿交易的 `$submission` 数组（[createReconciliation() 第 235-254 行](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Controllers/Account/ReconcileController.php#L235-L254)）中 **完全没有 `error_if_duplicate_hash` 字段**。

**结论**：`errorIfDuplicate()` 方法第 422 行的判断 `if (false === $this->errorOnHash) { return; }` 会直接短路返回，**对账补偿交易不会执行重复检测**，`DuplicateTransactionException` 在对账流程中**不会被主动抛出**。

#### 10.3.2 若被激活时的异常传递路径（假设 `errorOnHash=true`）

虽然对账默认不触发，但为完整起见，梳理一旦激活后的完整传递链：

```
调用层级：
  createReconciliation()
    └─ TransactionGroupFactory::create()
         └─ TransactionJournalFactory::create()
              └─ createJournal()
                   └─ errorIfDuplicate()  ← 这里抛出 DuplicateTransactionException
```

逐层向上的处理：

| 层级 | 捕获情况 | 清理动作 | 后续行为 |
|------|---------|---------|---------|
| ① `createJournal()` | ❌ 无捕获 | 无 | 直接向上抛出 |
| ② `TransactionJournalFactory::create()` 第 134 行 | ✅ 有捕获 | `forceDeleteOnError($collection)` 删除已创建的 journals | 重新 `throw new DuplicateTransactionException(...)` |
| ③ `TransactionGroupFactory::create()` 第 66 行 | ✅ 有捕获 | 无（因为 journal 层已清理） | 重新 `throw new DuplicateTransactionException(...)` |
| ④ `createReconciliation()` 第 264 行 | ❌ **漏捕获**！（只 catch FireflyException） | 无 | 继续向上冒泡 |
| ⑤ `submit()` | ❌ 无捕获 | 无（STEP 1 对账标记已提交） | 继续向上冒泡 |
| ⑥ Laravel 全局异常处理器 | ✅ 兜底捕获 | 无 | 返回 500 错误页面 |

**漏捕获后果**：STEP 1 标记的对账流水已持久化 → 补偿交易在 journal 层被清理（无残留）→ 用户看到 500 错误 → 对账标记生效但余额未修正（同"半对账"不一致状态）

#### 10.3.3 重复检测的哈希计算与匹配规则

[hashArray() 第 524-538 行](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Factory/TransactionJournalFactory.php#L524-L538) 将整行交易数据做 SHA-256 哈希：

```php
private function hashArray(NullArrayObject $row): string
{
    unset($row['import_hash_v2'], $row['original_source']);
    $json = json_encode($row, JSON_THROW_ON_ERROR);
    return hash('sha256', $json);
}
```

[errorIfDuplicate() 第 419-444 行](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Factory/TransactionJournalFactory.php#L419-L444) 的匹配查询：

```sql
SELECT journal_meta.*
FROM journal_meta
LEFT JOIN transaction_journals ON transaction_journals.id = journal_meta.transaction_journal_id
WHERE transaction_journals.user_id = :userId
  AND journal_meta.data = :hashValue
  AND transaction_journals.deleted_at IS NULL
LIMIT 1
```

- 匹配基于 `transaction_journal_meta` 表中名为 `import_hash_v2` 的元数据
- 范围覆盖当前用户所有 journal（包含已软删除的？不，`WHERE transaction_journals.deleted_at IS NULL` 排除了软删除）
- 命中则抛出异常，消息格式为 `"Duplicate of transaction #%d."`

### 10.4 各类失败场景的一致性状态矩阵

| 场景 | STEP 1 标记 | STEP 2 补偿 | 数据库状态 | 结果 |
|------|------------|------------|-----------|------|
| ✅ 正常情况 | 全部完成 | 创建成功 | **一致** | 显示 success |
| ❌ 场景A：`create()` 抛出 `FireflyException` | ✅ 已永久提交 | ❌ 未创建 | **不一致** ⚠️ | 显示 error，**标记已对账但余额未修正** |
| ❌ 场景B：`create()` 抛出 `DuplicateTransactionException` | ✅ 已永久提交 | ❌ 未创建 | **不一致** ⚠️ + 崩溃 | Laravel 异常处理器捕获，报 500 错误，**用户看不到错误提示** |
| ❌ 场景C：`create()` 抛出其他 `Exception`（DB、网络等） | ✅ 已永久提交 | ❌ 未创建 | **不一致** ⚠️ + 崩溃 | 500 错误，状态同上 |
| ❌ 场景D：`reconcileById()` 中途某条抛异常（如 DB 连接断开） | ✅ 已循环到的已提交 | ❌ 未执行 | **部分不一致** ⚠️ | 500 错误，一部分交易已标记，另一部分未标记 |

### 10.4 不一致状态的用户视角与后果

以"场景A"为例（最可能的不一致情况，如账户验证失败等）：
1. 用户提交对账 → flash 显示 `reconciliation_error` 提示
2. 跳转回账户详情页
3. **用户看到的现象**：
   - 勾选的 N 条交易显示 "Reconciled" 对勾标记 ✅
   - 账户余额**与银行实际余额仍不一致**（因为补偿交易未创建）
   - 交易列表里**找不到**预期的"对账调节交易"记录
4. **用户可能的错误操作**：
   - 误以为提交失败，重新勾选同样的交易再次提交 → 重复标记（幂等无副作用，但差额还在）
   - 手动"创建一条差异交易" → 可能金额/方向搞错，或者下次对账把这条也勾进去导致二次补偿
   - 以为系统 bug，去反对账所有交易 → 大量手工操作

### 10.5 TransactionJournalFactory 内部的局部事务（不影响外部）

虽然 submit() 没有整体事务，但 [TransactionJournalFactory::createJournal()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Factory/TransactionJournalFactory.php) 内部在补偿交易的创建过程中**自身有原子性保障**：

```php
// createJournal() 中，若创建源侧 Transaction 失败：
} catch (FireflyException $e) {
    $this->forceDeleteOnError(new Collection()->push($journal));  // 删除已创建的 journal
    throw new FireflyException(...);
}

// 若创建目标侧 Transaction 失败：
} catch (FireflyException $e) {
    $this->forceTrDelete($negative);                           // 删除已创建的负侧 Transaction
    $this->forceDeleteOnError(new Collection()->push($journal));  // 删除 journal
    throw new FireflyException(...);
}
```

调用链 `forceDeleteOnError()` → `JournalDestroyService::destroy()`，确保补偿交易创建到一半失败时**不会留下半个交易数据**。

但这个局部保障**仅限于补偿交易内部**，不会回滚 submit() STEP 1 中已标记的对账流水。

### 10.6 补偿交易创建各阶段失败的残留步骤深度分析

前述的"局部事务保障"只覆盖了两条 Transaction 创建的阶段。实际上补偿交易创建是一个**多阶段流水线**，越靠后的阶段失败，残留的数据越多。以下按执行顺序逐阶段拆解。

#### 10.6.1 阶段总览：创建流水线与清理覆盖范围

补偿交易创建涉及三层工厂的协作，完整流水线如下：

```
createReconciliation()
    │
    └─ TransactionGroupFactory::create()
         │  （第三层：组包装）
         │
         └─ TransactionJournalFactory::create()  ← 有 try-catch + forceDeleteOnError
              │  （第二层：日记账循环）
              │
              └─ createJournal()  ← 仅部分阶段有 catch 清理
                   │  （第一层：日记账核心构造）
                   │
                   ├─ P1: hashArray + errorIfDuplicate
                   ├─ P2: validateAccounts
                   ├─ P3: findCurrency / findBill
                   ├─ P4: getAccount × 2（source + destination）
                   ├─ P5: TransactionJournal::create()  ← journal 入库
                   ├─ P6: TransactionFactory::createNegative()  ← 负侧 Transaction 入库
                   ├─ P7: TransactionFactory::createPositive()  ← 正侧 Transaction 入库
                   ├─ P8: $journal->save()
                   ├─ P9: storeBudget()
                   ├─ P10: storeCategory()
                   ├─ P11: storeNotes()
                   ├─ P12: storePiggyEvent()
                   ├─ P13: storeTags()
                   ├─ P14: storeMetaFields()
                   └─ P15: storeLocation()

         ─────── create() 返回后，进入 GroupFactory ───────
                   │
                   ├─ G1: 空集合检查（0 === count）
                   ├─ G2: TransactionGroup::save()  ← group 入库
                   ├─ G3: transactionJournals()->saveMany()  ← 关联 group 和 journals
                   └─ 返回 group
```

**清理逻辑的覆盖盲区**：
- ✅ P5-P7 失败：有 `catch (FireflyException)` → `forceDeleteOnError` 清理
- ❌ P8-P15 失败：**没有任何 try-catch** → 异常直接向上冒泡
- ❌ G1-G3 失败：**没有任何 try-catch** → 异常直接向上冒泡
- 而且 `createReconciliation()` 只捕获 `FireflyException`，非 FireflyException 的异常（如 PDOException、QueryException）**全部漏捕获**

#### 10.6.2 P5~P7 失败：有清理，无残留（已知保障）

对应代码第 352-398 行（`createNegative` 和 `createPositive` 周围的 catch 块）：

| 失败点 | 已创建数据 | catch 中的清理动作 | 最终残留 |
|-------|-----------|-------------------|---------|
| P5 `TransactionJournal::create()` | —（还没创建成功） | — | 无 |
| P6 `createNegative()` 失败 | 1 个 journal | `forceDeleteOnError([journal])` → 调 JournalDestroyService::destroy() 删除 journal + 级联删 transactions | **无残留** |
| P7 `createPositive()` 失败 | 1 个 journal + 1 条负 Transaction | ① `forceTrDelete($negative)` 删负 Transaction ② `forceDeleteOnError([journal])` 删 journal | **无残留** |

**注意**：这里的清理仅对 `FireflyException` 有效。如果 TransactionFactory 内部抛出的是 `\PDOException` 或 `QueryException`（非 FireflyException 子类），则不会被 catch 捕获，**残留会和后续阶段一样严重**。

#### 10.6.3 P8~P15 逐步写入流程与失败残留详解

P8 到 P15 是 [createJournal()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Factory/TransactionJournalFactory.php#L400-L407) 方法的尾部步骤（第 400-407 行），**全部在 try-catch 之外**，任何一步抛出异常都无清理逻辑。以下逐步骤精确追踪写入的数据表和可能抛出的异常类型。

##### P8: `$journal->save()` — journal 二次保存

```php
// TransactionJournalFactory.php 第 400 行
$journal->save();
```

- **写入的表**：`transaction_journals`（UPDATE，journal 已在 P5 创建并 INSERT）
- **实际变更**：更新 `updated_at` 时间戳；更新 P5~P7 过程中可能被改变的属性（如 `completed` 字段——在第 341 行设置 `'completed' => is_bool($row['batch_submission']) && !$row['batch_submission']`，但对账交易的 `batch_submission` 默认为 false，所以 `completed = true`）
- **可能抛出的异常**：`QueryException`（如数据库连接断开、字段长度溢出等），**不是** `FireflyException`
- **若失败残留**：journal 已存在于 DB（P5 创建），两条 Transaction 已存在（P6/P7 创建）→ 余额已被改变但无 group 无元数据

##### P9: `storeBudget()` — 预算关联

```php
// JournalServiceTrait.php 第 154-170 行
protected function storeBudget(TransactionJournal $journal, NullArrayObject $data): void
{
    if (TransactionTypeEnum::WITHDRAWAL->value !== $journal->transactionType->type) {
        $journal->budgets()->sync([]);  // ← 对账交易走这条分支！
        return;
    }
    // ... 以下对账交易不会执行
}
```

- **写入的表**：`budget_transaction_journal`（pivot 表），执行 `sync([])`
- **对账交易的实际行为**：类型为 `Reconciliation` ≠ `Withdrawal`，所以直接 `sync([])`（空数组），即确保无预算关联
- **可能抛出的异常**：`QueryException`（pivot 表写入失败），**不是** `FireflyException`
- **若失败残留**：journal + 2 条 transactions 已入库（P5-P7 的成果保留）

##### P10: `storeCategory()` — 分类关联

```php
// JournalServiceTrait.php 第 172-183 行
protected function storeCategory(TransactionJournal $journal, NullArrayObject $data): void
{
    $category = $this->categoryRepository->findCategory($data['category_id'], $data['category_name']);
    if (null !== $category) {
        $journal->categories()->sync([$category->id]);
        return;
    }
    $journal->categories()->sync([]);  // ← 对账交易走这条分支！
}
```

- **写入的表**：`category_transaction_journal`（pivot 表），执行 `sync([])`
- **对账交易的实际行为**：`category_id` 和 `category_name` 均为 null → `findCategory` 返回 null → `sync([])`
- **可能抛出的异常**：`QueryException`
- **若失败残留**：journal + 2 条 transactions + P9 的 budgets sync 结果

##### P11: `storeNotes()` — 备注写入

```php
// JournalServiceTrait.php 第 185-202 行
protected function storeNotes(TransactionJournal $journal, ?string $notes): void
{
    $notes = (string) $notes;
    $note  = $journal->notes()->first();
    if ('' !== $notes) {
        if (null === $note) {
            $note = new Note();
            $note->noteable()->associate($journal);
        }
        $note->text = $notes;
        $note->save();
        return;
    }
    $note?->delete();  // ← 对账交易走这条分支（notes 为空）
}
```

- **写入的表**：`notes`（Note 模型）
- **对账交易的实际行为**：`$notes` 参数为 null → `(string)null = ''` → 走 delete 分支 → journal 无已有 note → 无操作
- **可能抛出的异常**：极端情况下 `QueryException`
- **若失败残留**：journal + 2 条 transactions + P9-P10 的 sync 结果

##### P12: `storePiggyEvent()` — 储蓄罐事件

```php
// TransactionJournalFactory.php 第 606-619 行
private function storePiggyEvent(TransactionJournal $journal, NullArrayObject $data): void
{
    $piggyBank = $this->piggyRepository->findPiggyBank((int) $data['piggy_bank_id'], $data['piggy_bank_name']);
    if ($piggyBank instanceof PiggyBank) {
        $this->piggyEventFactory->create($journal, $piggyBank);
        return;
    }
    Log::debug('Create no piggy event');  // ← 对账交易走这条分支
}
```

- **写入的表**：`piggy_bank_events`
- **对账交易的实际行为**：`piggy_bank_id` 和 `piggy_bank_name` 均为 null → `findPiggyBank` 返回 null → 跳过
- **可能抛出的异常**：无（本步骤对账交易跳过）
- **若失败残留**：不适用于对账场景

##### P13: `storeTags()` — 标签关联

```php
// JournalServiceTrait.php 第 207-237 行
protected function storeTags(TransactionJournal $journal, ?array $tags): void
{
    // ...
    if (!is_array($tags)) {
        Log::debug('Tags is not an array, break.');
        return;                           // ← 对账交易走这条分支！
    }
    // ...
    try {
        $journal->tags()->sync($set);
    } catch (UniqueConstraintViolationException $e) {
        Log::error(sprintf('Firefly III could not sync tags: %s', $e->getMessage()));
    }
}
```

- **写入的表**：`tag_transaction_journal`（pivot 表）
- **对账交易的实际行为**：`$tags` 为 null → `!is_array(null)` = true → 直接 return，无任何 DB 操作
- **可能抛出的异常**：无（本步骤对账交易直接跳过）
- **若失败残留**：不适用于对账场景
- **⚠️ 设计亮点**：即使走到 `sync()` 分支，也有 `catch (UniqueConstraintViolationException)` 兜底，**静默失败不抛出**，是 P8-P15 中唯一自带容错处理的步骤

##### P14: `storeMetaFields()` — 元数据逐字段写入

```php
// TransactionJournalFactory.php 第 596-601 行
private function storeMetaFields(TransactionJournal $journal, NullArrayObject $transaction): void
{
    foreach ($this->fields as $field) {             // $this->fields = config('firefly.journal_meta_fields')
        $this->storeMeta($journal, $transaction->getArrayCopy(), $field);
    }
}

// TransactionJournalFactory.php 第 189-203 行
protected function storeMeta(TransactionJournal $journal, array $data, string $field): void
{
    $set = ['journal' => $journal, 'name' => $field, 'data' => (string) ($data[$field] ?? '')];
    // ...
    $factory = app(TransactionJournalMetaFactory::class);
    $factory->updateOrCreate($set);                // ← 逐字段调用
}
```

[TransactionJournalMetaFactory::updateOrCreate()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Factory/TransactionJournalMetaFactory.php#L36-L77) 的逻辑：

```php
public function updateOrCreate(array $data): ?TransactionJournalMeta
{
    $value = $data['data'];
    $entry = $data['journal']->transactionJournalMeta()->where('name', $data['name'])->first();

    if (null === $value && null !== $entry) { $entry->delete(); return null; }
    if ('' === (string) $value) {
        if (null !== $entry) { $entry->delete(); }
        return null;             // ← 大部分字段的空值走这条分支，不写 DB
    }

    if (null === $entry) {
        $entry = new TransactionJournalMeta();
        $entry->transactionJournal()->associate($data['journal']);
        $entry->name = $data['name'];
    }
    $entry->data = $value;
    $entry->save();              // ← 只有非空值才会写 DB

    return $entry;
}
```

- **写入的表**：`journal_meta`（`transaction_journal_meta` 表）
- **对账交易实际写入的字段**：取决于 `config('firefly.journal_meta_fields')` 和提交数据
  - `import_hash_v2`：**始终写入**（在 P1 的 `hashArray()` 中生成，值非空）
  - `original_source`：可能写入
  - 其他字段（`recurrence_id`、`sepa_*`、`external_id` 等）：对账交易通常为空 → 跳过不写
- **关键行为**：`foreach` 循环逐字段处理，每个字段是独立的 `updateOrCreate()` 调用
- **可能抛出的异常**：`QueryException`（任一字段写入时 DB 错误），**不是** `FireflyException`
- **若失败残留**：失败前已写入的 meta 字段保留在 DB 中，后续字段缺失
- **最关键的字段**：`import_hash_v2` 是重复交易检测的依据（参见 10.3.3 节），如果 P14 失败导致此字段未写入，则该 journal 不会被重复检测命中

##### P15: `storeLocation()` — 地理位置写入

```php
// TransactionJournalFactory.php 第 584-594 行
private function storeLocation(TransactionJournal $journal, NullArrayObject $data): void
{
    if (!in_array(null, [$data['longitude'], $data['latitude'], $data['zoom_level']], true)) {
        $location = new Location();
        $location->longitude = $data['longitude'];
        $location->latitude = $data['latitude'];
        $location->zoom_level = $data['zoom_level'];
        $location->locatable()->associate($journal);
        $location->save();
    }
    // ← 对账交易三个值均为 null → in_array(null, [...], true) = true → 跳过
}
```

- **写入的表**：`locations`
- **对账交易的实际行为**：经纬度均为 null → 条件不满足 → 完全跳过
- **可能抛出的异常**：无（本步骤对账交易跳过）

##### P8~P15 对账场景的实际写入摘要

由于对账交易（`Reconciliation` 类型）的提交数据中大部分关联字段为空，实际写入操作远少于普通交易：

| 步骤 | 对账交易实际操作 | 是否写 DB | 失败可能性 |
|-----|----------------|----------|-----------|
| P8 `$journal->save()` | UPDATE journal（刷新 updated_at, completed=true） | ✅ 是 | 低（但 DB 异常可发生） |
| P9 `storeBudget()` | `sync([])` 确保 Reconciliation 类型无预算 | ✅ 是（写空） | 低 |
| P10 `storeCategory()` | `sync([])` 确保无分类 | ✅ 是（写空） | 低 |
| P11 `storeNotes()` | 无操作（notes 为空，journal 也无旧 note） | ❌ 否 | 无 |
| P12 `storePiggyEvent()` | 跳过（无储蓄罐关联） | ❌ 否 | 无 |
| P13 `storeTags()` | 跳过（tags 为 null） | ❌ 否 | 无 |
| P14 `storeMetaFields()` | 写入 `import_hash_v2` 等非空字段 | ✅ 是 | **中等**（循环写入多个字段） |
| P15 `storeLocation()` | 跳过（无地理位置） | ❌ 否 | 无 |

**结论**：对账场景下 P8-P15 中真正有 DB 写入风险的步骤是 **P8、P9、P10、P14**，其中 P14（元数据写入）是最可能因 DB 异常而失败的步骤。

##### P8~P15 失败后的残留状态

| 数据表 | 是否残留 | 说明 |
|-------|---------|------|
| `transaction_journals` | ✅ 1 条 | P5 创建，P8 可能更新了部分字段 |
| `transactions` | ✅ 2 条 | P6/P7 创建，源侧负金额 + 目标侧正金额 |
| `budget_transaction_journal` | 部分 | P9 的 `sync([])` 可能已执行 |
| `category_transaction_journal` | 部分 | P10 的 `sync([])` 可能已执行 |
| `notes` | ❌ 无 | P11 对账交易不写 note |
| `tag_transaction_journal` | ❌ 无 | P13 对账交易不写 tag |
| `journal_meta` | 部分 | P14 逐字段写入，`import_hash_v2` 可能已写入，后续字段可能缺失 |
| `locations` | ❌ 无 | P15 对账交易不写 location |
| `transaction_groups` | ❌ 无 | 还没到 Group 阶段 |

##### P8~P15 异常传递链

```
P8/P9/P10/P14 抛出 QueryException 或 PDOException
  → createJournal() 无 try-catch（P8-P15 在 catch 块之外）→ 直接冒泡
  → TransactionJournalFactory::create() 第 141 行
      → catch (FireflyException $e): ❌ 类型不匹配，漏捕获
      → catch (DuplicateTransactionException $e): ❌ 类型不匹配，漏捕获
  → TransactionGroupFactory::create(): ❌ 无任何 catch
  → createReconciliation() 第 264 行: catch (FireflyException $e): ❌ 类型不匹配，漏捕获
  → submit(): ❌ 无 catch
  → Laravel 全局异常处理器: 返回 500 错误页面
```

**⚠️ 关键发现**：P8-P15 抛出的 `QueryException` / `PDOException` 在**整个调用链中没有任何一层能捕获**，必然导致 500 错误。而且 [TransactionJournalFactory::create()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Factory/TransactionJournalFactory.php#L134-L148) 的 catch 块只覆盖 `DuplicateTransactionException` 和 `FireflyException`，其他异常**既不会被清理也不会被转换**。

#### 10.6.4 G1~G3 Group 关联阶段：saveMany 机制与失败残留详解

journals 创建成功后，回到 [TransactionGroupFactory::create()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Factory/TransactionGroupFactory.php#L57-L90) 执行组包装。这一阶段**完全没有 try-catch 和清理逻辑**。

##### G1: 空集合检查

```php
// TransactionGroupFactory.php 第 77-79 行
if (0 === $collection->count()) {
    throw new FireflyException('Created zero transaction journals.');
}
```

- **前提**：对账交易只有 1 条 journal，`count() = 1`，**不会进入此分支**
- **此步骤对对账交易无风险**

##### G2: `$group->save()` — 创建 TransactionGroup

```php
// TransactionGroupFactory.php 第 81-85 行
$group = new TransactionGroup();
$group->user()->associate($this->user);
$group->userGroup()->associate($this->userGroup);
$group->title = $title;        // 对账交易的 group_title = null
$group->save();                // ← INSERT INTO transaction_groups
```

- **写入的表**：`transaction_groups`
- **写入内容**：`user_id`、`user_group_id`、`title = null`、`created_at`、`updated_at`
- **可能抛出的异常**：`QueryException`（DB 连接失败、磁盘满等）
- **若失败残留**：
  - `transaction_groups`：❌ 未写入（save 失败）
  - `transaction_journals`：✅ 1 条（P5 创建），但 `transaction_group_id = null`（尚未关联）
  - `transactions`：✅ 2 条（P6/P7 创建）
  - `journal_meta` 等：✅ P14 写入的部分元数据

##### G3: `$group->transactionJournals()->saveMany($collection)` — 关联 journals 到 group

```php
// TransactionGroupFactory.php 第 87 行
$group->transactionJournals()->saveMany($collection);
```

这是 Laravel Eloquent 的 `saveMany()` 方法，属于 `HasMany` 关系。其内部机制：

1. 遍历 `$collection` 中的每个 `TransactionJournal`
2. 对每个 journal 调用 `$journal->save()`，在 save 之前设置 `$journal->transaction_group_id = $group->id`
3. 每个 save 是独立的 SQL UPDATE

**对账场景**：collection 中只有 1 条 journal，所以 `saveMany()` 只会执行 1 次 UPDATE：

```sql
UPDATE transaction_journals
SET transaction_group_id = :groupId, updated_at = :timestamp
WHERE id = :journalId
```

- **写入的表**：`transaction_journals`（UPDATE，设置 `transaction_group_id`）
- **可能抛出的异常**：`QueryException`
- **若失败残留**：
  - `transaction_groups`：✅ 1 条（G2 已保存）
  - `transaction_journals`：✅ 1 条，但 `transaction_group_id` 可能为 null（saveMany 失败前）或已设置（saveMany 成功但后续 journal 处理失败——对账只有 1 条 journal，不存在此情况）
  - `transactions`：✅ 2 条

##### G2/G3 失败的完整性风险深度分析

**外键约束**：根据迁移文件 [2019_03_22_183214_changes_for_v480.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/database/migrations/2019_03_22_183214_changes_for_v480.php#L124) 第 124 行：

```php
$table->foreign('transaction_group_id')
      ->references('id')->on('transaction_groups')
      ->onDelete('cascade');
```

`transaction_journals.transaction_group_id` 指向 `transaction_groups.id` 的外键设置了 **`onDelete('cascade')`**。这意味着：

- ✅ 如果 group 被删除，其下所有 journal 会被数据库级联删除
- ❌ 但 **`transaction_group_id = null` 的 journal 不受此约束影响**——外键只对非 null 值生效

**"孤儿 journal" 的具体含义**：

若 G2 失败（group 未创建），journal 的 `transaction_group_id` 仍为 P5 创建时的初始值。查看 [TransactionJournal](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Models/TransactionJournal.php#L60-L72) 的 `$fillable`：

```php
protected $fillable = [
    'user_id', 'user_group_id', 'transaction_type_id',
    'bill_id', 'tag_count', 'transaction_currency_id',
    'description', 'completed', 'order', 'date', 'date_tz',
];
```

**`transaction_group_id` 不在 `$fillable` 中**！也就是说 P5 的 `TransactionJournal::create([...])` 不会设置此字段，其值取决于数据库列的默认值（通常为 null）。

**孤儿 journal 的 UI 影响**：

- Firefly III 的账户详情页（`accounts.show`）通过查询 `transaction_groups` 来展示交易列表
- 孤儿 journal 不属于任何 group → **不会出现在交易列表中**
- 但 [Steam::accountsBalancesOptimized()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Support/Steam.php#L72-L157) 直接查 `transactions` 表 → **余额计算仍然包含此 journal 的两条 Transaction**
- **后果**：余额被默默改变，UI 上看不到对应交易记录，用户无法通过正常操作定位这笔补偿交易

**DeletedTransactionGroupObserver 的级联删除机制**：

[DeletedTransactionGroupObserver](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Handlers/Observer/DeletedTransactionGroupObserver.php#L34-L40)：

```php
public function deleting(TransactionGroup $transactionGroup): void
{
    foreach ($transactionGroup->transactionJournals()->get() as $journal) {
        $journal->delete();   // 软删除 group 时，级联软删除其下所有 journal
    }
}
```

这意味着：group 被软删除 → 其下所有 journal 也被软删除 → 这些 journal 的 transactions 也因 `WHERE deleted_at IS NULL` 而从余额计算中消失。

但对于孤儿 journal（`transaction_group_id = null`），**没有任何 group 触发此 observer**，所以孤儿 journal 只能被手动发现和删除。

#### 10.6.5 forceDeleteOnError 清理的实际范围与局限

[forceDeleteOnError()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Factory/TransactionJournalFactory.php#L450-L460) 调用 [JournalDestroyService::destroy()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Services/Internal/Destroy/JournalDestroyService.php#L35-L49)：

```php
public function destroy(TransactionJournal $journal): void
{
    $group = $journal->transactionGroup;
    if (null !== $group) {
        $count = $group->transactionJournals->count();
        if (0 === $count) {
            $group->delete();    // 触发 DeletedTransactionGroupObserver → 级联软删
        }
    }
    $journal->delete();          // 软删除
}
```

**清理范围的精确评估**：

| 被清理的数据 | 清理机制 | 是否彻底 |
|------------|---------|---------|
| `transaction_journals` | 软删除（`deleted_at` 非空） | ⚠️ 记录仍存在，但余额计算排除 |
| `transactions` | 依赖外键级联或 observer？**均无显式清理** | ⚠️ 但 journal 软删除后，余额查询 JOIN `transaction_journals` 时 `WHERE deleted_at IS NULL` 排除了这些 transactions |
| `transaction_groups` | 若 group 为空则 `delete()`，触发 observer | ✅ group 被清理 |
| `journal_meta` | **无显式清理** | ❌ 残留在 DB 中（journal_id 仍指向已软删除的 journal） |
| `budget_transaction_journal` | **无显式清理** | ❌ pivot 记录残留 |
| `category_transaction_journal` | **无显式清理** | ❌ pivot 记录残留 |
| `notes` | **无显式清理** | ❌ Note 记录残留 |
| `tag_transaction_journal` | **无显式清理** | ❌ pivot 记录残留 |
| `locations` | **无显式清理** | ❌ Location 记录残留 |
| `piggy_bank_events` | **无显式清理** | ❌ 事件记录残留 |

**关键发现**：

1. `$journal->delete()` 是**软删除**（`SoftDeletes` trait），不是物理删除。已软删除的 journal 在余额查询中被排除，所以**对余额计算无影响**
2. 但子表数据（meta、pivot、notes、locations 等）**全部残留**在 DB 中，成为"软孤儿"
3. 这些软孤儿不会影响功能正确性（因为关联查询都会 JOIN journal 并过滤 `deleted_at`），但会：
   - 占用存储空间
   - 在直接 SQL 查询或数据导出时产生困惑
   - 在 journal 被恢复（`restore()`）时自动重新关联

4. **forceDeleteOnError 的调用时机极其有限**：仅在 P6/P7 的 `catch (FireflyException $e)` 中被调用（见 10.6.2 节）。P8-P15 和 G2-G3 失败时**完全不会触发此清理逻辑**

#### 10.6.6 补偿交易创建失败场景的完整残留矩阵（修正版）

以下矩阵基于对账场景的实际代码路径，精确标注每个阶段失败后各数据表的状态：

| 失败阶段 | 异常类型 | 被 catch? | transaction_journals | transactions | journal_meta | pivot(budget/category/tag) | notes | locations | transaction_groups | STEP 1 对账标记 | 用户看到 |
|---------|---------|----------|---------------------|-------------|-------------|---------------------------|-------|-----------|-------------------|----------------|---------|
| P1~P4 | FireflyException | ✅ journal 层 catch | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ✅ 已标记 | error flash |
| P5 | QueryException → FireflyException | ✅ journal 层 catch | ❌ 无（被 forceDelete 清理） | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ✅ 已标记 | error flash |
| P6~P7 | FireflyException | ✅ journal 层 catch + 清理 | ❌ 无（被清理） | ❌ 无（被清理） | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ✅ 已标记 | error flash |
| P6~P7 | 非 FireflyException | ❌ 漏捕获 | ✅ 1 条(group_id=null) | ✅ 2 条 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ✅ 已标记 | 500 + 孤儿交易 |
| P8 | QueryException | ❌ 漏捕获 | ✅ 1 条(group_id=null) | ✅ 2 条 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ✅ 已标记 | 500 + 孤儿交易 |
| P9 | QueryException | ❌ 漏捕获 | ✅ 1 条 | ✅ 2 条 | ❌ 无 | ⚠️ budgets sync 已执行 | ❌ 无 | ❌ 无 | ❌ 无 | ✅ 已标记 | 500 + 孤儿交易 |
| P10 | QueryException | ❌ 漏捕获 | ✅ 1 条 | ✅ 2 条 | ❌ 无 | ⚠️ budgets+categories sync 已执行 | ❌ 无 | ❌ 无 | ❌ 无 | ✅ 已标记 | 500 + 孤儿交易 |
| P14 (meta) | QueryException | ❌ 漏捕获 | ✅ 1 条 | ✅ 2 条 | ⚠️ 部分（import_hash_v2 可能已写入） | ✅ P9-P10 已 sync | ❌ 无 | ❌ 无 | ❌ 无 | ✅ 已标记 | 500 + 孤儿交易 |
| G2 | QueryException | ❌ 无 catch | ✅ 1 条(group_id=null) | ✅ 2 条 | ✅ 已写入 | ✅ 已 sync | ❌ 无 | ❌ 无 | ❌ 无 | ✅ 已标记 | 500 + 孤儿交易 |
| G3 | QueryException | ❌ 无 catch | ✅ 1 条(group_id=?) | ✅ 2 条 | ✅ 已写入 | ✅ 已 sync | ❌ 无 | ❌ 无 | ✅ 1 条(可能) | ✅ 已标记 | 500 + 可能部分关联 |

**最严重的一致性问题**：
1. 无论 STEP 2 在哪一步失败，STEP 1 的对账标记都**已经持久化**，且永远不会被回滚
2. P8-P15 和 G2/G3 失败产生的"孤儿 journal"（`transaction_group_id = null`）**不影响余额但不可见于 UI**，是最难发现的数据不一致
3. P14 失败若导致 `import_hash_v2` 未写入，该 journal 将**永远不参与重复交易检测**，增加了重复创建的风险

---

## 十一、提交数据来源与服务端重算策略分析

### 11.1 数据流全景：前端计算 → 隐藏表单 → 后端信任

对账提交流程涉及**三处差额/余额计算**的代码位置：

| 阶段 | 代码位置 | 计算方 | 目的 | 是否被 submit 信任 |
|------|---------|-------|------|------------------|
| ① 页面初始加载 | [ReconcileController::reconcile()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Controllers/Account/ReconcileController.php#L130-L137) | 服务端 | 给 input 框填默认余额值 | ❌ |
| ② 前端勾选时实时显示 | [Json\ReconcileController::overview()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Controllers/Json/ReconcileController.php#L127-L134) | 服务端 | 返回 HTML 显示差额 + 渲染隐藏字段 | ❌（只是**用于显示**，submit 不回调此方法） |
| ③ 真正提交 | [ReconcileStoreRequest::getAll()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Requests/ReconciliationStoreRequest.php#L48-L66) → [submit()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Controllers/Account/ReconcileController.php#L176-L193) | **完全来自 POST 参数** | 实际执行对账与补偿 | ✅ |

### 11.2 ReconciliationStoreRequest 数据提取：逐字段来源

[getAll() 方法](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Requests/ReconciliationStoreRequest.php#L48-L66)：

```php
public function getAll(): array
{
    $transactions = $this->get('journals');
    if (!is_array($transactions)) {
        $transactions = [];
    }
    $data = [
        'start'         => $this->getCarbonDate('start'),        // ✅ 来自 POST['start']（隐藏字段）
        'end'           => $this->getCarbonDate('end'),          // ✅ 来自 POST['end']（隐藏字段）
        'start_balance' => $this->convertString('startBalance'), // ✅ 来自 POST['startBalance']（隐藏字段）
        'end_balance'   => $this->convertString('endBalance'),   // ✅ 来自 POST['endBalance']（隐藏字段）
        'difference'    => $this->convertString('difference'),   // ✅ 来自 POST['difference']（隐藏字段）
        'journals'      => $transactions,                        // ✅ 来自 POST['journals[]']（隐藏字段数组）
        'reconcile'     => $this->convertString('reconcile'),    // ✅ 来自 POST['reconcile']（单选按钮值）
    ];
    return $data;
}
```

验证规则 [rules() 方法](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Requests/ReconciliationStoreRequest.php#L71-L82)：

```php
public function rules(): array
{
    return [
        'start'        => 'required|date',
        'end'          => 'required|date',
        'startBalance' => ['nullable', new IsValidAmount()],  // ⚠️ 仅校验格式合法，不校验数值正确
        'endBalance'   => ['nullable', new IsValidAmount()],  // ⚠️ 仅校验格式合法，不校验数值正确
        'difference'   => ['required', new IsValidAmount()],  // ⚠️ 仅校验格式合法，不校验数值正确
        'journals'     => [new ValidJournals()],              // ⚠️ 仅校验"所有权归属当前用户"
        'reconcile'    => 'required|in:create,nothing',
    ];
}
```

### 11.3 ValidJournals 规则的局限性

[ValidJournals::validate()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Rules/ValidJournals.php#L40-L58) 仅做**存在性+归属校验**：

```php
public function validate(string $attribute, mixed $value, Closure $fail): void
{
    if (!is_array($value)) return;
    $userId = auth()->user()->id;
    foreach ($value as $journalId) {
        // ⚠️ 只检查：该 journal_id 是否存在于当前用户下
        $count = TransactionJournal::where('id', $journalId)->where('user_id', $userId)->count();
        if (0 === $count) {
            $fail('validation.invalid_selection')->translate();
            return;
        }
        // ❌ 不检查：journal 的日期是否在 start/end 范围内
        // ❌ 不检查：journal 是否属于当前对账的账户
        // ❌ 不检查：journal 是否已被标记 reconciled=true（重复标记）
        // ❌ 不检查：journal 的交易类型是否允许对账
    }
}
```

### 11.4 submit() 对传入数据的使用方式

查看 [submit()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Controllers/Account/ReconcileController.php#L176-L193) 如何使用 `$data`：

```php
// STEP 1: 直接使用前端传来的 journals 数组，不做二次过滤/重算
foreach ($data['journals'] as $journalId) {
    $this->repository->reconcileById((int) $journalId);   // 信任 journals，不检查日期/账户/重复
}

// STEP 2: 直接使用前端传来的 difference，不做二次计算
if ('create' === $data['reconcile']) {
    $result = $this->createReconciliation(
        $account, $start, $end,
        $data['difference']        // ⚠️ 信任前端传的差额金额，不重新计算
    );
}

// ⚠️ 注意：$data['start_balance'] 和 $data['end_balance'] 被提取但 submit() 中 NEVER USED！
// 它们存在于 POST 数据中，也被 getAll() 取出，但在 submit() 和 createReconciliation() 中完全未被引用
```

### 11.5 前端隐藏字段的注入路径（overview.twig）

前端之所以能传这些值，是因为 [Json\ReconcileController::overview()](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Controllers/Json/ReconcileController.php#L127-L134) 计算后通过 [overview.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/resources/views/accounts/reconcile/overview.twig#L11-L50) 渲染为隐藏表单字段：

```twig
<form action="{{ route }}" method="POST">
    <input type="hidden" name="start" value="{{ start.format('Y-m-d') }}"/>
    <input type="hidden" name="end" value="{{ end.format('Y-m-d') }}"/>
    <input type="hidden" name="startBalance" value="{{ startBalance }}"/>
    <input type="hidden" name="endBalance" value="{{ endBalance }}"/>
    <input type="hidden" name="difference" value="{{ difference }}"/>
    {% for id in selectedIds %}
        <input type="hidden" name="journals[]" value="{{ id }}"/>
    {% endfor %}
    <!-- 单选按钮 -->
    <input type="radio" name="reconcile" value="create">
    <input type="radio" name="reconcile" value="nothing" checked>
</form>
```

但**浏览器提交时不会再次请求 overview() 重新计算**——这些隐藏字段的值是**用户点击"确认对账"那一瞬间的前端 DOM 快照**。

---

## 十二、一致性风险汇总与异常场景推演

### 12.1 风险分类与严重程度

| # | 风险点 | 触发条件 | 严重程度 | 影响 |
|---|-------|---------|---------|------|
| R1 | **无整体事务** | STEP 2 补偿交易创建失败时 STEP 1 已提交 | 🟥 高 | 对账标记已完成但余额未修正，用户困惑 |
| R2 | **差额客户端信任** | 恶意篡改 POST['difference'] 金额 | 🟧 中高 | 可创建任意金额的补偿交易，影响账户余额 |
| R3 | **journals 跨账户/跨期** | 篡改 POST['journals[]'] 加入非本账户、非本期、已对账的 journal | 🟧 中 | 错误标记无关交易为已对账，可能被重复对账 |
| R4 | **DuplicateTransactionException 未捕获** | 极端重复提交（hash 冲突时） | 🟧 中 | 500 错误，状态同 R1 |
| R5 | **起止余额未校验** | 篡改 startBalance/endBalance（虽然当前未使用，但语义上不严谨） | 🟩 低 | 当前无直接影响，若后续代码改动可能引入风险 |
| R6 | **前端与服务端时间差** | 用户打开对账页面后，其他浏览器标签新增了期间内交易 | 🟧 中 | overview 计算时基于旧状态，提交时数据已变化，差额可能不准 |

### 12.2 风险场景详细推演

#### 场景 R1：补偿交易失败导致"半对账"（最易触发）

**时序与后果**：

```
T1: 用户勾选 5 条交易，差额 = +25.00，选择"创建补偿交易"
T2: 点击提交 → submit() 开始
T3: STEP 1 - 循环调用 reconcileById()：
    ├─ 更新 journal #101 → transactions reconciled=true (已 COMMIT)
    ├─ 更新 journal #102 → transactions reconciled=true (已 COMMIT)
    ├─ 更新 journal #103 → transactions reconciled=true (已 COMMIT)
    ├─ 更新 journal #104 → transactions reconciled=true (已 COMMIT)
    └─ 更新 journal #105 → transactions reconciled=true (已 COMMIT)
T4: STEP 2 - createReconciliation()：
    ├─ getReconciliation() OK
    ├─ TransactionGroupFactory::create() → journal 创建时
    │   └─ validateAccounts() 失败（极端情况下对账账户被软删除）
    │       └─ 抛出 FireflyException("Source: xxx is invalid")
    └─ catch 捕获，返回错误字符串
T5: flash('error', 'reconciliation_error: Source: xxx is invalid')
T6: 重定向到账户详情页

状态快照：
  • journal #101~#105: reconciled = true ✅ 已标记
  • 补偿交易: 不存在 ❌
  • 账户余额 SUM(amount): 仍比银行对账单少 25.00
  • 用户体验: 看到 error 提示，但所有交易都显示"已对账"对勾
```

#### 场景 R2：恶意篡改差额数据（安全风险）

假设攻击者通过浏览器 DevTools 修改隐藏字段：

```
POST /accounts/reconcile/3/submit/20260101/20260131

原表单（正常）：
  difference = 12.50
  reconcile  = create
  journals[] = [101, 102, 103]

篡改后（恶意）：
  difference = 9999999.99   ← 人为放大
  reconcile  = create
  journals[] = [101]        ← 只标记 1 条
```

**后端实际执行**：
1. STEP 1：只将 journal #101 标记 `reconciled=true`（校验通过，因为 journal 属于用户）
2. STEP 2：使用 `difference = 9999999.99` 创建补偿交易
   - 若 `difference > 0`：创建 资产账户 → 对账账户 的 9,999,999.99 "Reconciliation" 交易
   - **后果**：资产账户余额凭空减少近千万，余额严重失真
   - 此交易类型在报表中是独立的，但对"净资产"和"可用余额"计算完全生效

#### 场景 R3：跨账户、跨期标记对账

正常流程：账户 A，2026-01 对账，选 journal #101 #102 #103

篡改后：
```
journals[] = [101, 102, 103, 205, 333]
```
- journal #205：属于账户 B，2026-02 的一笔提现（属于同一用户 → ValidJournals 校验通过）
- journal #333：已 reconciled=true 的历史交易（ValidJournals 不检查此状态）

**执行后果**：
- journal #205 的 `reconciled` 从 `false` → `true`（但它属于账户 B，用户在账户 A 对账时错误地标记了账户 B 的交易）
- journal #333 的 `reconciled` 再次 `UPDATE ... SET true`（幂等，无副作用，但语义上不应该被"再次对账"）
- 下次用户在账户 B 的对账流程中，journal #205 将作为"已对账交易"参与差额计算（clearedAmount），可能造成账户 B 的对账差额异常

#### 场景 R6：并发写入导致的差额失真

```
T0: 用户A打开账户对账页面，加载 2026-01 的交易列表
    → 期初余额 = 10000.00, 期末余额 = 12000.00, 交易 5 笔
T1: 用户B（同账户另一标签页）新增一笔 2026-01-15 的支出 500.00（此时 DB 已更新）
T2: 用户A勾选全部 5 条交易 → overview() 返回
    → difference = (10000 - 12000) + 0 + (-2000) = 0
    → 但 overview() 使用的是 T0 时刻的 journal 集合，不包含 T1 新增的 500
T3: 用户A提交 → STEP 1 标记 5 条 → STEP 2 因 diff=0 不创建补偿
T4: 状态：
    • 5 条交易: reconciled = true
    • T1 新增的 500: reconciled = false, 且未被纳入本次对账范围
    • 实际余额 = 12000 - 500 = 11500，对账期末余额仍记为 12000
    → 出现 500 的隐形差异，下次对账时才会暴露
```

### 12.3 针对风险的改进建议（概念性）

| 风险 | 建议方案 |
|------|---------|
| R1（无整体事务） | 在 submit() 外层包裹 `DB::transaction(function() { ... })`，STEP 1 和 STEP 2 要么全成功要么全回滚 |
| R2（差额信任前端） | 在 submit() 中服务端重新调用 Json\ReconcileController::overview 中的差额计算逻辑，校验前端传入的 difference 与服务端重算值一致（容差 0.01） |
| R3（journals 校验不足） | 在 ValidJournals 中补充：① journal 涉及账户必须包含目标资产账户 ② journal 日期须在 [start, end] 范围内 ③ journal 当前 reconciled=false |
| R4（异常漏捕获） | 在 createReconciliation() 的 try-catch 中加入 `catch (\Exception $e)` 兜底，或在 submit() 外层 catch 后手动回滚已对账标记 |
| R6（并发差额） | 提交时重新基于 DB 当前状态重算差额，而非信任 overview 时刻的前端快照值 |

---

## 十三、附：代码引用索引表

| 功能点 | 文件路径 | 行号范围 |
|-------|---------|---------|
| 对账提交主入口 | [ReconcileController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Controllers/Account/ReconcileController.php) | #L169-L204 |
| 补偿交易创建 | [ReconcileController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Controllers/Account/ReconcileController.php) | #L211-L270 |
| 对账标记（底层） | [JournalRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Repositories/Journal/JournalRepository.php) | #L245-L250 |
| 提交请求数据提取 | [ReconciliationStoreRequest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Requests/ReconciliationStoreRequest.php) | #L48-L82 |
| 交易 ID 所有权校验 | [ValidJournals.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Rules/ValidJournals.php) | #L40-L58 |
| 差额（服务端显示用）计算 | [Json\ReconcileController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Http/Controllers/Json/ReconcileController.php) | #L71-L163 |
| 对账调节账户获取/创建 | [AccountRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Repositories/Account/AccountRepository.php) | #L426-L458 |
| 补偿交易落地（复式记账） | [TransactionJournalFactory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Factory/TransactionJournalFactory.php) | #L230-L410 |
| 单条 Transaction 创建（带 reconciled） | [TransactionFactory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Factory/TransactionFactory.php) | #L121-L172 |
| 余额聚合查询 | [Steam.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Support/Steam.php) | #L72-L157 |
| 重复交易异常类 | [DuplicateTransactionException.php](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/app/Exceptions/DuplicateTransactionException.php) | #L29-L32 |
| 提交表单（隐藏字段）渲染 | [overview.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/24-firefly-iii/resources/views/accounts/reconcile/overview.twig) | #L11-L104 |
