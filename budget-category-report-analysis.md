# Firefly III 预算与分类统计聚合机制 - 可复核代码分析

> 分析版本：Firefly III (develop branch)
> 分析日期：2026-05-19
> 文档状态：可复核

---

## 目录

1. 预算周期与交易日期对齐规则
2. 分类查询到报表生成器的调用链
3. 跨币种汇总的换算与回退策略
4. 关键边界条件梳理
5. 代码引用索引

---

## 1. 预算周期与交易日期对齐规则

### 1.1 核心数据结构

**BudgetLimit 模型** 是预算周期的载体：

```php
// app/Models/BudgetLimit.php:49
protected $fillable = [
    'budget_id',              // 关联预算ID
    'start_date', 'end_date', // 预算周期范围
    'start_date_tz', 'end_date_tz', // 时区信息
    'amount',                 // 预算金额
    'transaction_currency_id', // 预算货币
    'native_amount'           // 本位币预换算金额
];
```

### 1.2 预算周期与查询范围的重叠检测

**代码位置**：`app/Repositories/Budget/BudgetLimitRepository.php:208-254`

重叠检测采用三分支逻辑，覆盖所有可能的时间关系：

```
查询范围:    [S────────────────────────E]
情况1:           [BL_start────BL_end]      (BL结束于范围内)
情况2:   [BL_start────BL_end]              (BL开始于范围内)
情况3: [BL_start────────────────────BL_end] (BL完全包含范围)
```

**实现代码**：

```php
// app/Repositories/Budget/BudgetLimitRepository.php:229-250
return $budget
    ->budgetlimits()
    ->where(static function (Builder $q5) use ($start, $end): void {
        $q5->where(static function (Builder $q1) use ($start, $end): void {
            // 分支1: 预算限制结束日期落在查询范围内
            $q1->where(static function (Builder $q2) use ($start, $end): void {
                $q2->where('budget_limits.end_date', '>=', $start->format('Y-m-d 00:00:00'))
                   ->where('budget_limits.end_date', '<=', $end->format('Y-m-d 23:59:59'));
            })
            // 分支2: 预算限制开始日期落在查询范围内
            ->orWhere(static function (Builder $q3) use ($start, $end): void {
                $q3->where('budget_limits.start_date', '>=', $start->format('Y-m-d 00:00:00'))
                   ->where('budget_limits.start_date', '<=', $end->format('Y-m-d 23:59:59'));
            });
        })
        // 分支3: 预算限制完全包含查询范围
        ->orWhere(static function (Builder $q4) use ($start, $end): void {
            $q4->where('budget_limits.start_date', '<=', $start->format('Y-m-d 23:59:59'))
               ->where('budget_limits.end_date', '>=', $end->format('Y-m-d 00:00:00'));
        });
    })
```

> **边界条件**：使用时间边界 `00:00:00` 和 `23:59:59` 确保整天数据被正确包含。

### 1.3 预算金额的按日分摊算法

**代码位置**：`app/Repositories/Budget/BudgetLimitRepository.php:61-105`

当预算周期与报表周期不完全匹配时，采用**按日线性分摊**策略：

```php
// app/Repositories/Budget/BudgetLimitRepository.php:92-102
foreach ($set as $budgetLimit) {
    // 完全匹配：直接累加
    if ($budgetLimit->start_date->isSameDay($start) && $budgetLimit->end_date->isSameDay($end)) {
        $result = bcadd((string) $budgetLimit->amount, $result);
        continue;
    }
    // 部分重叠：按重叠天数 × 日均金额计算
    $period = Period::make($start, $end, precision: Precision::DAY(), boundaries: Boundaries::EXCLUDE_NONE());
    $amountPerDay = $this->getDailyAmount($budgetLimit);
    $result = bcadd($result, bcmul((string) $period->length(), $amountPerDay));
}
```

日均金额计算（12位精度）：

```php
// app/Repositories/Budget/BudgetLimitRepository.php:256-270
public function getDailyAmount(BudgetLimit $budgetLimit): string
{
    $limitPeriod = Period::make(
        $budgetLimit->start_date, 
        $budgetLimit->end_date, 
        precision: Precision::DAY(), 
        boundaries: Boundaries::EXCLUDE_NONE()
    );
    $days = $limitPeriod->length();
    return bcdiv($budgetLimit->amount, (string) $days, 12);
}
```

