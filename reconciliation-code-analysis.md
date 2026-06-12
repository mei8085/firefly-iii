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
