# 月度报表与周期统计聚合流程详解

本文档梳理 Firefly III 中月度报表与周期统计从原始交易数据到最终图表展示的完整数据流转过程，分为**统计区间切分**、**聚合查询**、**图表数据装配**三大阶段，并在末尾独立分析周期统计缓存的失效机制。

---

## 一、整体数据流

```
原始交易(Transaction)
    ↓
[阶段1: 统计区间切分] Navigation
    ↓ 按周期(月/季/年)切分时间轴
[阶段2: 聚合查询] PeriodOverview + PeriodStatisticRepository
    ↓ 按币种/类型聚合，缓存到 period_statistics 表
[阶段3: 图表数据装配] ChartGeneration + ChartJsGenerator
    ↓ 装配为 Chart.js 格式
前端图表展示
```

---

## 二、第一阶段：统计区间切分

### 2.1 核心组件

**核心类**：app/Support/Navigation.php

### 2.2 关键方法

#### `getViewRange(bool $correct): string`
- **位置**：app/Support/Navigation.php #L423-L441
- **作用**：从用户偏好中读取视图范围（viewRange），并对动态范围进行标准化转换。
- **转换规则**：
  - `last7` → `1W`
  - `last30` / `MTD` → `1M`
  - `last90` / `QTD` → `3M`
  - `last365` / `YTD` → `1Y`

#### `blockPeriods(Carbon $start, Carbon $end, string $range): array`
- **位置**：app/Support/Navigation.php #L89-L132
- **作用**：将给定的起止日期区间，按指定周期粒度切分成若干个完整的周期块。
- **算法逻辑**：
  1. 从 `$end` 日期开始向前回溯
  2. 先按 `$range` 粒度生成最多 13 个周期块（保证足够细粒度）
  3. 如果范围还不够，继续按 `1Y` 粒度向后补充（最多 20 个年周期）
  4. 每个周期块包含 `start`、`end`、`period` 三个字段
- **输出示例**（range=1M）：
  ```
  [
      ['start' => '2024-01-01 00:00:00', 'end' => '2024-01-31 23:59:59', 'period' => '1M'],
      ['start' => '2024-02-01 00:00:00', 'end' => '2024-02-29 23:59:59', 'period' => '1M'],
  ]
  ```

#### `startOfPeriod(Carbon $theDate, string $repeatFreq): Carbon`
- **位置**：app/Support/Navigation.php #L651-L724
- **作用**：计算给定日期所在周期的起始时间点。

#### `endOfPeriod(Carbon $end, string $repeatFreq): Carbon`
- **位置**：app/Support/Navigation.php #L185-L384
- **作用**：计算给定日期所在周期的结束时间点。
- **关键细节**：周/月/季/年等周期的结束时间会 `subDay()` 再 `endOfDay()->milli(0)`，保证区间为**闭区间**。例如：
  - 1月周期：start=2024-01-01 00:00:00，end=2024-01-31 23:59:59.999
  - 1周周期：start=周一 00:00:00，end=周日 23:59:59.999

#### `getPeriodFromBlocks(array $dates, Carbon $start, Carbon $end): array`
- **位置**：app/Support/Http/Controllers/PeriodOverview.php #L362-L379
- **作用**：扩展起止日期以对齐到周期块的边界。遍历切好的周期块，取所有块中最小的 start 和最大的 end，确保后续查询能覆盖所有周期。

### 2.3 切分流程调用链

以账户周期概览为例（app/Support/Http/Controllers/PeriodOverview.php #L92-L114）：
```
getAccountPeriodOverview()
    → Navigation::getViewRange(true)         // 获取标准化视图范围
    → Navigation::blockPeriods(start, end, range)  // 切分周期块
    → getPeriodFromBlocks()                  // 扩展起止日期对齐完整周期
    → 遍历每个周期块，逐一聚合
```

---

## 三、第二阶段：聚合查询

聚合查询采用**懒加载 + 缓存**的两层架构：先查预计算的 `period_statistics` 表，没命中则从原始交易实时计算并写回缓存。

### 3.1 数据模型

**PeriodStatistic 模型**：app/Models/PeriodStatistic.php

| 字段 | 说明 |
|------|------|
| `start` / `end` | 统计周期的起止时间（通过 SeparateTimezoneCaster 从 start_tz/end_tz 字段还原时区） |
| `type` | 统计类型（spent、earned、transferred_in、transferred_away 等） |
| `amount` | 聚合金额（字符串，高精度） |
| `count` | 交易笔数 |
| `transaction_currency_id` | 币种 ID |
| `primary_statable_type` / `primary_statable_id` | 多态关联对象（Account/Category/Tag 等） |

**时区还原**：app/Casts/SeparateTimezoneCaster.php
- 从 DB 读 `start` 字段时，同时读取 `start_tz` 作为时区解析
- 若 `start_tz` 为空，回退到 `config('app.timezone')`
- 最终统一设置为应用时区，保证比较一致性

对于无具体关联对象的统计，使用前缀类型：
- `no_category_spent` / `no_category_earned` — 无分类的支出/收入
- `no_budget_spent` — 无预算的支出
- `all_withdrawal` / `all_deposit` / `all_transfer` — 全局交易统计

### 3.2 周期统计缓存的区间边界处理（查询/保存侧）

周期统计缓存的边界匹配使用**精确匹配 + 范围查询**双模式：

1. **批量查询（范围模式）**：app/Repositories/PeriodStatistic/PeriodStatisticRepository.php
   - `allInRangeForModel()`：`WHERE start >= $start AND end <= $end`
   - `allInRangeForPrefix()`：同上，再加前缀 LIKE
   - 一次把范围内所有周期的统计全部查出，放内存中