### 1.4 交易查询的周期对齐策略

**代码位置**：`app/Support/Report/Budget/BudgetReportGenerator.php:289-333`

报表生成时，每个预算限制**独立查询**自身周期内的交易：

```php
// app/Support/Report/Budget/BudgetReportGenerator.php:289-296
private function processLimit(Budget $budget, BudgetLimit $limit): void
{
    $budgetId = $budget->id;
    $limitId = $limit->id;
    $limitCurrency = $limit->transactionCurrency ?? $this->currency;
    $currencyId = $limitCurrency->id;
    
    // 关键：使用预算限制自身的日期范围查询交易
    $expenses = $this->opsRepository->sumExpenses(
        $limit->start_date, 
        $limit->end_date, 
        $this->accounts, 
        new Collection()->push($budget)
    );
    $spent = $expenses[$currencyId]['sum'] ?? '0';
    // ... 计算剩余、超支等 ...
}
```

> **设计考量**：使用 BL 自身周期而非报表全局周期，确保预算与实际支出的时间维度严格对齐。

### 1.5 预算执行计算逻辑

```php
// app/Support/Report/Budget/BudgetReportGenerator.php:297-298
// 剩余预算 = 预算金额 + 已支出（支出为负数），若结果 < 0 则剩余为 0
$left = -1 === bccomp(bcadd($limit->amount, $spent), '0') ? '0' : bcadd($limit->amount, $spent);

// 超支金额 = 若 |支出| > 预算，则超出部分，否则为 0
$overspent = 1 === bccomp(bcmul($spent, '-1'), $limit->amount) ? bcadd($spent, $limit->amount) : '0';
```

---

## 2. 分类查询到报表生成器的调用链

### 2.1 调用链路全景图

```
┌───────────────────────────────────────────────────────────┐
│           CategoryReportGenerator (入口)                  │
│  operations()                                             │
│     ├─ listIncome()         → 有分类收入                  │
│     ├─ listExpenses()       → 有分类支出                  │
│     ├─ listTransferredIn()  → 有分类转入                  │
│     ├─ listTransferredOut() → 有分类转出                  │
│     ├─ listIncome()         → 无分类收入 (NoCategory)     │
│     └─ listExpenses()       → 无分类支出 (NoCategory)     │
└──────────────────────────┬────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────┐
│        OperationsRepository (分类操作仓库)                 │
│  collectExpenses/Income/Transfers()                        │
│    ├─ setRange(start, end)                                │
│    ├─ setTypes([WITHDRAWAL/DEPOSIT/TRANSFER])             │
│    ├─ setCategories(categories)                           │
│    ├─ withCategoryInformation()                           │
│    └─ withAccountInformation() / withBudgetInformation()  │
└──────────────────────────┬────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────┐
│        GroupCollectorInterface (交易收集器)                │
│  getExtractedJournals() → 标准化交易数组                   │
└──────────────────────────┬────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────┐
│      TransactionSummarizer (交易汇总器)                    │
│  groupByCurrencyId() → 按货币汇总金额                      │
└───────────────────────────────────────────────────────────┘
```

### 2.2 分类报表生成流程

**代码位置**：`app/Support/Report/Category/CategoryReportGenerator.php:63-81`

```php
// app/Support/Report/Category/CategoryReportGenerator.php:63-81
public function operations(): void
{
    // 1. 收集6类数据源
    $earnedWith     = $this->opsRepository->listIncome($this->start, $this->end, $this->accounts);
    $spentWith      = $this->opsRepository->listExpenses($this->start, $this->end, $this->accounts);
    $transferredIn  = $this->opsRepository->listTransferredIn($this->start, $this->end, $this->accounts);
    $transferredOut = $this->opsRepository->listTransferredOut($this->start, $this->end, $this->accounts);
    $earnedWithout  = $this->noCatRepository->listIncome($this->start, $this->end, $this->accounts);
    $spentWithout   = $this->noCatRepository->listExpenses($this->start, $this->end, $this->accounts);

    // 2. 初始化结果结构
    $this->report = ['categories' => [], 'sums' => []];

    // 3. 统一处理所有数据源
    foreach ([$earnedWith, $spentWith, $earnedWithout, $spentWithout, $transferredIn, $transferredOut] as $data) {
        $this->processOpsArray($data);
    }
}
```

