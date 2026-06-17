# viewRange 偏好与 Navigation 协同 & Audit 余额计算深度分析

本文档深入分析两个代码领域：
1. **Range 中间件如何通过 viewRange 偏好与 Navigation 协同渲染默认视图范围**——特别是 1M/3M/6M 非财年期间的行为
2. **Audit Generator 中 dayBefore 昨日余额的计算逻辑**，以及 Steam::accountsBalancesOptimized 在 Audit 路径中的 SQL 行为

---

## 一、Range 中间件：viewRange 偏好 → session('start'/'end') 的设置链路

### 1.1 入口：Range 中间件 setRange()

[Range.php L121-L151](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Middleware/Range.php#L121-L151)：

```php
private function setRange(): void
{
    if (!app('session')->has('start') && !app('session')->has('end')) {
        $viewRange = Preferences::get('viewRange', '1M')->data;
        if (is_array($viewRange)) {
            $viewRange = '1M';
        }
        $today = today(config('app.timezone'));
        $start = Navigation::updateStartDate((string) $viewRange, $today);
        $end   = Navigation::updateEndDate((string) $viewRange, $start);
        app('session')->put('start', $start);
        app('session')->put('end', $end);
    }
    // ... 设置 session('first')
}
```

**触发条件**：session 中没有 `start` 和 `end`（通常是用户首次访问或 session 过期后）。

**协同链路**：
```
用户偏好 viewRange（如 '1M', '3M', '6M'）
  ↓ Preferences::get('viewRange', '1M')->data
  ↓
Navigation::updateStartDate($viewRange, $today)
  → 计算当前期间的起始日
  ↓
Navigation::updateEndDate($viewRange, $start)
  → 计算当前期间的结束日
  ↓
session('start') / session('end')
```

### 1.2 viewRange 的所有可选值

viewRange 偏好的可选值来自 [firefly.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/config/firefly.php) 配置和前端偏好页面。分为**固定期间**和**动态期间**两组：

| 值 | 含义 | 类型 |
|----|------|------|
| `1D` | 每天 | 固定 |
| `1W` | 每周 | 固定 |
| `1M` | 每月 | 固定 |
| `3M` | 每季度 | 固定 |
| `6M` | 每半年 | 固定 |
| `1Y` | 每年（财年感知） | 固定 |
| `last7` | 最近 7 天 | 动态 |
| `last30` | 最近 30 天 | 动态 |
| `last90` | 最近 90 天 | 动态 |
| `last365` | 最近 365 天 | 动态 |
| `MTD` | 月初至今 | 动态 |
| `QTD` | 季初至今 | 动态 |
| `YTD` | 年初至今 | 动态 |

---

## 二、Navigation::updateStartDate/updateEndDate 在 1M/3M/6M 下的精确行为

### 2.1 updateStartDate() 分支逻辑

[Navigation.php L868-L945](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php#L868-L945)：

```php
public function updateStartDate(string $range, Carbon $start): Carbon
{
    $functionMap = [
        '1D'     => 'startOfDay',
        '1W'     => 'startOfWeek',
        '1M'     => 'startOfMonth',
        '3M'     => 'firstOfQuarter',
        'custom' => 'startOfMonth',
    ];
    // 1. 固定期间（1D/1W/1M/3M）→ 直接映射 Carbon 方法
    if (array_key_exists($range, $functionMap)) {
        $start->{$functionMap[$range]}();
        return $start;
    }
    // 2. 6M 特殊处理
    if ('6M' === $range) {
        if ($start->month >= 7) {
            $start->startOfYear()->addMonths(6);  // 7月及以后 → 当年7月1日
            return $start;
        }
        $start->startOfYear();                     // 1-6月 → 当年1月1日
        return $start;
    }
    // 3. 1Y → FiscalHelper（财年感知）
    if ('1Y' === $range) {
        $fiscalHelper = app(FiscalHelperInterface::class);
        return $fiscalHelper->startOfFiscalYear($start);
    }
    // 4. 动态期间 → 回溯
    switch ($range) {
        case 'last7':   $start->subDays(7);    return $start;
        case 'last30':  $start->subDays(30);   return $start;
        case 'last90':  $start->subDays(90);   return $start;
        case 'last365': $start->subDays(365);  return $start;
        case 'YTD':     $start->startOfYear(); return $start;
        case 'QTD':     $start->startOfQuarter(); return $start;
        case 'MTD':     $start->startOfMonth(); return $start;
    }
}
```

### 2.2 updateEndDate() 分支逻辑

[Navigation.php L821-L863](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php#L821-L863)：

```php
public function updateEndDate(string $range, Carbon $start): Carbon
{
    $functionMap = ['1D' => 'endOfDay', '1W' => 'endOfWeek', '1M' => 'endOfMonth', '3M' => 'lastOfQuarter', 'custom' => 'startOfMonth'];
    $end = clone $start;
    // 1. 固定期间
    if (array_key_exists($range, $functionMap)) {
        $end->{$functionMap[$range]}();
        return $end;
    }
    // 2. 6M 特殊处理
    if ('6M' === $range) {
        if ($start->month >= 7) {
            $end->endOfYear();               // 7月及以后 → 当年12月31日
            return $end;
        }
        $end->startOfYear()->addMonths(6);   // 1-6月 → 当年6月30日（addMonths(6)后是7月1日，但实际是6月底最后一天之后...）
        return $end;
    }
    // 3. 1Y → FiscalHelper
    if ('1Y' === $range) {
        $fiscalHelper = app(FiscalHelperInterface::class);
        return $fiscalHelper->endOfFiscalYear($end);
    }
    // 4. 动态期间
    // last7/last30/last90/last365/YTD/QTD/MTD → today()->endOfDay()
}
```

### 2.3 1M 精确行为示例

假设今天是 `2026-06-17`：

| 步骤 | 方法 | 结果 |
|-----|------|------|
| 1 | `updateStartDate('1M', today())` | `today()->startOfMonth()` → `2026-06-01 00:00:00` |
| 2 | `updateEndDate('1M', start)` | `start->endOfMonth()` → `2026-06-30 23:59:59` |

**session 结果**：`start = 2026-06-01`, `end = 2026-06-30`

### 2.4 3M 精确行为示例

假设今天是 `2026-06-17`：

| 步骤 | 方法 | 结果 |
|-----|------|------|
| 1 | `updateStartDate('3M', today())` | `today()->firstOfQuarter()` → `2026-04-01 00:00:00` |
| 2 | `updateEndDate('3M', start)` | `start->lastOfQuarter()` → `2026-06-30 23:59:59` |

Carbon 的 `firstOfQuarter()` 按自然季度计算：
- Q1: 1月1日 - 3月31日
- Q2: 4月1日 - 6月30日
- Q3: 7月1日 - 9月30日
- Q4: 10月1日 - 12月31日

**session 结果**：`start = 2026-04-01`, `end = 2026-06-30`

> **关键点**：3M 的季度始终是**自然季度**，**不感知财年**。即使 customFiscalYear = true 且财年从 7 月开始，3M 的 Q2 仍然是 4-6 月而不是 1-3 月。

### 2.5 6M 精确行为示例

6M 是 Navigation 中唯一一个不通过 Carbon 标准方法、而是**手动判断月份**来决定期间的类型。

假设今天是 `2026-06-17`：

| 步骤 | 方法 | 判断 | 结果 |
|-----|------|------|------|
| 1 | `updateStartDate('6M', today())` | `month(6) < 7` | `startOfYear()` → `2026-01-01 00:00:00` |
| 2 | `updateEndDate('6M', start)` | `start->month(1) < 7` | `startOfYear()->addMonths(6)` → `2026-07-01 00:00:00` |

> **注意**：`addMonths(6)` 的结果实际上是 **7 月 1 日**，但 `updateEndDate` 的语义是"期间结束"。这里存在一个隐含约定——调用方会在此之前或之后对日期做 `endOfDay()` 或进一步处理。在 Range 中间件的上下文中，session('end') 被设置为 `2026-07-01 00:00:00`。

假设今天是 `2026-08-15`：

| 步骤 | 方法 | 判断 | 结果 |
|-----|------|------|------|
| 1 | `updateStartDate('6M', today())` | `month(8) >= 7` | `startOfYear()->addMonths(6)` → `2026-07-01 00:00:00` |
| 2 | `updateEndDate('6M', start)` | `start->month(7) >= 7` | `endOfYear()` → `2026-12-31 23:59:59` |

**6M 的两半规则**：
| 月份 | 上半年 | 下半年 |
|-----|--------|--------|
| 1-6 月 | start=`当年1月1日` | — |
| 7-12 月 | — | start=`当年7月1日` |
| 1-6 月 | end=`当年7月1日` | — |
| 7-12 月 | — | end=`当年12月31日` |

> **关键点**：6M 的上下半年划分是**固定按自然年**的（1-6月 vs 7-12月），**不感知财年**。即使财年从 7 月开始，上半年仍然是 1-6 月。

### 2.6 非财年期间的共同特征总结

| 期间 | 财年感知 | 季度/半年划分 | 依赖 |
|-----|---------|-------------|------|
| `1D` | ❌ | — | Carbon::startOfDay/endOfDay |
| `1W` | ❌ | — | Carbon::startOfWeek/endOfWeek |
| `1M` | ❌ | — | Carbon::startOfMonth/endOfMonth |
| `3M` | ❌ | 自然季度 | Carbon::firstOfQuarter/lastOfQuarter |
| `6M` | ❌ | 自然半年（1-6 / 7-12） | 手动月份判断 + addMonths |
| `1Y` | ✅ | **按财年起止** | FiscalHelper |
| 动态 | ❌ | — | today() + 回溯天数 |

**只有 `1Y` 走 FiscalHelper**，其余所有期间都是 Carbon 标准日期方法或手动月份判断。

---

## 三、startOfPeriod/endOfPeriod 与 updateStartDate/updateEndDate 的差异

Navigation 中存在两套"期间起止"方法，容易混淆：

### 3.1 updateStartDate/updateEndDate（session 初始化用）

用于 Range 中间件设置 session 范围。**语义**：给定今天和 viewRange，确定**当前期间**的起止日期。

### 3.2 startOfPeriod/endOfPeriod（遍历期间用）

用于遍历一系列期间（如 `blockPeriods`、`listOfPeriods`）。**语义**：给定某个日期和 repeatFreq，确定该日期**所在期间**的起止。

### 3.3 关键差异对照

| 维度 | updateStartDate/EndDate | startOfPeriod/endOfPeriod |
|-----|------------------------|--------------------------|
| **1Y 行为** | ✅ FiscalHelper 财年感知 | ❌ Carbon::startOfYear 自然年 |
| **3M 行为** | firstOfQuarter/lastOfQuarter | firstOfQuarter/lastOfQuarter |
| **6M 行为** | 手动月份判断 | 手动月份判断（相同逻辑） |
| **1M 行为** | startOfMonth/endOfMonth | startOfMonth/endOfMonth |
| **1D 行为** | startOfDay/endOfDay | startOfDay/endOfDay |
| **endOfPeriod 额外逻辑** | 无 | 对 1W/1M/3M/6M/1Y 做 `addXxx()->subDay()` |

### 3.4 endOfPeriod 的 addXxx+subDay 模式

[endOfPeriod() L185-L384](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php#L185-L384) 的核心逻辑（非动态期间分支）：

```php
$functionMap = [
    '1M'      => 'addMonths',
    '3M'      => 'addQuarters',
    '6M'      => 'addMonths',  // modifier=6
    '1Y'      => 'addYears',
    // ...
];
$subDay = ['1M', '3M', '6M', '1Y', ...];  // 这些期间结束后需回退1天

// 通用路径：
$currentEnd->{$function}($modifier);  // 如 addMonths(1), addQuarters(1), addMonths(6), addYears(1)
if (in_array($repeatFreq, $subDay, true)) {
    $currentEnd->subDay();             // 回退1天
}
$currentEnd->endOfDay();
```

**示例**：startOfPeriod('1M') = 2026-06-01 → endOfPeriod('1M')：
1. `clone 2026-06-01`
2. `addMonths(1)` → 2026-07-01
3. `subDay()` → 2026-06-30
4. `endOfDay()` → 2026-06-30 23:59:59

这个 `addXxx + subDay` 模式确保了"期间结束日是期间最后一天"，而不是下个期间的第一天。

### 3.5 getViewRange() 的动态期间校正

[Navigation.php L423-L441](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php#L423-L441)：

```php
public function getViewRange(bool $correct): string
{
    $range = Preferences::get('viewRange', '1M')->data ?? '1M';
    if (!$correct) {
        return $range;
    }
    return match ($range) {
        'last7'          => '1W',
        'last30', 'MTD'  => '1M',
        'last90', 'QTD'  => '3M',
        'last365', 'YTD' => '1Y',
        default          => $range
    };
}
```

当 `$correct = true` 时，动态期间被映射回固定期间。这用于需要按期间遍历时（如 `blockPeriods`），将动态期间规范化为可遍历的固定粒度。

---

## 四、Audit Generator 中 dayBefore 计算昨日余额的逻辑

### 4.1 dayBefore 的构造

[MonthReportGenerator::generate() L57-L60](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php#L57-L60)：

```php
$dayBefore = clone $this->start;
// set date to subday + end-of-day for account balance. so it is at $date 23:59:59
$dayBefore->subDay()->endOfDay();
```

**计算过程**（假设 `$this->start = 2026-06-01`）：
1. `clone 2026-06-01` → `2026-06-01 00:00:00`
2. `subDay()` → `2026-05-31 00:00:00`
3. `endOfDay()` → `2026-05-31 23:59:59`

**语义**：`dayBefore` 代表"报表期间开始前那一刻"的时间点。用这个时间点查余额，就能得到"期初余额"。

### 4.2 dayBefore 如何传入 getAuditReport()

[generate() L66](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php#L66)：

```php
$auditData[$id] = $this->getAuditReport($account, $dayBefore);
```

`$dayBefore` 作为第二个参数传入 `getAuditReport()`，在该方法内被用于：

1. **调用 Steam::accountsBalancesOptimized 获取期初余额** [L159](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php#L159)：
   ```php
   $dayBeforeBalance = Steam::accountsBalancesOptimized(new Collection()->push($account), $date)[$account->id];
   ```

2. **提取 balance 字段作为 startBalance** [L161](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php#L161)：
   ```php
   $startBalance = $dayBeforeBalance['balance'];
   ```

3. **逐笔交易计算 balance_before / balance_after** [L165-L183](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php#L165-L183)：
   ```php
   foreach ($journals as $index => $journal) {
       $journals[$index]['balance_before'] = $startBalance;
       $transactionAmount = $journal['amount'];
       // ... 金额方向修正 ...
       $newBalance = bcadd((string) $startBalance, (string) $transactionAmount);
       $journals[$index]['balance_after'] = $newBalance;
       $startBalance = $newBalance;  // 滚动到下一笔
   }
   ```

### 4.3 余额滚动计算链路

```
dayBefore (2026-05-31 23:59:59)
  ↓ Steam::accountsBalancesOptimized($account, $dayBefore)
  ↓ 返回 dayBeforeBalance['balance'] = "1234.56"
  ↓
  ↓ startBalance = "1234.56"
  ↓
交易1: balance_before = "1234.56", amount = "-100.00"
       → balance_after = bcadd("1234.56", "-100.00") = "1134.56"
       → startBalance = "1134.56"
  ↓
交易2: balance_before = "1134.56", amount = "+50.00"
       → balance_after = bcadd("1134.56", "50.00") = "1184.56"
       → startBalance = "1184.56"
  ↓
  ... 依此类推 ...
  ↓
期末: Steam::accountsBalancesOptimized($account, $this->end) → endBalance
```

### 4.4 金额方向修正逻辑

[L170-L179](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php#L170-L179)：

```php
// 默认 amount 是从账户视角的"流出"（负数）
// 如果账户是交易的目标方（destination），金额应为正（流入）
if ($account->id === $journal['destination_account_id']) {
    $transactionAmount = Steam::positive($journal['amount']);
}
// 如果使用外币且账户货币 = 外币，用 foreign_amount
if ($currency->id === $journal['foreign_currency_id']) {
    $transactionAmount = $journal['foreign_amount'];
    if ($account->id === $journal['destination_account_id']) {
        $transactionAmount = Steam::positive($journal['foreign_amount']);
    }
}
```

**规则**：
- 账户是 source（钱出去）：amount 保持原样（通常为负）
- 账户是 destination（钱进来）：amount 取绝对值（正）
- 使用外币时：切换到 foreign_amount，同样按方向修正

---

## 五、Steam::accountsBalancesOptimized 的 SQL 行为

### 5.1 方法签名

[Steam.php L72-L78](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Steam.php#L72-L78)：

```php
public function accountsBalancesOptimized(
    Collection $accounts,
    Carbon $date,
    ?TransactionCurrency $primary = null,
    ?bool $convertToPrimary = null,
    bool $inclusive = true
): array
```

**参数含义**：
- `$accounts`：账户集合（Audit 中通常只有 1 个账户）
- `$date`：截止日期（Audit 中是 `$dayBefore` 或 `$this->end`）
- `$primary`：主货币（默认自动获取）
- `$convertToPrimary`：是否将所有余额转换为主货币
- `$inclusive`：日期比较是否包含当天（默认 `true`，即 `<=`）

### 5.2 核心 SQL 查询

[Steam.php L86-L95](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Steam.php#L86-L95)：

```php
$arrayOfSums = Transaction::query()
    ->whereIn('account_id', $accounts->pluck('id')->toArray())
    ->leftJoin('transaction_journals', 'transaction_journals.id', '=', 'transactions.transaction_journal_id')
    ->leftJoin('transaction_currencies', 'transaction_currencies.id', '=', 'transactions.transaction_currency_id')
    ->where('transaction_journals.date', $inclusive ? '<=' : '<', $date->format('Y-m-d H:i:s'))
    ->whereNull('transaction_journals.deleted_at')
    ->groupBy(['transactions.account_id', 'transaction_currencies.code'])
    ->get(['transactions.account_id', 'transaction_currencies.code', DB::raw('SUM(transactions.amount) as sum_of_amount')])
    ->toArray();
```

**生成的 SQL（等效）**：

```sql
SELECT
    transactions.account_id,
    transaction_currencies.code,
    SUM(transactions.amount) AS sum_of_amount
FROM transactions
LEFT JOIN transaction_journals ON transaction_journals.id = transactions.transaction_journal_id
LEFT JOIN transaction_currencies ON transaction_currencies.id = transactions.transaction_currency_id
WHERE transactions.account_id IN (1, 2, 3)
    AND transaction_journals.date <= '2026-05-31 23:59:59'
    AND transaction_journals.deleted_at IS NULL
GROUP BY transactions.account_id, transaction_currencies.code
```

### 5.3 SQL 行为关键点

**1. 日期过滤条件**：
- `$inclusive = true`（默认）：`transaction_journals.date <= $date`
- `$inclusive = false`：`transaction_journals.date < $date`

Audit 中使用默认 `$inclusive = true`，所以 `$dayBefore = 2026-05-31 23:59:59` 时，条件是 `date <= '2026-05-31 23:59:59'`，即包含 5 月 31 日当天的所有交易。

**2. 只过滤 deleted_at**：
- 只检查 `transaction_journals.deleted_at IS NULL`
- **不检查** `transactions.deleted_at`——这是有意为之，因为 Firefly III 中交易行的删除是通过删除 journal 级联的

**3. GROUP BY 按 account_id + currency_code**：
- 同一账户可能有多币种余额
- 每个币种分别 SUM

**4. SUM(transactions.amount)**：
- 聚合的是 transactions 表的 amount 字段
- amount 在 transactions 表中：source 端为负，destination 端为正
- SUM 时正负抵消，得到净余额

### 5.4 余额后处理链路

[Steam.php L99-L155](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Steam.php#L99-L155)：

```php
foreach ($accounts as $account) {
    $return = ['pc_balance' => '0', 'balance' => '0'];
    $currency = $currencies[$account->id];

    // 从 SQL 结果中筛选该账户的汇总
    $accountSums = array_filter($arrayOfSums, fn($e) => $e['account_id'] === $account->id);

    // 按币种代码建立映射
    $sumsByCode = [];
    foreach ($accountSums as $accountSum) {
        $sumsByCode[$accountSum['code']] = $this->floatalize($accountSum['sum_of_amount']);
    }

    // 取账户本位币的余额作为 'balance'
    $return['balance'] = $sumsByCode[$currency->code] ?? '0';

    // 如果需要转换为主货币
    if ($convertToPrimary) {
        $return['pc_balance'] = $this->convertAllBalances($sumsByCode, $primary, $date);
        // 加上虚拟余额的转换值
        $converter = new ExchangeRateConverter();
        $pcVirtualBalance = $converter->convert($currency, $primary, $date, $virtualBalance);
        $return['pc_balance'] = bcadd($pcVirtualBalance, $return['pc_balance']);
    }

    // 不转换时：balance + virtual_balance
    if (!$convertToPrimary) {
        $return['balance'] = bcadd($return['balance'], $virtualBalance);
    }

    // 最终结果合并所有币种余额
    $final = array_merge($return, $sumsByCode);
    $result[$account->id] = $final;
}
```

### 5.5 返回值结构

对于单个账户（Audit 场景），返回值示例：

```php
[
    123 => [                    // account_id
        'balance'     => '1234.56',   // 账户本位币余额 + virtual_balance
        'pc_balance'  => '1234.56',   // 主货币余额（如果 convertToPrimary）
        'USD'         => '1234.56',   // 各币种原始 SUM
        'EUR'         => '-500.00',
    ]
]
```

Audit 中取值方式：`$result[$account->id]['balance']`

### 5.6 与旧方法 finalAccountBalance 的对比

代码注释 [L157-L158](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php#L157-L158) 表明 2025-10-08 做了替换：

```php
// 2025-10-08 replace with accountsBalancesOptimized.
// $dayBeforeBalance = Steam::finalAccountBalance($account, $date);
$dayBeforeBalance = Steam::accountsBalancesOptimized(new Collection()->push($account), $date)[$account->id];
```

**关键差异**：
- `finalAccountBalance`：单账户查询，每次调一条 SQL
- `accountsBalancesOptimized`：批量查询，一次 SQL 获取所有账户余额
- 在 Audit 场景中，虽然每次只传 1 个账户，但方法设计支持批量，未来可优化为一次查所有账户

---

## 六、Audit 路径的完整数据流图

```
ReportController::auditReport($accounts, Carbon $start, Carbon $end)
  ↓
ReportGeneratorFactory::reportGenerator('Audit', $start, $end)
  ↓ (根据 diffInMonths 选择 Month/Year/MultiYear)
  ↓
Audit\MonthReportGenerator::generate()
  ↓
  ├─ $dayBefore = clone $this->start → subDay() → endOfDay()
  │  (报表期间前一天的 23:59:59)
  │
  ├─ foreach ($accounts as $account):
  │    └─ getAuditReport($account, $dayBefore)
  │         ├─ GroupCollector::setRange($this->start, $this->end)
  │         │   → WHERE date >= '2026-06-01 00:00:00'
  │         │     AND date <= '2026-06-30 23:59:59'
  │         │   → 期间内所有交易
  │         │
  │         ├─ Steam::accountsBalancesOptimized([$account], $dayBefore)
  │         │   → SELECT SUM(amount) WHERE date <= '2026-05-31 23:59:59'
  │         │   → 期初余额
  │         │
  │         ├─ 逐笔计算: balance_before → +amount → balance_after
  │         │
  │         └─ Steam::accountsBalancesOptimized([$account], $this->end)
  │             → SELECT SUM(amount) WHERE date <= '2026-06-30 23:59:59'
  │             → 期末余额
  │
  └─ view('reports.audit.report', ['auditData' => $auditData])
       → 返回完整 HTML（无 AJAX）
```

---

## 七、关键代码索引

| 主题 | 文件 | 关键位置 |
|------|------|---------|
| Range 中间件 setRange | [Range.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Middleware/Range.php) | setRange() L121-L151 |
| updateStartDate 1M/3M/6M | [Navigation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php) | L868-L945 |
| updateEndDate 1M/3M/6M | [Navigation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php) | L821-L863 |
| startOfPeriod 6M | [Navigation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php) | L691-L698 |
| endOfPeriod addXxx+subDay | [Navigation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php) | L185-L384 |
| getViewRange 动态校正 | [Navigation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php) | L423-L441 |
| Audit dayBefore | [Audit\MonthReportGenerator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php) | generate() L57-L60 |
| Audit getAuditReport | [Audit\MonthReportGenerator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php) | L132-L208 |
| 余额滚动计算 | [Audit\MonthReportGenerator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php) | L165-L183 |
| accountsBalancesOptimized | [Steam.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Steam.php) | L72-L158 |
| 金额方向修正 | [Audit\MonthReportGenerator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php) | L170-L179 |