2. **单周期命中（精确模式）**：app/Support/Http/Controllers/PeriodOverview.php
   - `filterStatistics()`：`$statistic->start->isSameSecond($start) && $statistic->end->isSameSecond($end)`
   - `filterPrefixedStatistics()`：`$statistic->start->eq($start) && $statistic->end->eq($end)`
   - 必须秒级完全一致，因为周期块的起止时间都是规整的（月初/月末）

3. **保存时的边界规整**：app/Repositories/PeriodStatistic/PeriodStatisticRepository.php
   - `saveStatistic()` / `savePrefixedStatistic()` 直接把周期块的 start/end 原样存入
   - 同时记录 `start_tz` / `end_tz`（即 `$start->format('e')`）
   - 周期块本身已由 Navigation 规整，所以边界天然一致

### 3.3 核心组件

#### PeriodStatisticRepository
app/Repositories/PeriodStatistic/PeriodStatisticRepository.php

关键方法：
- `allInRangeForModel($model, $start, $end)` — 按模型+时间范围查询统计
- `allInRangeForPrefix($prefix, $start, $end)` — 按前缀类型查询
- `findPeriodStatistic($model, $start, $end, $type)` — 精确查找单条
- `saveStatistic(...)` — 保存模型关联统计
- `savePrefixedStatistic(...)` — 保存前缀类型统计
- `deleteStatisticsForModel(...)` — 按模型+单个日期失效统计
- `deleteStatisticsForType(...)` — 按模型类型+对象集合+日期集合批量失效
- `deleteStatisticsForPrefix(...)` — 按前缀+日期集合批量失效

#### PeriodOverview Trait
app/Support/Http/Controllers/PeriodOverview.php

周期统计的核心业务逻辑 Trait，被多个控制器复用。

### 3.4 聚合流程（以单模型单周期为例）

核心方法：app/Support/Http/Controllers/PeriodOverview.php #L423-L511

```
1. 先从 $this->statistics（已预加载的内存集合）过滤匹配
   ↓ 命中
2a. 按币种装配数组（补充币种元信息）返回
   ↓ 未命中（懒加载触发）
2b. 按模型类型调用 periodCollection() 拉原始交易
    - Account → AccountRepository::periodCollection()
    - Category → CategoryRepository::periodCollection()
    - Tag → TagRepository::periodCollection()
2c. filterTransactionsByType() 按交易类型过滤：
    - spent → WITHDRAWAL（金额取反为负）
    - earned → DEPOSIT
    - transferred_in / transferred_away → 按转账方向区分正负
2d. groupByCurrency() 按币种聚合
2e. saveGroupedAsStatistics() 写回 period_statistics 表（每条币种一条记录）
    - 即使聚合结果为空，也会存一条 count=0, amount='0' 的记录
2f. 返回聚合结果
```

### 3.5 按币种聚合逻辑

**方法**：app/Support/Http/Controllers/PeriodOverview.php #L614-L664

聚合规则：
- 以 `currency_id` 为键
- 每个币种条目含：amount（累计）、count（笔数）、完整币种元信息
- 顶层额外有一个 `count` 字段记总笔数
- 使用 `bcadd()` 做高精度加法，避免浮点误差

**多币种换算（convertToPrimary）**：
由 Controller 基类 app/Http/Controllers/Controller.php #L137-L173 的中间件初始化：
```
$this->primaryCurrency   = Amount::getPrimaryCurrency();
$this->convertToPrimary  = Amount::convertToPrimary();
```

当 `convertToPrimary=true` 时，`groupByCurrency()` 做如下处理：
1. 若交易币种 ≠ 主币种 且 外币 ≠ 主币种 → 换用 `pc_amount`（预计算的主币种换算金额），币种元信息切换为主币种
2. 若交易币种 ≠ 主币种 但 外币 = 主币种 → 直接用 `foreign_amount` + 外币元信息
3. 否则按原币种处理

`resolveJournalAmountAndCurrency()`（app/Support/Http/Controllers/ResolvesJournalAmountAndCurrency.php）是图表控制器中使用的等价版本，逻辑一致，但专门给图表装配流程用。

### 3.6 报表聚合（AccountTasker：月度收支局部汇总）

月度报表页面的收支局部汇总走的是独立于 PeriodStatistic 的聚合路径。

**入口串联方式**：

1. 页面骨架：app/Http/Controllers/ReportController.php #L164-L183
   - `defaultReport()` 接收 accounts/start/end
   - 通过 `ReportGeneratorFactory::reportGenerator('Standard', $start, $end)` 选生成器
   - 月跨度 → `Standard\MonthReportGenerator`
   - 年跨度 → `Standard\YearReportGenerator`
   - 跨年 → `Standard\MultiYearReportGenerator`

2. 骨架渲染：app/Generator/Report/Standard/MonthReportGenerator.php
   - `generate()` 渲染 `resources/views/reports/default/month.twig`
   - 这个模板只是**页面壳子**：定义多个空 div（accountReport、incomeReport、expenseReport、incomeVsExpenseReport、budgetReport、categoryReport、balanceReport、billReport）
   - 每个 div 有独立 URL，由前端 JS 异步加载

3. 局部汇总异步端点：
   | 区域 | 路由端点 | 控制器方法 | 内部聚合 |
   |------|----------|-----------|----------|
   | 账户余额表 | report-data.account.general | Report\AccountController::general() | AccountTasker::getAccountReport() |
   | 收入列表 | report-data.operations.income | Report\OperationsController::income() | AccountTasker::getIncomeReport() |
   | 支出列表 | report-data.operations.expenses | Report\OperationsController::expenses() | AccountTasker::getExpenseReport() |
   | 收支对比表 | report-data.operations.operations | Report\OperationsController::operations() | Income + Expense 合并 |
   | 预算表 | report-data.budget.general | Report\BudgetController | 预算聚合 |
   | 分类表 | report-data.category.operations | Report\CategoryController | 分类聚合 |
   | 余额表 | report-data.balance.general | Report\BalanceController | 余额聚合 |
   | 账单表 | report-data.bills.overview | Report\BillController | 账单聚合 |