### 2.3 数据处理流水线

#### 第一层：按货币分组

```php
// app/Support/Report/Category/CategoryReportGenerator.php:145-165
private function processCurrencyArray(int $currencyId, array $currencyRow): void
{
    $this->report['sums'][$currencyId] ??= [
        'spent' => '0', 'earned' => '0', 'sum' => '0',
        'currency_id' => $currencyRow['currency_id'],
        'currency_symbol' => $currencyRow['currency_symbol'],
        'currency_name' => $currencyRow['currency_name'],
        'currency_code' => $currencyRow['currency_code'],
        'currency_decimal_places' => $currencyRow['currency_decimal_places'],
    ];

    foreach ($currencyRow['categories'] as $categoryId => $categoryRow) {
        $this->processCategoryRow($currencyId, $currencyRow, $categoryId, $categoryRow);
    }
}
```

#### 第二层：按分类汇总

```php
// app/Support/Report/Category/CategoryReportGenerator.php:104-143
private function processCategoryRow(int $currencyId, array $currencyRow, int $categoryId, array $categoryRow): void
{
    $key = sprintf('%s-%s', $currencyId, $categoryId);
    $this->report['categories'][$key] ??= [
        'id' => $categoryId,
        'title' => $categoryRow['name'],
        'currency_id' => $currencyRow['currency_id'],
        // ... 货币信息 ...
        'spent' => '0', 'earned' => '0', 'sum' => '0',
    ];

    foreach ($categoryRow['transaction_journals'] as $journal) {
        // 货币级汇总
        $this->report['sums'][$currencyId]['sum'] = bcadd($this->report['sums'][$currencyId]['sum'], (string) $journal['amount']);
        $this->report['sums'][$currencyId]['spent'] = -1 === bccomp((string) $journal['amount'], '0')
            ? bcadd($this->report['sums'][$currencyId]['spent'], (string) $journal['amount'])
            : $this->report['sums'][$currencyId]['spent'];
        $this->report['sums'][$currencyId]['earned'] = 1 === bccomp((string) $journal['amount'], '0')
            ? bcadd($this->report['sums'][$currencyId]['earned'], (string) $journal['amount'])
            : $this->report['sums'][$currencyId]['earned'];

        // 分类级汇总
        $this->report['categories'][$key]['sum'] = bcadd($this->report['categories'][$key]['sum'], (string) $journal['amount']);
        $this->report['categories'][$key]['spent'] = -1 === bccomp((string) $journal['amount'], '0')
            ? bcadd($this->report['categories'][$key]['spent'], (string) $journal['amount'])
            : $this->report['categories'][$key]['spent'];
        $this->report['categories'][$key]['earned'] = 1 === bccomp((string) $journal['amount'], '0')
            ? bcadd($this->report['categories'][$key]['earned'], (string) $journal['amount'])
            : $this->report['categories'][$key]['earned'];
    }
}
```

### 2.4 GroupCollector 字段映射

**代码位置**：`app/Helpers/Collector/GroupCollector.php:109-161`

关键查询字段：

```php
// app/Helpers/Collector/GroupCollector.php:137-154
'currency.code as currency_code',
'currency.name as currency_name',
'currency.symbol as currency_symbol',
'currency.decimal_places as currency_decimal_places',

// 原生金额（本位币预换算）
'source.native_amount as pc_amount',
'source.native_foreign_amount as pc_foreign_amount',

// 外币信息
'source.foreign_amount as foreign_amount',
'source.foreign_currency_id as foreign_currency_id',
'foreign_currency.code as foreign_currency_code',
// ...
```

### 2.5 交易类型过滤

在 `listExpenses()` 中排除负债账户的转入：

```php
// app/Repositories/Category/OperationsRepository.php:112-113
if ($accounts instanceof Collection && $accounts->count() > 0) {
    $collector->setAccounts($accounts);
    $collector->excludeDestinationAccounts($accounts); // 排除向负债账户的转账
}
```

在 `listIncome()` 中排除负债账户的转出：

```php
// app/Repositories/Category/OperationsRepository.php:193-194
if ($accounts instanceof Collection && $accounts->count() > 0) {
    $collector->setAccounts($accounts);
    $collector->excludeSourceAccounts($accounts); // 排除从负债账户的转出
}
```

---

## 3. 跨币种汇总的换算与回退策略

