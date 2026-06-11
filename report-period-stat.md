# 月度报表与周期统计聚合流程详解

本文档梳理 Firefly III 中月度报表与周期统计从原始交易数据到最终图表展示的完整数据流转过程，分为**统计区间切分**、**聚合查询**、**图表数据装配**三大阶段。

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

**核心类**：[Navigation.php](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Support/Navigation.php)

### 2.2 关键方法

#### `getViewRange(bool $correct): string`
- **位置**：[Navigation.php#L423-L441](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Support/Navigation.php#L423-L441)
- **作用**：从用户偏好中读取视图范围（viewRange），并对动态范围（如 last7、MTD、YTD）进行标准化转换。
- **输入输出**：
  - 输入：`true`（是否修正动态范围）
  - 输出：标准化范围字符串（`1D` / `1W` / `1M` / `3M` / `6M` / `1Y`）
- **转换规则**：
  - `last7` → `1W`
  - `last30` / `MTD` → `1M`
  - `last90` / `QTD` → `3M`
  - `last365` / `YTD` → `1Y`

#### `blockPeriods(Carbon $start, Carbon $end, string $range): array`
- **位置**：[Navigation.php#L89-L132](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Support/Navigation.php#L89-L132)
- **作用**：将给定的起止日期区间，按指定周期粒度切分成若干个完整的周期块。
- **算法逻辑**：
  1. 从 `$end` 日期开始向前回溯
  2. 先按 `$range` 粒度生成最多 13 个周期块（保证足够细粒度）
  3. 如果范围还不够，继续按 `1Y` 粒度向后补充（最多 20 个年周期）
  4. 每个周期块包含 `start`、`end`、`period` 三个字段
- **输出示例**（`range=1M`）：
  ```php
  [
      ['start' => '2024-01-01', 'end' => '2024-01-31', 'period' => '1M'],
      ['start' => '2024-02-01', 'end' => '2024-02-29', 'period' => '1M'],
      // ...
  ]
  ```

#### `startOfPeriod(Carbon $theDate, string $repeatFreq): Carbon`
- **位置**：[Navigation.php#L651-L724](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Support/Navigation.php#L651-L724)
- **作用**：计算给定日期所在周期的起始时间点。
- **支持周期**：日、周、月、季、半年、年，以及 MTD/QTD/YTD 等相对周期。

#### `endOfPeriod(Carbon $end, string $repeatFreq): Carbon`
- **位置**：[Navigation.php#L185-L384](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Support/Navigation.php#L185-L384)
- **作用**：计算给定日期所在周期的结束时间点。
- **注意**：周/月/季/年等周期的结束时间会 `subDay()` 再 `endOfDay()`，保证区间为闭区间。

### 2.3 切分流程调用链

以账户周期概览为例：
```
getAccountPeriodOverview()
    → Navigation::getViewRange(true)         // 获取标准化视图范围
    → Navigation::blockPeriods(start, end, range)  // 切分周期块
    → getPeriodFromBlocks()                  // 扩展起止日期以对齐完整周期
    → 遍历每个周期块，逐一聚合
```

调用入口：[PeriodOverview.php#L92-L114](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Support/Http/Controllers/PeriodOverview.php#L92-L114)

---

## 三、第二阶段：聚合查询

聚合查询采用**懒加载 + 缓存**的两层架构：先查预计算的 `period_statistics` 表，没有命中则从原始交易实时计算并写回缓存。

### 3.1 数据模型

**PeriodStatistic 模型**：[PeriodStatistic.php](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Models/PeriodStatistic.php)

| 字段 | 说明 |
|------|------|
| `start` / `end` | 统计周期的起止时间 |
| `type` | 统计类型（如 `spent`、`earned`、`transferred_in` 等） |
| `amount` | 聚合金额 |
| `count` | 交易笔数 |
| `transaction_currency_id` | 币种 ID |
| `primary_statable_type` / `primary_statable_id` | 关联对象（多态：Account/Category/Tag 等） |

对于没有具体关联对象的统计（如"无分类"、"全部交易"），使用前缀类型：
- `no_category_spent` / `no_category_earned` — 无分类的支出/收入
- `all_withdrawal` / `all_deposit` — 全部取款/存款

### 3.2 核心组件

#### PeriodStatisticRepository
**文件**：[PeriodStatisticRepository.php](file:///d:/fz/0508-2\solo-dogfeeding/code/128-firefly-iii/app/Repositories/PeriodStatistic/PeriodStatisticRepository.php)

关键方法：
- `allInRangeForModel($model, $start, $end)` — 按模型+时间范围查询统计数据
- `allInRangeForPrefix($prefix, $start, $end)` — 按前缀类型+时间范围查询
- `saveStatistic(...)` — 保存模型关联的统计
- `savePrefixedStatistic(...)` — 保存前缀类型的统计

#### PeriodOverview Trait
**文件**：[PeriodOverview.php](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Support/Http/Controllers/PeriodOverview.php)

这是周期统计的核心业务逻辑 Trait，被多个控制器复用。

### 3.3 聚合流程（以单模型单周期为例）

核心方法：`getSingleModelPeriodByType()`
[PeriodOverview.php#L423-L511](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Support/Http/Controllers/PeriodOverview.php#L423-L511)

```
1. 先从 $this->statistics 中过滤匹配（start/end/type 完全匹配）
   ↓ 命中缓存
2a. 直接装配为按币种分组的数组返回
   ↓ 未命中缓存（懒加载触发）
2b. 从对应 Repository 的 periodCollection() 获取原始交易列表
2c. filterTransactionsByType() 按交易类型过滤（spent→WITHDRAWAL，earned→DEPOSIT）
2d. groupByCurrency() 按币种聚合金额和笔数
2e. saveGroupedAsStatistics() 将结果写回 period_statistics 表
2f. 返回聚合结果
```

### 3.4 按币种聚合逻辑

**方法**：`groupByCurrency(array $journals): array`
[PeriodOverview.php#L614-L664](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Support/Http/Controllers/PeriodOverview.php#L614-L664)

聚合规则：
- 以 `currency_id` 为键
- 每个币种包含：`amount`（累计金额）、`count`（交易笔数）、币种元信息
- 支持"转换为主币种"模式（`convertToPrimary`）：
  - 如果交易币种不是主币种，使用 `pc_amount`（主币种换算金额）
  - 如果外币恰好是主币种，使用 `foreign_amount`
- 顶层还有一个 `count` 字段记录总笔数

### 3.5 报表聚合（AccountTasker）

除了周期概览使用的 PeriodStatistic 缓存，报表模块还有一套独立的聚合逻辑：

**文件**：[AccountTasker.php](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Repositories/Account/AccountTasker.php)

- `getExpenseReport()` — 支出报告：按对方账户分组
  - 收集 `WITHDRAWAL` + `TRANSFER` 类型的出账交易
  - 排除内部账户互转（`excludeDestinationAccounts`）
  - 按目标账户+币种分组聚合

- `getIncomeReport()` — 收入报告：按对方账户分组
  - 收集 `DEPOSIT` + `TRANSFER` 类型的入账交易
  - 排除内部账户互转

- `getAccountReport()` — 账户报告：期初/期末余额

### 3.6 TransactionSummarizer

**文件**：[TransactionSummarizer.php](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Support/Report/Summarizer/TransactionSummarizer.php)

通用的交易聚合工具类，提供两种聚合维度：
- `groupByCurrencyId()` — 按币种聚合（支持正/负金额处理）
- `groupByDirection()` — 按账户方向（source/destination）+ 币种聚合

---

## 四、第三阶段：图表数据装配

### 4.1 核心组件

| 层级 | 组件 | 职责 |
|------|------|------|
| 控制器层 | `Chart\ReportController` 等 | 接收请求，协调数据获取与图表生成 |
| 辅助 Trait | `ChartGeneration` | 通用图表数据组装逻辑 |
| 生成器层 | `ChartJsGenerator` | 输出 Chart.js 兼容的数据结构 |
| 前端 | Chart.js 库 | 实际渲染图表 |

### 4.2 ChartJsGenerator

**文件**：[ChartJsGenerator.php](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Generator/Chart/Basic/ChartJsGenerator.php)

负责将业务数据装配为 Chart.js 可识别的格式。

#### `multiSet(array $data, array $labels = []): array`
- **位置**：[ChartJsGenerator.php#L96-L137](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Generator/Chart/Basic/ChartJsGenerator.php#L96-L137)
- **输入格式**：
  ```php
  [
      [
          'label' => '收入',
          'type' => 'bar',           // 可选：bar/line
          'backgroundColor' => '...',// 可选
          'currency_symbol' => '¥',  // 可选
          'entries' => [
              '1月' => '1000',
              '2月' => '1500',
          ]
      ],
      // 更多数据集...
  ]
  ```
- **输出格式**（Chart.js 兼容）：
  ```php
  [
      'count' => 2,
      'labels' => ['1月', '2月', ...],     // 从第一个数据集提取
      'datasets' => [
          ['label' => '收入', 'type' => 'bar', 'data' => ['1000', '1500']],
          // ...
      ]
  ]
  ```

其他方法：
- `singleSet()` — 单数据集图表
- `pieChart()` — 饼图
- `multiCurrencyPieChart()` — 多币种饼图

### 4.3 图表装配流程（以收支图为例）

以 `ReportController::operations()` 方法为例：
[ReportController.php#L143-L277](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Http/Controllers/Chart/ReportController.php#L143-L277)

```
1. 确定时间粒度
   → Navigation::preferredCarbonFormat()   // 日期格式化
   → Navigation::preferredRangeFormat()    // 周期步进（1D/1M/1Y）

2. 获取原始交易
   → GroupCollectorInterface 收集指定范围内的所有交易
   → 包含 WITHDRAWAL / DEPOSIT / RECONCILIATION / TRANSFER

3. 第一重聚合：按 币种 + 周期 分组
   foreach (journals as journal):
       period = date.format(format)
       按类型判断是 spent 还是 earned
       累加到 data[currencyId][period][spent/earned]

4. 第二重聚合：装配为图表数据集
   foreach (currencies as currency):
       构建 income 数据集（绿色柱状）
       构建 expense 数据集（红色柱状）
       → Navigation::addPeriod() 逐周期推进，补全空白周期（值为0）
       保证每个周期都有对应数据点

5. 调用生成器输出
   → ChartJsGenerator::multiSet($chartData)
   → 缓存结果
   → 返回 JSON 响应
```

### 4.4 ChartGeneration Trait

**文件**：[ChartGeneration.php](file:///d:/fz/0508-2/solo-dogfeeding/code/128-firefly-iii/app/Support/Http/Controllers/ChartGeneration.php)

提供通用图表方法，如 `accountBalanceChart()` —— 账户余额走势图：
- 遍历每个账户
- 按天获取余额（使用 `Steam::finalAccountBalanceInRange()`）
- 支持主币种转换
- 调用 `ChartJsGenerator::multiSet()` 输出

---

## 五、三块接力的完整调用链路

### 5.1 周期概览页面（账户详情页的月统计列表）

```
AccountController::show()
  ↓
PeriodOverview::getAccountPeriodOverview()
  ├─ Navigation::getViewRange()           [阶段1: 确定周期粒度]
  ├─ Navigation::blockPeriods()           [阶段1: 切分周期块]
  ├─ getPeriodFromBlocks()                [阶段1: 对齐起止日期]
  ├─ PeriodStatisticRepository::allInRangeForModel()  [阶段2: 查缓存]
  └─ foreach (周期块):
       getSingleModelPeriod()
         ├─ filterStatistics()            [阶段2: 缓存命中过滤]
         ├─ 命中 → 直接使用
         └─ 未命中 → periodCollection()   [阶段2: 实时聚合]
              ├─ filterTransactionsByType()
              ├─ groupByCurrency()
              └─ saveGroupedAsStatistics()  [阶段2: 写回缓存]
  ↓
视图渲染周期列表
```

### 5.2 报表页面的图表

```
ReportController（Chart）::operations()
  ├─ Navigation::preferredCarbonFormat()  [阶段1: 确定日期格式]
  ├─ Navigation::preferredRangeFormat()   [阶段1: 确定周期步进]
  ├─ GroupCollector::getExtractedJournals()  [阶段2: 取原始交易]
  ├─ 按币种+周期双重聚合                    [阶段2: 业务聚合]
  ├─ 补全空白周期（值为0）
  └─ ChartJsGenerator::multiSet()         [阶段3: 装配图表格式]
  ↓
JSON → 前端 Chart.js 渲染
```

### 5.3 月度报表页面

```
Report\OperationsController::operations()
  ├─ AccountTasker::getIncomeReport()     [阶段2: 收入聚合]
  ├─ AccountTasker::getExpenseReport()    [阶段2: 支出聚合]
  └─ 视图渲染报表表格
```

---

## 六、关键设计要点

### 6.1 两层缓存策略

1. **PeriodStatistic 表**（数据库级缓存）
   - 预计算的周期统计数据，按模型/类型/币种/周期存储
   - 交易变更时通过 Listener 失效相关统计（删除对应记录）
   - 下次查询时自动重新计算并写回

2. **CacheProperties**（应用级缓存）
   - 图表/报表结果使用 Laravel Cache 缓存
   - 以日期范围、账户 ID 等为键
   - 缓存命中率高，大幅提升页面加载速度

### 6.2 周期对齐原则

- 所有统计周期都是**完整周期**（完整的月/季/年）
- 用户给定的起止日期可能被扩展以对齐周期边界
- 空白周期用零值填充，保证图表连续性

### 6.3 多币种处理

- 每笔交易按原币种聚合
- 每个币种独立统计，不直接混算
- 支持"转换为主币种"视图（`convertToPrimary`），使用汇率换算后的 `pc_amount`
- 图表中不同币种可能分为独立的数据集或坐标轴

### 6.4 多态关联设计

`period_statistics` 表使用多态关联（`primary_statable_type` + `primary_statable_id`），同一套统计模型支持：
- 账户维度统计（Account）
- 分类维度统计（Category）
- 标签维度统计（Tag）
- 预算维度统计（Budget）

对于无维度的全局统计，使用前缀类型（如 `all_`、`no_category_`）。