4. 每个局部汇总的聚合+渲染模式完全一致（以 Report\OperationsController::operations() 为例）：
   ```
   a. CacheProperties 查缓存（key = start + end + 标识 + accountIds）
   b. 未命中则：
      - 调 AccountTasker::getIncomeReport() 聚合收入
      - 调 AccountTasker::getExpenseReport() 聚合支出
      - 按币种合并 sums：in（收入和）、out（支出和）、sum（差额）
   c. 渲染 resources/views/reports/partials/operations.twig
   d. 把渲染好的 HTML 存缓存，返回字符串
   ```

**AccountTasker 的聚合实现**：app/Repositories/Account/AccountTasker.php

- `getExpenseReport($start, $end, $accounts)` #L119-L145：
  - GroupCollector：setSourceAccounts + excludeDestinationAccounts（排除内部互转）
  - setTypes：[WITHDRAWAL, TRANSFER]
  - `groupExpenseByDestination()`：按「destination_account_id + currency_id」为 key 聚合
  - 每条记录：sum、count、average（sum/count 当 count>1 时）
  - sums 维度：按 currency_id 汇总所有账户的支出和
  - 排序：按 sum 升序（支出最小在前）

- `getIncomeReport($start, $end, $accounts)` #L150-L173：
  - GroupCollector：setDestinationAccounts + excludeSourceAccounts
  - setTypes：[DEPOSIT, TRANSFER]
  - `groupIncomeBySource()`：按「source_account_id + currency_id」为 key 聚合
  - 金额 `bcmul(amount, '-1')` 取反（使收入显示为正数）
  - 排序：按 sum 降序（收入最大在前）

- `getAccountReport($accounts, $start, $end)` #L49-L114：
  - Steam::accountsBalancesInRange() 获取期初（start 前一天）和期末余额
  - 每个账户算 start_balance、end_balance
  - 按币种汇总 sums.start / sums.end / sums.difference
  - 特殊处理：若账户首笔流水恰好是期初当天的 OPENING_BALANCE，直接用其金额替代

### 3.7 TransactionSummarizer

app/Support/Report/Summarizer/TransactionSummarizer.php — 通用的交易聚合工具

- `groupByCurrencyId($journals, $method, $includeForeign)` — 按币种聚合
  - `method='positive'` → Steam::positive()（绝对值为正）
  - `method='negative'` → Steam::negative()（绝对值为负）
  - 同时处理原币种和外币（includeForeign=true 时外币单独聚合）
- `groupByDirection($journals, $method, $direction)` — 按「source/destination 账户 + 币种」聚合

---

## 四、第三阶段：图表数据装配

### 4.1 核心组件

| 层级 | 组件 | 职责 |
|------|------|------|
| 控制器层 | Chart\ReportController 等 | 接收请求，协调数据获取与图表生成 |
| 辅助 Trait | ChartGeneration | 通用图表数据组装逻辑 |
| 辅助 Trait | ResolvesJournalAmountAndCurrency | 统一处理金额和币种解析（含主币种换算） |
| 生成器层 | ChartJsGenerator | 输出 Chart.js 兼容的数据结构 |
| 前端 | Chart.js 库 | 实际渲染图表 |

### 4.2 ChartJsGenerator

app/Generator/Chart/Basic/ChartJsGenerator.php

将业务数据装配为 Chart.js 可识别的格式。

#### `multiSet(array $data, array $labels = []): array`
- **位置**：app/Generator/Chart/Basic/ChartJsGenerator.php #L96-L137
- **输入格式**：
  ```
  [
      [
          'label' => '收入',
          'type' => 'bar',
          'backgroundColor' => 'rgba(...)',
          'currency_symbol' => '¥',
          'entries' => ['2024-01' => '1000', '2024-02' => '1500']
      ],
  ]
  ```
- **输出格式**（Chart.js 兼容）：
  ```
  [
      'count' => N,
      'labels' => ['2024-01', '2024-02', ...],   // 从第一个数据集的 entries 键提取
      'datasets' => [
          ['label' => '收入', 'type' => 'bar', 'data' => ['1000', '1500'],
           'backgroundColor' => '...', 'currency_symbol' => '¥'],
      ]
  ]
  ```
- labels 提取规则：若调用方未显式传 labels，则从 `$data[0]['entries']` 的 keys 中取

### 4.3 图表装配流程（以收支图 ReportController::operations() 为例）

app/Http/Controllers/Chart/ReportController.php #L143-L277