### 3.1 本位币金额字段体系

系统在多个核心模型中引入 `native_` 前缀字段，预存本位币换算值：

| 模型 | 字段 | 说明 | 代码位置 |
|------|------|------|----------|
| Transaction | `native_amount` | 交易金额的本位币值 | `app/Models/Transaction.php:56` |
| Transaction | `native_foreign_amount` | 外币金额的本位币值 | `app/Models/Transaction.php:57` |
| BudgetLimit | `native_amount` | 预算金额的本位币值 | `app/Models/BudgetLimit.php:49` |
| AvailableBudget | `native_amount` | 可用预算的本位币值 | 数据库迁移 |
| Bill | `native_amount_min` / `native_amount_max` | 账单金额的本位币值 | 数据库迁移 |
| Account | `native_virtual_balance` | 账户虚拟余额的本位币值 | 数据库迁移 |

> **迁移文件**：`database/migrations/2024_12_19_061003_add_native_amount_column.php`

### 3.2 汇率转换器核心架构

**代码位置**：`app/Support/Http/Api/ExchangeRateConverter.php`

#### 3.2.1 四级回退策略与完整回退路径

```php
// app/Support/Http/Api/ExchangeRateConverter.php:239-287
private function getRate(TransactionCurrency $from, TransactionCurrency $to, Carbon $date): string
{
    $key = $this->getCacheKey($from, $to, $date);

    // 第1级：Laravel 缓存命中
    $res = Cache::get($key);
    if (null !== $res) {
        Log::debug('Return cached rate');
        return $res;
    }

    // 第2级：数据库正向查询 (from → to)
    $rate = $this->getFromDB($from->id, $to->id, $date->format('Y-m-d'));
    if (null !== $rate) {
        Cache::forever($key, $rate);
        return $rate;
    }

    // 第3级：数据库反向查询并计算倒数 (to → from)
    $rate = $this->getFromDB($to->id, $from->id, $date->format('Y-m-d'));
    if (null !== $rate) {
        $rate = bcdiv('1', $rate);
        Cache::forever($key, $rate);
        return $rate;
    }

    // 第4级：EUR 作为中介货币的三角换算
    $first = $this->getEuroRate($from, $date);   // from → EUR
    $second = $this->getEuroRate($to, $date);     // to → EUR
    if (0 === bccomp('0', $first) || 0 === bccomp('0', $second)) {
        Log::warning('Not enough info for conversion, return 1');
        return '1';
    }
    $rate = bcmul($first, bcdiv('1', $second));   // (from→EUR) × (EUR→to)
    Cache::forever($key, $rate);
    return $rate;
}
```

> **完整回退路径深度分析（以查询日期早于所有汇率的场景为例）**：
> 
> ```
> 前置条件:
>   - 数据库汇率: [R1:2024-01-01, R2:2024-01-15, R3:2024-02-01] （USD→EUR）
>   - 查询日期: 2023-12-01（早于所有汇率）
>   - 配置文件: cer.rates.USD = 1.1349044 （EUR→USD 的静态备份汇率）
> 
> 回退路径:
> ┌───────────────────────────────────────────────────────────┐
> │ 第1级: Laravel 缓存                                       │
> │   Cache::get('cer-USD-EUR-2023-12-01') → null（无缓存）    │
> └───────────────────────┬───────────────────────────────────┘
>                         ↓
> ┌───────────────────────────────────────────────────────────┐
> │ 第2级: 数据库正向查询 (USD → EUR)                         │
> │   getFromDB(USD, EUR, '2023-12-01')                       │
> │   → where(date <= '2023-12-01') → 无匹配 → 返回 null       │
> └───────────────────────┬───────────────────────────────────┘
>                         ↓
> ┌───────────────────────────────────────────────────────────┐
> │ 第3级: 数据库反向查询 (EUR → USD)                         │
> │   getFromDB(EUR, USD, '2023-12-01')                       │
> │   → where(date <= '2023-12-01') → 无匹配 → 返回 null       │
> └───────────────────────┬───────────────────────────────────┘
>                         ↓
> ┌───────────────────────────────────────────────────────────┐
> │ 第4级: EUR 三角换算（关键！备份汇率在此生效）               │
> │   getEuroRate(USD, '2023-12-01')                          │
> │   ├─ 正向查询: getFromDB(USD, EUR, '2023-12-01') → null   │
> │   ├─ 反向查询: getFromDB(EUR, USD, '2023-12-01') → null   │
> │   ├─ 🔹 备份汇率: config('cer.rates.USD') = 1.1349044     │
> │   │     → 返回 bcdiv('1', '1.1349044') = 0.8811...        │
> │   └─ 返回: 0.881131...（非零！）                           │
> │                                                           │
> │   getEuroRate(EUR, '2023-12-01')                          │
> │   └─ 同货币直接返回: '1'                                  │
> │                                                           │
> │   检查: 0 === bccomp('0', '0.8811...') → false ✅         │
> │        0 === bccomp('0', '1') → false ✅                   │
> │   计算: bcmul('0.8811...', bcdiv('1', '1')) = 0.8811...    │
> └───────────────────────┬───────────────────────────────────┘
>                         ↓
> 最终结果: 返回汇率 = 0.881131...（使用配置中的备份汇率，而非 1:1）
> ```
> 
> **关键分支详解**：
> 
> 备份汇率仅在 `getEuroRate()` 的第 3 个分支生效：
> ```php
> // app/Support/Http/Api/ExchangeRateConverter.php:161-168
> // grab backup values from config file:
> $backup = config(sprintf('cer.rates.%s', $currency->code));
> if (null !== $backup) {
>     return bcdiv('1', (string) $backup);  // 注意：配置存储的是 EUR→X，所以取倒数
> }
> ```
> 
> **何时会回落到 1:1 汇率？**
> 
> 必须同时满足以下条件：
> 1. ✅ 查询日期早于所有数据库汇率（正向/反向查询均返回 null）
> 2. ✅ 配置文件 `cer.rates` 中**没有**该货币的备份汇率
> 3. ✅ 三角换算失败（`getEuroRate` 返回 `'0'`）
> 
> 代码证据：
> ```php
> // app/Support/Http/Api/ExchangeRateConverter.php:275-278
> if (0 === bccomp('0', $first) || 0 === bccomp('0', $second)) {
>     Log::warning('There is not enough information to convert %s to %s on date %s', ...);
>     return '1';  // 仅当三角换算失败时才返回 1
> }
> ```
> 
> **配置文件中的备份汇率**（`config/cer.php:34-79`）：
> ```php
> 'date'  => '2025-04-15',  // 备份汇率的基准日期
> 'rates' => [
>     'EUR' => 1,
>     'USD' => 1.1349044,   // 1 EUR = 1.1349044 USD
>     'GBP' => 0.86003261,
>     'JPY' => 162.47195,
>     // ... 约 30 种主流货币
> ],
> ```
> 
> **重要结论**：
> - 对于配置中存在的 ~30 种主流货币，即使查询日期早于所有数据库汇率，**也不会回落到 1:1**，而是使用配置中的静态备份汇率
> - 仅对于配置中不存在的小众货币，才会最终回落到 1:1 换算
> - 备份汇率是**静态**的，不随查询日期变化（基准日期为 2025-04-15）

#### 3.2.2 数据库查询逻辑与日期边界分析

```php
// app/Support/Http/Api/ExchangeRateConverter.php:174-234
private function getFromDB(int $from, int $to, string $date): ?string
{
    // 同货币直接返回 1
    if ($from === $to) { return '1'; }

    // 请求内缓存（避免同一请求重复查询）
    $preparedRate = $this->prepared[$date][$from][$to] ?? null;
    if (null !== $preparedRate && 0 !== bccomp('0', $preparedRate)) {
        return $preparedRate;
    }

    // 属性缓存
    $cache = new CacheProperties();
    $cache->addProperty(sprintf('cer-%d-%d-%s', $from, $to, $date));
    if ($cache->has()) { return $cache->get(); }

    // 实际数据库查询：核心日期筛选逻辑
    $result = $this->userGroup
        ->currencyExchangeRates()
        ->where('from_currency_id', $from)
        ->where('to_currency_id', $to)
        ->where('date', '<=', $date)    // 关键1：只选择日期 <= 查询日期的汇率
        ->orderBy('date', 'DESC')       // 关键2：按日期倒序（最新在前）
        ->first()                       // 关键3：取第一条（最新可用汇率）
    ;
    ++$this->queryCount;

    $rate = (string) $result?->rate;
    if ('' === $rate || 0 === bccomp('0', $rate)) { return null; }

    $cache->store($rate);
    $this->prepared[$date][$from][$to] = $rate;
    $this->prepared[$date][$to][$from] = bcdiv('1', $rate); // 同时缓存反向

    return $rate;
}
```