```
1. 区间切分（阶段1的轻量化版本）
   → Navigation::preferredCarbonFormat(start, end)
       < 1月 → 'Y-m-d'（按天）
       1月~1年 → 'Y-m'（按月）
       ≥ 1年 → 'Y'（按年）
   → Navigation::preferredRangeFormat(start, end)
       同上对应 '1D' / '1M' / '1Y'，用于 addPeriod 步进
   → 对于年度图（preferredRange='1Y'），额外把 $end 调为 Navigation::endOfPeriod(end, '1Y')

2. 取原始交易（走 GroupCollector，不走 PeriodStatistic 缓存）
   → setRange(start, end) + setXorAccounts(accounts)
   → setTypes：[WITHDRAWAL, DEPOSIT, RECONCILIATION, TRANSFER]
   → getExtractedJournals() 拿展开后的交易数组

3. 第一重聚合：按 [currencyId][period] 二维分组
   foreach (journals as journal):
       period = journal.date.format(format)   // 格式化后的周期标签
       resolveJournalAmountAndCurrency(journal)  // 解析币种+金额（含主币种换算）
       判断 earned 还是 spent：
           - DEPOSIT → earned
           - TRANSFER/RECONCILIATION/OPENING_BALANCE 且目标账户在 accounts 里 → earned
           - 其余（WITHDRAWAL 或转出）→ spent
       data[currencyId][period][earned/spent] = bcadd(...)

4. 第二重聚合 + 空周期补零
   foreach (currencies as currency):
       构建 income 数据集（绿色 bar，label='box_earned_in_currency'）
       构建 expense 数据集（红色 bar，label='box_spent_in_currency'）
       → 用 while ($currentStart <= $currentEnd) 逐周期推进
           key = currentStart.format(format)
           title = currentStart.isoFormat(titleFormat)
           if data[currency] 有 key → 用 bcround 四舍五入到对应小数位
           else → 补 '0'（空周期补零）
           currentStart = Navigation::addPeriod(currentStart, preferredRange)
       → 这样保证即使某月无数据，x 轴也完整，图不会断裂

5. 最终装配
   → ChartJsGenerator::multiSet($chartData)
   → CacheProperties 缓存
   → response()->json() 返回
```

### 4.4 空周期补零的实现细节

app/Http/Controllers/Chart/ReportController.php #L250-L264

补零不是按「已有周期」对齐，而是**按 Navigation 周期步进生成完整的时间轴**：

```php
$currentStart = clone $start;
while ($currentStart <= $currentEnd) {
    $key   = $currentStart->format($format);
    $title = $currentStart->isoFormat($titleFormat);

    // 有数据 → 用 Steam::bcround 保留对应币种小数位
    if (array_key_exists($key, $currency)) {
        $income['entries'][$title]  = Steam::bcround($currency[$key]['earned'] ?? '0', $dp);
        $expense['entries'][$title] = Steam::bcround($currency[$key]['spent'] ?? '0', $dp);
    }
    // 无数据 → 补零
    if (!array_key_exists($key, $currency)) {
        $income['entries'][$title]  = '0';
        $expense['entries'][$title] = '0';
    }
    $currentStart = Navigation::addPeriod($currentStart, $preferredRange);
}
```

关键点：
- `$currentEnd` 对年图表会被扩展为 `endOfPeriod($end, '1Y')`，保证最末周期完整
- 步进使用 `Navigation::addPeriod()`，会自动处理月末/闰年
- entries 的 key 是**显示标题**（isoFormat 本地化格式），value 是字符串数字
- 周期步长和 `$format` 都是由 `preferredRangeFormat` / `preferredCarbonFormat` 根据总跨度自适应

### 4.5 ChartGeneration Trait

app/Support/Http/Controllers/ChartGeneration.php

提供通用图表方法，如 `accountBalanceChart()`（账户余额走势图）：
- 遍历每个账户
- `Steam::finalAccountBalanceInRange()` 拿每天余额
- 若 convertToPrimary=true，用 ExchangeRateConverter 按天把余额转为主币种
- 逐天推进生成 entries（天为粒度，不补零因为每天都有余额）
- 调 `ChartJsGenerator::multiSet()` 输出

---

## 五、三块接力的完整调用链路

### 5.1 月度报表页面（完整链路）

```
ReportController::defaultReport(accounts, start, end)
  │
  ├─ ReportGeneratorFactory::reportGenerator('Standard', start, end)
  │    按跨度选 MonthReportGenerator
  │
  └─ MonthReportGenerator::generate()
       渲染 views/reports/default/month.twig（页面骨架）
         │
         ├─ chart: account-balances-chart
         │   GET chart.account.report → Chart\AccountController
         │       阶段1: preferredCarbonFormat + preferredRangeFormat
         │       阶段2: GroupCollector 拉交易
         │       阶段3: ChartJsGenerator::multiSet()
         │
         ├─ table: accountReport
         │   GET report-data.account.general → Report\AccountController::general()
         │       CacheProperties 查缓存
         │       AccountTasker::getAccountReport() 阶段2聚合
         │       渲染 reports/partials/accounts.twig → 返回 HTML 字符串
         │
         ├─ table: incomeReport
         │   GET report-data.operations.income → Report\OperationsController::income()
         │       AccountTasker::getIncomeReport() 阶段2聚合
         │       渲染 reports/partials/income-expenses.twig → HTML
         │
         ├─ table: expenseReport
         │   GET report-data.operations.expenses → Report\OperationsController::expenses()
         │       AccountTasker::getExpenseReport() 阶段2聚合
         │       渲染 reports/partials/income-expenses.twig → HTML
         │
         ├─ table: incomeVsExpenseReport
         │   GET report-data.operations.operations → Report\OperationsController::operations()
         │       Income + Expense 合并，渲染 reports/partials/operations.twig → HTML
         │
         ├─ 其他区域（budget / category / balance / bill）同理异步加载
         │
         └─ 所有区域都是独立端点，各自有独立缓存，互不阻塞
```

### 5.2 周期概览页面（账户详情页的月统计列表）

```
Account\ShowController
  ↓
PeriodOverview::getAccountPeriodOverview(account, start, end)
  ├─ Navigation::getViewRange()                 [阶段1]
  ├─ Navigation::blockPeriods()                 [阶段1: 切周期块]
  ├─ getPeriodFromBlocks()                      [阶段1: 对齐边界]
  ├─ PeriodStatisticRepository::allInRangeForModel()   [阶段2: 批量查缓存]
  └─ foreach (周期块):
       getSingleModelPeriod()
         └─ foreach (spent/earned/transferred_in/transferred_away):
              getSingleModelPeriodByType()
                ├─ filterStatistics()           [阶段2: 精确匹配缓存]
                ├─ 命中 → 直接用
                └─ 未命中 → periodCollection() 拉原始交易
                     ├─ filterTransactionsByType()
                     ├─ groupByCurrency()       [含 convertToPrimary 换算]
                     └─ saveGroupedAsStatistics()  [写回 period_statistics]
  ↓
视图渲染周期列表（不是异步，是后端直接渲染）
```

### 5.3 报表图表（收支柱状图）

```
Chart\ReportController::operations(accounts, start, end)
  ├─ Navigation::preferredCarbonFormat()        [阶段1: 定格式]
  ├─ Navigation::preferredRangeFormat()         [阶段1: 定步进]
  ├─ GroupCollector::getExtractedJournals()     [阶段2: 取原始交易]
  ├─ 按 [currencyId][period] 二维聚合            [阶段2]
  ├─ while + Navigation::addPeriod() 补零        [阶段2/3 之间]
  └─ ChartJsGenerator::multiSet()               [阶段3]
  ↓
JSON → 前端 Chart.js 渲染
```

---

## 六、周期统计缓存失效机制详解

### 6.1 触发源：三类事件分别挂载三个监听器

交易组（TransactionGroup）的新增、更新、删除都会触发周期统计缓存失效。三个监听器都通过 `SupportsGroupProcessingTrait` 复用同一套失效逻辑。

| 事件类型 | 事件类 | 监听器类 | 调用 removePeriodStatistics 的时机 |
|---------|--------|----------|----------------------------------|
| 新增 | CreatedSingleTransactionGroup | ProcessesNewTransactionGroup | 在规则引擎处理之后、标记完成之前 |
| 更新 | UpdatedSingleTransactionGroup | ProcessesUpdatedTransactionGroup | 在 unifyAccounts 和规则处理之后 |
| 删除 | DestroyedSingleTransactionGroup | ProcessesDestroyedTransactionGroup | 在 webhook 触发之后、运行余额重算之前 |

三个监听器都使用 `SupportsGroupProcessingTrait::removePeriodStatistics()` 做实际的缓存删除。

**监听器的位置**：app/Listeners/Model/TransactionGroup/ 目录下
- ProcessesNewTransactionGroup.php
- ProcessesUpdatedTransactionGroup.php
- ProcessesDestroyedTransactionGroup.php
- SupportsGroupProcessingTrait.php（共用逻辑）

### 6.2 失效入口：removePeriodStatistics()

**位置**：app/Listeners/Model/TransactionGroup/SupportsGroupProcessingTrait.php #L93-L119

每次失效会一次性删除**四个模型维度**加**四个前缀维度**的统计：

```php
$dates = $this->collectDatesFromJournals($objects->transactionJournals);

// 模型维度（4类）：
$repository->deleteStatisticsForType(Account::class,  $objects->accounts,     $dates);
$repository->deleteStatisticsForType(Budget::class,   $objects->budgets,      $dates);
$repository->deleteStatisticsForType(Category::class, $objects->categories,   $dates);
$repository->deleteStatisticsForType(Tag::class,      $objects->tags,         $dates);

// 前缀维度（4类）：
$repository->deleteStatisticsForPrefix('all_',        $dates);   // 全局交易统计
$repository->deleteStatisticsForPrefix('no_budget',   $dates);   // 无预算统计
$repository->deleteStatisticsForPrefix('no_category', $dates);   // 无分类统计
$repository->deleteStatisticsForPrefix('no_tag',      $dates);   // 无标签统计
```

也就是说，一笔交易变更后，所有相关维度的周期统计都会被清掉：
- 涉及的账户、分类、预算、标签各自的周期统计
- 全局（all_）、无分类、无预算、无标签的周期统计

### 6.3 日期集合的收集：collectDatesFromJournals()

**位置**：app/Listeners/Model/TransactionGroup/SupportsGroupProcessingTrait.php #L121-L129

```php
private function collectDatesFromJournals(Collection $journals): Collection
{
    $collection = $journals->pluck('date');
    if (0 === count($collection)) {
        $collection->push(now(config('app.timezone')));
    }
    return $collection;
}
```

- 直接从 `$journals` 集合里每个 journal 的 `date` 属性提取（不是查数据库，是内存中已有对象的属性）
- 如果 journals 为空（理论上不该发生），回退为当前时间

**⚠️ 更新时 dates 里只有新日期，旧日期不会进入周期统计删除条件**

完整更新流程（以 app/Api/V1/Controllers/Models/Transaction/UpdateController.php #L80-L83 为例）：

```php
// 步骤1：先从数据库收集旧的 group+journals
$objects = TransactionGroupEventObjects::collectFromTransactionGroup($transactionGroup);

// 步骤2：执行更新（修改数据库和内存中的 journal 对象）
$transactionGroup = $this->groupRepository->update($transactionGroup, $data);

// 步骤3：再次 append 同一个 group（refresh 后的新对象）
$objects->appendFromTransactionGroup($transactionGroup);
```

看起来步骤1和步骤3各收集了一次，应该有新旧两套。但实际有两个机制使旧日期丢失：

#### 机制 A：PHP 对象引用导致第一个 journal 实例的 date 已经被改成新值

步骤1收集 journal 时，push 到 `$this->transactionJournals` 的是**对象引用**。
步骤2的 update 过程中，JournalUpdateService（app/Services/Internal/Update/JournalUpdateService.php #L669）直接修改了内存中同一个 journal 对象实例的 date：

```php
// JournalUpdateService::updateField('date')
$this->transactionJournal->{$fieldName} = $value;  // 直接修改原对象属性
```