> **日期边界深度分析**：
> 
> 核心筛选条件是 `where('date', '<=', $date)` + `orderBy('date', 'DESC')` + `first()` 的组合：
> 
> ```
> 时间轴 →
> [R1:2024-01-01] [R2:2024-01-15] [R3:2024-02-01]
>                                     ↑ 查询日期: 2024-02-15
> ```
> 
> | 场景 | 查询日期 | 匹配逻辑 | 结果 |
> |------|----------|----------|------|
> | ① 查询日期在所有汇率之后 | 2024-02-15 | `date <= 2024-02-15` 匹配 R1,R2,R3，倒序取 R3 | ✅ 返回 R3（最新汇率） |
> | ② 查询日期在汇率之间 | 2024-01-20 | `date <= 2024-01-20` 匹配 R1,R2，倒序取 R2 | ✅ 返回 R2（最近的历史汇率） |
> | ③ 查询日期在所有汇率之前 | 2023-12-01 | `date <= 2023-12-01` 无匹配 | ❌ 返回 `null`，进入回退流程 |
> 
> **重要结论**：
> - 当查询日期早于所有汇率时，**不会**取最早汇率，而是返回 `null`
> - 返回 `null` 后会触发后续回退：反向查询 → EUR 三角换算 → 最终返回 `1`（1:1 换算）
> - 这是一种保守策略：宁可 1:1 换算，也不使用未来的汇率（因为未来汇率不可知）

#### 3.2.3 EUR 中介汇率查询

```php
// app/Support/Http/Api/ExchangeRateConverter.php:142-172
private function getEuroRate(TransactionCurrency $currency, Carbon $date): string
{
    $euroId = $this->getEuroId();
    if ($euroId === $currency->id) { return '1'; }

    // 正向查询: currency → EUR
    $rate = $this->getFromDB($currency->id, $euroId, $date->format('Y-m-d'));
    if (null !== $rate) { return $rate; }

    // 反向查询: EUR → currency，然后取倒数
    $rate = $this->getFromDB($euroId, $currency->id, $date->format('Y-m-d'));
    if (null !== $rate) { return bcdiv('1', $rate); }

    // 配置文件备份汇率
    $backup = config(sprintf('cer.rates.%s', $currency->code));
    if (null !== $backup) { return bcdiv('1', (string) $backup); }

    return '0'; // 无可用汇率
}
```

### 3.3 换算执行入口

```php
// app/Support/Http/Api/ExchangeRateConverter.php:61-71
public function convert(TransactionCurrency $from, TransactionCurrency $to, Carbon $date, string $amount): string
{
    if (false === $this->enabled()) { return $amount; }
    $rate = $this->getCurrencyRate($from, $to, $date);
    return Steam::bcround(bcmul($amount, $rate), $to->decimal_places);
}
```

### 3.4 交易汇总时的币种选择

**代码位置**：`app/Support/Report/Summarizer/TransactionSummarizer.php:46-152`

```php
// app/Support/Report/Summarizer/TransactionSummarizer.php:67-90
if ($this->convertToPrimary) {
    $usePrimary = $this->default->id !== (int) $journal['currency_id'];
    $useForeign = $this->default->id === (int) $journal['foreign_currency_id'];

    if ($usePrimary) {
        // 交易货币 ≠ 本位币：使用预换算的 pc_amount
        $field = 'pc_amount';
        $currencyId = $this->default->id;
        $currencyName = $this->default->name;
        // ... 更新所有货币信息为本位币 ...
    }
    if ($useForeign) {
        // 外币恰好是本位币：直接使用 foreign_amount
        $field = 'foreign_amount';
        $currencyId = (int) $journal['foreign_currency_id'];
        $currencyName = $journal['foreign_currency_name'];
        // ... 使用外币信息 ...
    }
}
```

### 3.5 批量重算机制

**代码位置**：`app/Services/Internal/Recalculate/PrimaryAmountRecalculationService.php`

当汇率数据变化时，可触发全量重算：