所以虽然 unique 之后保留了步骤1 push 进去的「第一个」journal，但它的 date 属性已经在步骤2被改成新值了。

#### 机制 B：unique('id') 保留第一个，丢弃 refresh 后的新实例

步骤3 append 时再次 `push` 同一个 journal id 的新实例（refresh 后重新从 DB 加载的对象）。但 `appendFromTransactionGroup` 末尾调用了 `$this->transactionJournals->unique('id')`，Laravel Collection 的 unique() 行为是保留**第一次出现**的元素，丢弃后续同 id 的元素。

所以 refresh 后的新 journal 实例（date 也是新值，和第一个一致）被丢弃。

**最终效果**：`collectDatesFromJournals()` 拿到的 dates 里，每个 journal 只出现一次，且 date 是**更新后的新值**。旧日期不会进入删除条件。

---

### 6.4 对象集合的收集与去重机制

**位置**：app/Events/Model/TransactionGroup/TransactionGroupEventObjects.php

#### appendFromTransactionGroup() 的完整流程

```php
public function appendFromTransactionGroup(TransactionGroup $transactionGroup): void
{
    $this->transactionGroups->push($transactionGroup);
    foreach ($transactionGroup->transactionJournals as $journal) {
        $this->transactionJournals->push($journal);           // 1. 先 push
        $this->budgets    = $this->budgets->merge($journal->budgets);
        $this->categories = $this->categories->merge($journal->categories);
        $this->tags       = $this->tags->merge($journal->tags);
        foreach ($journal->transactions as $transaction) {
            $this->accounts->push($transaction->account);
        }
    }
    // 2. 末尾统一 unique('id') 去重：保留第一个出现的元素，丢弃后续同 id 的
    $this->transactionGroups   = $this->transactionGroups->unique('id');
    $this->transactionJournals = $this->transactionJournals->unique('id');
    $this->budgets             = $this->budgets->unique('id');
    $this->categories          = $this->categories->unique('id');
    $this->tags                = $this->tags->unique('id');
    $this->accounts            = $this->accounts->unique('id');
}
```

收集规则：
- accounts：遍历所有 transaction → account（每个拆分的两方账户）
- budgets：journal->budgets（一对多）
- categories：journal->categories（一对多）
- tags：journal->tags（多对多）
- transactionJournals：组内所有 journals
- transactionGroups：组本身

#### 去重对更新场景的影响

对于 accounts/budgets/categories/tags 这些「基本不会因日期修改而变化」的对象，unique 前后没区别。
对于 transactionJournals，因上面的对象引用机制，unique 后保留的是「第一个 push 但 date 已被改成新值」的那个实例。

---

### 6.5 为什么运行余额用了 _internal_previous_date 但周期统计删除没用

**两者的区别**：

| 维度 | 周期统计缓存删除 | 运行余额重算 |
|------|----------------|------------|
| 方法 | `collectDatesFromJournals()` | `recalculateRunningBalance()` + `getFromInternalDate()` |
| 日期来源 | `$journals->pluck('date')`（内存对象的属性） | `pluck('date')` + `transaction_journal_meta` 表的 `_internal_previous_date` |
| 位置 | SupportsGroupProcessingTrait #L106 | SupportsGroupProcessingTrait #L68-L91 + #L157-L172 |

#### _internal_previous_date 是什么时候写入的

app/Services/Internal/Update/JournalUpdateService.php #L630-L657 中的 `updateField('date')`：

```php
$res = $value->gt($this->transactionJournal->date);  // 新日期 > 旧日期？
$set = ['journal' => $this->transactionJournal, 'name' => '_internal_previous_date', 'data' => null];
if ($res) {
    // 只在"把日期往后改"时，才把旧日期存到 _internal_previous_date 元字段
    $set['data'] = clone $this->transactionJournal->date;
}
// 如果是"往回改日期"，不存这个字段（data=null）
$factory->updateOrCreate($set);
```

**也就是说，_internal_previous_date 只在「新日期 > 旧日期」时才有值。**

#### 运行余额重算怎么用的

app/Listeners/Model/TransactionGroup/SupportsGroupProcessingTrait.php #L80-L84：

```php
$earliest         = $objects->transactionJournals->pluck('date')->sort()->first();
$fromInternalDate = $this->getFromInternalDate($objects->transactionJournals->pluck('id')->toArray());
$earliest         = $fromInternalDate->lt($earliest) ? $fromInternalDate : $earliest;
```

`getFromInternalDate()` 额外查询 `transaction_journal_meta` 表拿到所有 `_internal_previous_date`，取其中最早的那个。运行余额重算需要从「最早受影响的日期」开始重算，所以必须把旧日期也考虑进来。

#### 为什么周期统计删除没这样做

这是代码的设计遗漏（或者说 bug）：
- `removePeriodStatistics()` 的 `collectDatesFromJournals()` 直接从内存对象 pluck date，只拿到新日期
- 没有查询 `_internal_previous_date` 元字段
- 也没有额外的逻辑尝试获取旧日期

**后果**：当交易从 1月改到 2月时，1月的周期统计不会被失效删除（因为删除条件里只有 2月15日，不落在 1月的 start..end 区间里），只有 2月及更大粒度的周期会被删除。这和 AND 叠加效应一起，使细粒度周期统计的数据一致性问题更加严重。

### 6.6 多日期删除条件的 SQL 逻辑：AND 叠加

**位置**：app/Repositories/PeriodStatistic/PeriodStatisticRepository.php #L121-L140