```php
// app/Services/Internal/Recalculate/PrimaryAmountRecalculationService.php:55-74
public function recalculate(): void
{
    if (false === FireflyConfig::get('enable_exchange_rates', config('cer.enabled'))->data) {
        return;
    }
    foreach ($repository->getAll() as $userGroup) {
        $this->resetGenericTables($userGroup);      // 清空 accounts/available_budgets/bills
        $this->resetPiggyBanks($userGroup);         // 清空储蓄罐相关
        $this->resetBudgets($userGroup);            // 清空预算相关
        $this->resetTransactions($userGroup);       // 清空交易 native_amount
        $this->recalculateForGroup($userGroup);     // 触发重新计算
    }
}
```

实际重算通过 `touch()` 触发模型观察者完成：

```php
// app/Services/Internal/Recalculate/PrimaryAmountRecalculationService.php:109-141
private function calculateTransactions(UserGroup $userGroup, TransactionCurrency $currency): void
{
    $set = DB::table('transactions')
        ->join('transaction_journals', 'transaction_journals.id', '=', 'transactions.transaction_journal_id')
        ->where('transaction_journals.user_group_id', $userGroup->id)
        ->where(...) // 筛选需要重算的交易
        ->get(['transactions.id']);

    TransactionObserver::$recalculate = false;
    foreach ($set as $item) {
        $transaction = Transaction::find($item->id);
        $transaction?->touch(); // 触发观察者进行 native_amount 重算
    }
    TransactionObserver::$recalculate = true;
}
```

---

## 4. 关键边界条件梳理

### 4.1 预算周期边界

| 边界情况 | 处理逻辑 | 代码位置 |
|----------|----------|----------|
| 预算限制开始/结束日期与报表周期完全一致 | 直接使用全部预算金额 | `BudgetLimitRepository.php:94-97` |
| 预算限制部分重叠报表周期 | 按重叠天数 × 日均金额计算 | `BudgetLimitRepository.php:99-101` |
| 单日预算限制 | 按 1 天计算日均金额 | `getDailyAmount()` |
| 预算限制日期早于报表周期开始 | 仅计算重叠部分 | 三分支检测逻辑 |
| 预算限制日期晚于报表周期结束 | 仅计算重叠部分 | 三分支检测逻辑 |

### 4.2 分类查询边界

| 边界情况 | 处理逻辑 | 代码位置 |
|----------|----------|----------|
| 交易无分类 | `listExpenses()` 中跳过（`continue`），`listIncome()` 中归为"无分类" | `OperationsRepository.php:130-131, 212-214` |
| 无分类收入/支出 | 通过 `NoCategoryRepository` 单独查询 | `CategoryReportGenerator.php:72-73` |
| 向负债账户的支出 | 排除（不作为支出统计） | `OperationsRepository.php:112-113` |
| 从负债账户的收入 | 排除（不作为收入统计） | `OperationsRepository.php:193-194` |
| 账户间转账 | 单独统计（`listTransferredIn/Out`） | `CategoryReportGenerator.php:69-70` |

### 4.3 汇率换算边界

| 边界情况 | 处理逻辑 | 代码位置 |
|----------|----------|----------|
| 汇率功能全局禁用 | 直接返回原金额，不进行换算 | `ExchangeRateConverter.php:63-66` |
| 源货币 = 目标货币 | 返回汇率 = 1 | `ExchangeRateConverter.php:88-91` |
| 无任何可用汇率（含备份） | 返回汇率 = 1（1:1 换算） | `ExchangeRateConverter.php:275-278` |
| 汇率为 0 | 视为无效，继续回退 | `ExchangeRateConverter.php:220-224` |
| 查询日期早于所有数据库汇率（主流货币） | 进入回退流程，**使用配置中的静态备份汇率**，而非 1:1 | `ExchangeRateConverter.php:161-168` |
| 查询日期早于所有数据库汇率（小众货币） | 进入回退流程，配置无备份汇率，最终返回 1 | `ExchangeRateConverter.php:171, 275-278` |
| 查询日期在汇率之间 | 取查询日期前最近的历史汇率（倒序取第一条） | `ExchangeRateConverter.php:208-210` |
| 查询日期晚于所有汇率 | 取最新可用汇率（倒序取第一条） | `ExchangeRateConverter.php:208-210` |
| 外币恰好是本位币 | 直接使用 `foreign_amount`，无需换算 | `TransactionSummarizer.php:81-89` |

> **特别说明**："早于所有汇率"的场景下，是否回落到 1:1 取决于该货币是否在 `config/cer.php` 的 `rates` 数组中有备份汇率。目前配置中包含约 30 种主流货币（USD, GBP, JPY, CNY 等）。

### 4.4 金额计算边界

| 边界情况 | 处理逻辑 | 代码位置 |
|----------|----------|----------|
| 支出金额 > 预算金额 | 剩余 = 0，超支 = 支出 - 预算 | `BudgetReportGenerator.php:297-298` |
| 支出金额 <= 预算金额 | 剩余 = 预算 - 支出，超支 = 0 | `BudgetReportGenerator.php:297-298` |
| 金额为 0 | 不累加到支出/收入统计 | 多处 `bccomp()` 判断 |
| 负数金额 | 识别为支出，使用 `Steam::negative()` | 多处 |
| 正数金额 | 识别为收入，使用 `Steam::positive()` | 多处 |

---

## 5. 代码引用索引

### 5.1 预算周期相关

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| BudgetLimit 模型 | `app/Models/BudgetLimit.php` | 全文件 |
| 预算限制查询（重叠检测） | `app/Repositories/Budget/BudgetLimitRepository.php` | 208-254 |
| 预算金额按日分摊 | `app/Repositories/Budget/BudgetLimitRepository.php` | 61-105 |
| 日均金额计算 | `app/Repositories/Budget/BudgetLimitRepository.php` | 256-270 |
| 预算报表生成器 | `app/Support/Report/Budget/BudgetReportGenerator.php` | 全文件 |
| 预算交易汇总 | `app/Support/Report/Budget/BudgetReportGenerator.php` | 289-333 |

### 5.2 分类统计相关

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 分类报表生成器 | `app/Support/Report/Category/CategoryReportGenerator.php` | 全文件 |
| 分类操作仓库 | `app/Repositories/Category/OperationsRepository.php` | 全文件 |
| 支出列表查询 | `app/Repositories/Category/OperationsRepository.php` | 105-180 |
| 收入列表查询 | `app/Repositories/Category/OperationsRepository.php` | 187-261 |
| GroupCollector 字段定义 | `app/Helpers/Collector/GroupCollector.php` | 109-161 |

### 5.3 汇率换算相关

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 汇率转换器核心 | `app/Support/Http/Api/ExchangeRateConverter.php` | 全文件 |
| 四级回退汇率查询 | `app/Support/Http/Api/ExchangeRateConverter.php` | 239-287 |
| 数据库汇率查询 | `app/Support/Http/Api/ExchangeRateConverter.php` | 174-234 |
| EUR 中介汇率 | `app/Support/Http/Api/ExchangeRateConverter.php` | 142-172 |
| 交易汇总器（币种选择） | `app/Support/Report/Summarizer/TransactionSummarizer.php` | 46-152 |
| 本位币金额重算服务 | `app/Services/Internal/Recalculate/PrimaryAmountRecalculationService.php` | 全文件 |
| Transaction 模型（native 字段） | `app/Models/Transaction.php` | 51-63 |

### 5.4 配置与依赖

| 项目 | 配置路径 |
|------|----------|
| 汇率功能开关 | `config/cer.enabled` |
| 备份汇率 | `config/cer.rates.*` |
| 精度处理 | `bcmath` 扩展（PHP 内置） |
| 周期计算 | `spatie/period` 库 |

---

## 6. 设计特点总结

1. **双轨存储**：原始金额 + 本位币预换算金额，兼顾准确性与性能
2. **多重缓存**：请求内缓存 + 属性缓存 + Laravel 缓存，三级缓存策略
3. **渐进回退**：汇率查询从直接匹配 → 反向匹配 → EUR 三角换算 → 配置备份汇率，五级回退确保可用性
4. **保守策略**：查询日期早于所有汇率时，不使用未来汇率，而是优先使用配置中的静态备份汇率
5. **分层兜底**：主流货币通过配置备份汇率兜底，小众货币才回落到 1:1 换算，平衡准确性和可用性
6. **时间精度**：所有日期比较使用明确的时间边界（00:00:00 / 23:59:59）
7. **安全计算**：全部使用 `bc*` 系列函数进行任意精度数学运算，避免浮点误差

---

*文档生成时间：2026-05-19*
*分析基于 Firefly III 开发分支代码*