```php
$count = PeriodStatistic::where('primary_statable_type', $class)
    ->whereIn('primary_statable_id', $objects->pluck('id')->toArray())
    ->where(function (Builder $q) use ($dates): void {
        foreach ($dates as $date) {
            $q->where(function (Builder $q1) use ($date): void {
                $q1->where('start', '<=', $date)->where('end', '>=', $date);
            });
        }
    })
    ->delete()
;
```

生成的 SQL 等价于：
```sql
WHERE primary_statable_type = ?
  AND primary_statable_id IN (?, ?, ...)
  AND (
    (start <= date1 AND end >= date1)
    AND (start <= date2 AND end >= date2)
    AND ...
  )
```

**关键事实**：`foreach` 里用 `$q->where()` 连续叠加，Laravel 同一级多个 where 默认是 **AND** 连接。即统计区间必须**同时包含所有日期**才会被删。

结合前节结论「更新场景 dates 里只有更新后日期（单日期）」：
- 更新单条交易：dates 只有一个日期，AND 不生效，退化为单日期条件
- 批量导入多笔日期不同的交易：多日期 AND 叠加才真正触发

前缀维度 `deleteStatisticsForPrefix()` 逻辑一致，只是把模型条件换成 `type LIKE 'prefix%'`。

单日期版本 `deleteStatisticsForModel()`：
```php
$model->primaryPeriodStatistics()->where('start', '<=', $date)->where('end', '>=', $date)->delete();
```
删所有跨该日期的周期。

---

### 6.7 四种场景的真实失效效应推演

**前提（贯穿所有推演）**：缓存删除条件只看**更新后日期**。新增和同月修改无此问题；跨周期修改时，原周期的统计永远不被失效删除。

#### 场景 1：新增一笔交易，日期 1月15日
- **dates = [1月15日]**
- 删除条件：`start <= 1/15 AND end >= 1/15`
- 1月（1/1~1/31）：✓ 删除
- 1季度（1/1~3/31）：✓ 删除
- 1年度：✓ 删除
- **结论**：所有跨该日期的周期都被正确失效，下次查看触发懒重算得到正确值。

#### 场景 2：更新一笔交易的日期（1月15日 → 2月15日，跨月）
- **dates = [2月15日]（只有更新后日期，原日期 1月15日不进入条件）**
- 删除条件：`start <= 2/15 AND end >= 2/15`

| 周期 | 区间 | 条件判定 | 是否删除 | 真实影响 |
|------|------|---------|---------|---------|
| 1月份 | 1/1 ~ 1/31 | end=1/31 < 2/15 → 假 | ✗ 不删 | 脏数据：还留着这笔已移走的交易 |
| 2月份 | 2/1 ~ 2/29 | start<=2/15 且 end>=2/15 → 真 | ✓ 删除 | 正确：下次懒重算重新包含这笔 |
| 1季度 | 1/1 ~ 3/31 | 条件满足 → 真 | ✓ 删除 | 正确：重算后不含 1月，含 2月 |
| 1年度 | 1/1 ~ 12/31 | 条件满足 → 真 | ✓ 删除 | 正确：重算后分布对 |

**结论**：原月份（1月）的统计永远不被失效，长期残留脏数据。目标月份（2月）及更大粒度周期被正确删除，懒重算后准确。

#### 场景 3：更新一笔交易的日期但不跨月（1月15日 → 1月25日）
- **dates = [1月25日]（只有更新后日期）**
- 删除条件：`start <= 1/25 AND end >= 1/25`

所有涉及的周期（1月、1季度、1年度）都满足这个条件 → 全部被删 ✓

**结论**：同月修改没有问题。因为新旧日期落在同一个周期，只要这个周期被失效删除了，旧日期那边的脏数据自然也一起清掉了。

#### 场景 4：批量导入 10 笔交易，日期分散在 1~10 月各一笔
- **dates = [1月1日, 2月1日, ..., 10月1日]（多日期，AND 叠加生效）**
- 删除条件：一个周期要同时横跨这 10 个日期才会被删

| 周期 | 条件判定 | 是否删除 |
|------|---------|---------|
| 任何单个月 | 不可能覆盖 10 个月的日期 | ✗ 不删 |
| 任何单个季度 | 最多覆盖 3 个月 | ✗ 不删 |
| 1年度 | 同时包含全部 10 个日期 | ✓ 删除 |

**结论**：只有年度粒度被正确失效，月/季粒度保留旧值。

---

### 6.8 跨周期修改对懒重算的完整影响

#### 各粒度一致性总表

| 修改类型 | 周期粒度 | 是否被删 | 下次查看是否懒重算 | 数据正确性 |
|---------|---------|---------|-------------------|-----------|
| 新增 | 日/周/月/季/年 | ✓ | ✓ | 准确 |
| 同月修改 | 日/周/月/季/年 | ✓ | ✓ | 准确 |
| **跨月修改（1月→2月）** | **1月份（原周期）** | **✗ 永远不删** | **✗** | **不准确（多了这笔已移走的交易）** |
| 跨月修改（1月→2月） | 2月份（新周期） | ✓ | ✓ | 准确 |
| 跨月修改（1月→2月） | 季度/年度 | ✓ | ✓ | 准确（重算后） |
| 批量多周期导入 | 月/季 | ✗ | ✗ | 不准确 |
| 批量多周期导入 | 年 | ✓ | ✓ | 准确 |

#### 脏数据传导的完整路径

1. 用户把一笔 1月15日的交易改成 2月15日
2. 内存中同一个 journal 对象实例的 date 被改成了 2月15日
3. `appendFromTransactionGroup()` 用 unique('id') 去重，保留的是「第一个 push 但 date 已是新值」的对象
4. `collectDatesFromJournals()` 只拿到 `[2月15日]`，旧日期 1月15日彻底丢失
5. 删除 SQL 的 WHERE 条件只覆盖 2月15日，1月统计不被删
6. 用户查看账户详情页月度周期列表：
   - 2月统计不存在 → 懒重算 → 正确包含这笔 ✓
   - 1月统计存在 → `filterStatistics()` 精确匹配命中 → 直接返回脏数据 ✗
7. 用户看到：1月支出偏高（还挂着已移走的那笔），2月支出正确

#### 脏数据被纠正的唯一机会

原月份（1月）的脏数据不会自动消失，除非发生以下事件之一：
- 同一账户在 1月又新增/删除了另一笔交易 → 新交易的日期触发 1月统计被删除 → 懒重算时得到正确值
- 手动执行重建统计的命令或直接清 period_statistics 表
- 数据库里 1月对应的 period_statistics 记录被人工删除

否则，1月的脏数据会长期留存，即使季度/年度统计已刷新。

#### 为什么运行余额重算能避开这个问题

运行余额走独立路径（SupportsGroupProcessingTrait.php #L68-L91）：
- `recalculateRunningBalance()` 先取 `$journals->pluck('date')` 的最早值
- 再调用 `getFromInternalDate()` 额外查询 `transaction_journal_meta` 表拿 `_internal_previous_date`
- 两者取更早的那个作为重算起点

但 `_internal_previous_date` 只在「新日期 > 旧日期」（往后改）时写入。如果是「把日期往回改」（比如 2月→1月），这个元字段不存旧值，运行余额同样可能从错误的日期开始重算。

### 6.9 两层缓存的独立性

周期统计缓存失效**只影响 period_statistics 数据库表**，不影响 CacheProperties 应用级缓存。两者是独立的：

1. **交易变更 → Listener → 删除 period_statistics 记录**
   - 只删数据库里的预计算统计
   - 下次 PeriodOverview 查询时，因查不到记录而触发懒重算

2. **CacheProperties 缓存（报表 HTML / 图表 JSON）**
   - 有自己的 key 和过期时间
   - 交易变更**不会**主动清这层缓存
   - 依赖缓存过期自动失效，或者依赖页面参数变化导致 key 变化

所以报表页面（Report\*Controller 的 HTML 输出）的缓存不会因为交易变更而立即刷新，只有等缓存过期后才会重新聚合。

---

## 七、关键设计要点汇总

### 7.1 两层缓存策略

1. **PeriodStatistic 表**（数据库级）
   - 按模型/类型/币种/周期存预计算统计
   - 交易变更时 Listener 删除「跨新日期」的周期统计
     - 新增/同月修改：删除正确，所有涉及周期都会被失效
     - **跨月修改：原周期的统计永远不会被删除**（因旧日期丢失+AND叠加），残留脏数据
     - 多日期批量导入：只有同时横跨所有日期的最大粒度周期（通常是年）被删
   - 下次查询懒重算并写回
   - 空结果也存一条（count=0, amount=0），避免反复重查

2. **CacheProperties**（应用级）
   - 报表 HTML、图表 JSON 用 Laravel Cache 存
   - key = start + end + 标识 + accountIds + convertToPrimary
   - 命中率高，大幅提升页面加载速度
   - 交易变更不主动清这层缓存，依赖过期

两类缓存独立，互不影响。PeriodStatistic 就算删对了，CacheProperties 也要等过期才刷新。

### 7.2 周期边界处理的一致性

- **生成周期块**：Navigation::startOfPeriod/endOfPeriod 统一规整，秒和毫秒都清零
- **存储**：start/end 原样入库，同时存 start_tz/end_tz
- **读取还原**：SeparateTimezoneCaster 用 start_tz 解析后转应用时区，保证比较一致性
- **查询匹配**：批量用 `start >= X AND end <= Y`，单条用 `isSameSecond()` 精确比对
- **失效删除**：
  - 单日期：`start <= date AND end >= date`（删跨该日期的所有周期）
  - 多日期：每个日期的单日期条件再用 AND 叠加（只有同时跨所有日期的周期才被删）
  - **⚠️ 但跨月更新时 dates 里只有新日期，旧日期不会进入条件**

### 7.3 空周期补零

- 不是从数据反推有哪些周期，而是**按 Navigation::addPeriod() 步进构造完整时间轴**
- 步长由 start/end 总跨度自适应：<1月按天，1月~1年按月，≥1年按年
- 年度图最末周期还会扩展为完整 endOfPeriod(end, '1Y')，避免最后一个柱被截断
- 确保 Chart.js 的 x 轴标签完整连续，不会因为某月无数据而缺柱

### 7.4 多币种处理的两条等价路径

路径 A：PeriodOverview 系列（周期概览）→ `groupByCurrency()`
路径 B：图表控制器系列 → `resolveJournalAmountAndCurrency()`

两条路径逻辑完全一致：
```
if convertToPrimary and 交易币种 ≠ 主币种:
    if 外币恰好是主币种 → 用 foreign_amount + 外币元信息
    else → 用 pc_amount（预计算的主币种换算额）+ 主币种元信息
else → 用原 amount + 原币种元信息
```

- pc_amount 是交易落库时就预先算好的主币种金额，避免实时查汇率
- 所有金额计算用 bc* 系列函数，字符串高精度，避免浮点误差
- 聚合后每个币种独立一条记录，图表中不同币种可能作为独立 dataset 或独立 y 轴

### 7.5 月度报表的前后端分工

- 后端 ReportController 只输出页面骨架（含各区域的 URL）
- 每个表格区域对应一个独立的异步端点（Report\*Controller）
- 每个端点走「CacheProperties → 聚合 → 渲染 Twig 片段 → 返回 HTML」
- 图表区域对应 Chart\*Controller，返回 JSON
- 所有区域完全并行加载，各区域独立缓存
