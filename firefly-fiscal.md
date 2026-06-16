# Firefly III 财年辅助类与期间选择配合机制分析

## 一、架构总览

Firefly III 的财年处理涉及多个层级的协作，从用户配置到实际计算，形成了一条清晰的调用链：

```
用户配置 (preferences/index.twig)
    ↓
偏好存储 (PreferencesController.php)
    ↓
服务绑定 (FireflyServiceProvider.php)
    ↓
财年辅助类 (FiscalHelper.php)
    ↓
导航/期间类 (Navigation.php)
    ↓
页面控制器 (ReportController.php / Budget/IndexController.php)
    ↓
视图展示 (reports/index.twig / budgets/index.twig)
```

---

## 二、财年起止配置的读取点

### 2.1 配置项定义

系统使用两个关键的用户偏好设置来控制财年行为：

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `customFiscalYear` | boolean | `false` | 是否启用自定义财年 |
| `fiscalYearStart` | string (格式: `m-d`) | `'01-01'` | 财年开始日期 |

### 2.2 配置读取位置

#### 2.2.1 FiscalHelper 构造函数
文件：[FiscalHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php#L41-L44)

```php
public function __construct()
{
    $this->useCustomFiscalYear = (bool) Preferences::get('customFiscalYear', false)->data;
}
```

**关键点**：
- 在对象实例化时一次性读取 `customFiscalYear` 配置
- 该值被缓存为实例属性 `$useCustomFiscalYear`
- 这意味着如果用户在请求过程中更改了配置，已实例化的对象不会感知到变化

#### 2.2.2 startOfFiscalYear 方法中动态读取
文件：[FiscalHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php#L72-L97)

```php
public function startOfFiscalYear(Carbon $date): Carbon
{
    $startDate = clone $date;
    if ($this->useCustomFiscalYear) {
        $prefStartStr = Preferences::get('fiscalYearStart', '01-01')->data;
        // ... 解析并计算
    }
    // ...
}
```

**关键点**：
- `fiscalYearStart` 配置在每次调用 `startOfFiscalYear()` 时动态读取
- 包含类型校验：如果配置是数组（异常情况），则回退到 `'01-01'`
- 解析格式为 `m-d`，例如 `'07-01'` 表示7月1日

#### 2.2.3 配置保存入口
文件：[PreferencesController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Controllers/PreferencesController.php#L296-L303)

```php
$customFiscalYear = 1 === (int) $request->input('customFiscalYear');
Preferences::set('customFiscalYear', $customFiscalYear);

$fiscalYearString = (string) $request->input('fiscalYearStart');
if ('' !== $fiscalYearString) {
    $fiscalYearStart = Carbon::parse($fiscalYearString, config('app.timezone'))->format('m-d');
    Preferences::set('fiscalYearStart', $fiscalYearStart);
}
```

**关键点**：
- 用户输入的日期会被解析并统一格式化为 `m-d`
- 空字符串时不更新 `fiscalYearStart`
- 保存后会清除会话中的 `start`、`end`、`range` 变量（第264-266行），强制重新计算

#### 2.2.4 配置UI展示
文件：[preferences/index.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/resources/views/preferences/index.twig#L110-L112)

```twig
{% set isCustomFiscalYear = customFiscalYear == 1 ? true : false %}
{{ ExpandedForm.checkbox('customFiscalYear','1',isCustomFiscalYear,{ 'label' : 'pref_custom_fiscal_year_label'|_ }) }}
{{ ExpandedForm.date('fiscalYearStart',fiscalYearStart,{ 'label' : 'pref_fiscal_year_start_label'|_ }) }}
```

---

## 三、财年期间计算算法

### 3.1 服务注册
文件：[FireflyServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Providers/FireflyServiceProvider.php#L194)

```php
$this->app->bind(FiscalHelperInterface::class, FiscalHelper::class);
```

通过接口绑定，使得可以通过 `app(FiscalHelperInterface::class)` 获取实例。

### 3.2 startOfFiscalYear 算法

文件：[FiscalHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php#L72-L97)

```
输入日期 → 克隆日期 → 判断是否启用自定义财年
    ├─ 否 → 直接调用 Carbon::startOfYear() → 返回 1月1日
    └─ 是 → 读取 fiscalYearStart (m-d)
            ├─ 解析出月份和日期
            ├─ 设置到克隆的日期上
            └─ 判断：如果计算出的开始日期 > 输入日期？
                    ├─ 是 → 减去1年（上一个财年）
                    └─ 否 → 保持不变
```

**示例**（假设财年开始为 07-01）：

| 输入日期 | 计算过程 | 结果 |
|----------|----------|------|
| 2024-03-15 | 先设为 2024-07-01，发现 > 2024-03-15，减1年 | 2023-07-01 |
| 2024-08-15 | 设为 2024-07-01，发现 < 2024-08-15 | 2024-07-01 |
| 2024-07-01 | 设为 2024-07-01，等于输入日期 | 2024-07-01 |

### 3.3 endOfFiscalYear 算法

文件：[FiscalHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php#L49-L64)

```
调用 startOfFiscalYear() 获取开始日期
    ├─ 自定义财年 → 加1年 → 减1天
    └─ 自然年 → 调用 Carbon::endOfYear()
```

**示例**（财年开始为 07-01）：

| 输入日期 | startOfFiscalYear | endOfFiscalYear |
|----------|-------------------|-----------------|
| 2024-03-15 | 2023-07-01 | 2024-06-30 |
| 2024-08-15 | 2024-07-01 | 2025-06-30 |

**注意**：结束日期算法依赖 `startOfFiscalYear()` 的结果，因此跨年逻辑是一致的。

### 3.4 Navigation 对财年的集成

文件：[Navigation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Navigation.php#L868-L902)

#### updateStartDate (范围为 '1Y' 时)
```php
if ('1Y' === $range) {
    $fiscalHelper = app(FiscalHelperInterface::class);
    return $fiscalHelper->startOfFiscalYear($start);
}
```

#### updateEndDate (范围为 '1Y' 时)
```php
if ('1Y' === $range) {
    $fiscalHelper = app(FiscalHelperInterface::class);
    return $fiscalHelper->endOfFiscalYear($end);
}
```

**关键点**：
- 只有当范围是 `'1Y'`（年度视图）时，才会使用财年辅助类
- 其他范围（`1M`、`3M`、`6M` 等）使用自然日期计算
- 这是财年逻辑与期间选择的核心交汇点

---

## 四、报表和预算页面调用辅助类的地方

### 4.1 路由绑定层：魔术词支持

文件：[Date.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Binder/Date.php#L42-L80)

路由参数支持以下财年相关的"魔术词"：

```php
$magicWords = [
    'currentFiscalYearStart'  => $fiscalHelper->startOfFiscalYear(today()),
    'currentFiscalYearEnd'    => $fiscalHelper->endOfFiscalYear(today()),
    'previousFiscalYearStart' => $fiscalHelper->startOfFiscalYear(today())->subYear(),
    'previousFiscalYearEnd'   => $fiscalHelper->endOfFiscalYear(today())->subYear(),
];
```

这使得URL可以这样写：
```
/reports/audit/1,2,3/currentFiscalYearStart/currentFiscalYearEnd
```

### 4.2 会话初始化：Range 中间件

文件：[Range.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Middleware/Range.php#L121-L151)

```php
private function setRange(): void
{
    if (!app('session')->has('start') && !app('session')->has('end')) {
        $viewRange = Preferences::get('viewRange', '1M')->data;
        $today = today(config('app.timezone'));
        $start = Navigation::updateStartDate((string) $viewRange, $today);
        $end = Navigation::updateEndDate((string) $viewRange, $start);
        
        app('session')->put('start', $start);
        app('session')->put('end', $end);
    }
}
```

**调用链**：
1. 每个请求经过 Range 中间件
2. 如果会话中没有 `start` 和 `end`，则根据用户的 `viewRange` 偏好计算
3. 如果 `viewRange` 是 `'1Y'`，则 `updateStartDate` 和 `updateEndDate` 会调用财年辅助类

### 4.3 报表控制器

文件：[ReportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Controllers/ReportController.php#L220-L264)

```php
public function index(AccountRepositoryInterface $repository): Factory|View
{
    $customFiscalYear = Preferences::get('customFiscalYear', 0)->data;
    
    return view('reports.index', [
        'months'           => $months,
        'customFiscalYear' => $customFiscalYear,
        // ...
    ]);
}
```

**关键点**：
- 报表首页读取 `customFiscalYear` 配置并传递给视图
- 视图根据此配置决定是否显示财年快捷链接

### 4.4 报表辅助类：月份列表生成

文件：[ReportHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Report/ReportHelper.php#L102-L139)

```php
public function listOfMonths(Carbon $date): array
{
    $fiscalHelper = app(FiscalHelperInterface::class);
    // ...
    while ($start <= $end) {
        $year = $fiscalHelper->endOfFiscalYear($start)->year; // 按财年分组
        if (!array_key_exists($year, $months)) {
            $months[$year] = [
                'fiscal_start' => $fiscalHelper->startOfFiscalYear($start)->format('Y-m-d'),
                'fiscal_end'   => $fiscalHelper->endOfFiscalYear($start)->format('Y-m-d'),
                'start'        => Carbon::createFromDate($year, 1, 1)->format('Y-m-d'),
                'end'          => Carbon::createFromDate($year, 12, 31)->format('Y-m-d'),
                'months'       => [],
            ];
        }
        // ... 填充月份数据
    }
    return $months;
}
```

**关键点**：
- 月份按财年分组，而非自然年
- 每个财年分组包含 `fiscal_start`、`fiscal_end` 和自然年的 `start`、`end`
- 这是报表页面"财年 vs 自然年"切换的数据基础

### 4.5 报表页面视图

文件：[reports/index.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/resources/views/reports/index.twig#L70-L91)

```twig
{% for year, data in months %}
    <a href="#" class="date-select" data-start="{{ data.start }}" data-end="{{ data.end }}">{{ year }}</a>
    {% if customFiscalYear == 1 %}
        <br/>
        <a href="#" class="date-select" data-start="{{ data.fiscal_start }}" data-end="{{ data.fiscal_end }}">
            {{ year }} ({{ 'fiscal_year'|_|lower }})
        </a>
    {% endif %}
    {# 季度链接（非自定义财年时显示） #}
    {% if customFiscalYear == 0 %}
        (Q1, Q2, Q3, Q4 链接)
    {% endif %}
    {# 月份列表 #}
    <ul class="list-inline">
        {% for month in data.months %}
            <li><a data-start="{{ month.start }}" data-end="{{ month.end }}" ...>{{ month.formatted }}</a></li>
        {% endfor %}
    </ul>
{% endfor %}
```

**关键点**：
- 启用自定义财年后，每年会显示两个链接：自然年和财年
- 季度链接仅在非自定义财年时显示（因为季度是按自然年划分的）
- 快速链接区域也会根据 `customFiscalYear` 显示"本财年"快捷链接

### 4.6 预算页面控制器

文件：[Budget/IndexController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Controllers/Budget/IndexController.php#L95-L163)

```php
public function index(?Carbon $start = null, ?Carbon $end = null): Factory|View
{
    $range = Navigation::getViewRange(true);
    
    $start ??= session('start', today()->startOfMonth());
    $end   ??= Navigation::endOfPeriod($start, $range);
    
    $periodTitle = Navigation::periodShow($start, $range);
    $prevLoop    = $this->getPreviousPeriods($start, $range);
    $nextLoop    = $this->getNextPeriods($start, $range);
    
    // ...
}
```

**关键点**：
- 预算页面通过 `Navigation::endOfPeriod()` 间接使用财年逻辑
- 当 `viewRange` 是 `'1Y'` 时，`updateEndDate` 会调用财年辅助类
- 期间导航（`getPreviousPeriods`、`getNextPeriods`）通过 `Navigation::startOfPeriod()` 和 `Navigation::endOfPeriod()` 进行期间切换

---

## 五、跨年和短月的边界处理

### 5.1 跨年处理逻辑

#### 5.1.1 财年跨年判断
文件：[FiscalHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php#L85-L88)

```php
// if start date is after passed date, sub 1 year.
if ($startDate > $date) {
    $startDate->subYear();
}
```

**这是财年跨年的核心逻辑**：
- 当计算出的财年开始日期在输入日期之后时，说明输入日期属于上一个财年
- 例如：财年从7月1日开始，输入日期是3月15日
  - 先计算出 `2024-07-01`
  - 发现 `2024-07-01 > 2024-03-15`
  - 减1年得到 `2023-07-01`，这才是正确的财年开始

#### 5.1.2 期间切换的跨年
文件：[Navigation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Navigation.php#L89-L132)

```php
public function blockPeriods(Carbon $start, Carbon $end, string $range): array
{
    // 先获取13个指定范围的期间
    while ($loopCount < 13) {
        $workStart = $this->startOfPeriod($workStart, $range);
        $workEnd   = $this->endOfPeriod($workStart, $range);
        // ...
        $workStart->subDay()->startOfDay(); // 切换到上一期间
        ++$loopCount;
    }
    
    // 如果需要更多历史数据，按年度继续回溯
    while ($workEnd->gt($start) && $loopCount < 20) {
        $workStart = Facades\Navigation::startOfPeriod($workStart, '1Y');
        $workEnd   = Facades\Navigation::endOfPeriod($workStart, '1Y');
        // ...
    }
}
```

### 5.2 短月（小月/闰月）边界处理

#### 5.2.1 addMonthsNoOverflow 的使用

Firefly III 使用 Carbon 的 `addMonthsNoOverflow()` 方法来处理短月边界问题。

文件：[Calendar/Periodicity/Monthly.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Calendar/Periodicity/Monthly.php#L34-L37)

```php
public function nextDate(Carbon $date, int $interval = 1): Carbon
{
    return $date->clone()->addMonthsNoOverflow($this->skip($interval));
}
```

文件：[Navigation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Navigation.php#L456-L457)

```php
// increment by month (for year)
if ($diff >= 1.0001 && $diff < 12.001) {
    $increment = 'addMonthsNoOverflow';
}
```

#### 5.2.2 测试用例验证

文件：[MonthlyTest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/tests/unit/Support/Calendar/Periodicity/MonthlyTest.php#L48-L64)

```php
public static function provideIntervals(): array
{
    return [
        // 闰年2月边界
        new IntervalProvider(Carbon::parse('2020-01-29'), Carbon::parse('2020-02-29')),
        new IntervalProvider(Carbon::parse('2020-01-30'), Carbon::parse('2020-02-29')),
        new IntervalProvider(Carbon::parse('2020-01-31'), Carbon::parse('2020-02-29')),
        
        // 平年2月边界
        new IntervalProvider(Carbon::parse('2021-01-29'), Carbon::parse('2021-02-28')),
        new IntervalProvider(Carbon::parse('2021-01-30'), Carbon::parse('2021-02-28')),
        new IntervalProvider(Carbon::parse('2021-01-31'), Carbon::parse('2021-02-28')),
        
        // 其他小月边界（4月、6月、9月、11月）
        new IntervalProvider(Carbon::parse('2023-03-31'), Carbon::parse('2023-04-30')),
        new IntervalProvider(Carbon::parse('2023-05-31'), Carbon::parse('2023-06-30')),
        new IntervalProvider(Carbon::parse('2023-10-31'), Carbon::parse('2023-11-30')),
    ];
}
```

**边界处理规则**（`addMonthsNoOverflow` 的行为）：

| 起始日期 | 操作 | 结果 | 说明 |
|----------|------|------|------|
| 2020-01-29 | +1月 | 2020-02-29 | 闰年，有29日 |
| 2020-01-30 | +1月 | 2020-02-29 | 溢出，截断到月末 |
| 2020-01-31 | +1月 | 2020-02-29 | 溢出，截断到月末 |
| 2021-01-29 | +1月 | 2021-02-28 | 平年，29日不存在，截断到28日 |
| 2021-01-30 | +1月 | 2021-02-28 | 溢出，截断到月末 |
| 2021-01-31 | +1月 | 2021-02-28 | 溢出，截断到月末 |
| 2023-03-31 | +1月 | 2023-04-30 | 4月只有30天，截断到30日 |

#### 5.2.3 期间列表生成的边界

文件：[Navigation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Navigation.php#L446-L478)

```php
public function listOfPeriods(Carbon $start, Carbon $end): array
{
    // 根据日期跨度选择增量方式
    $diff = $start->diffInMonths($end, true);
    
    if ($diff < 1.0001) {
        $increment = 'addDay';           // 跨度 < 1月：按天
    } elseif ($diff < 12.001) {
        $increment = 'addMonthsNoOverflow'; // 1月 ≤ 跨度 < 1年：按月（无溢出）
    } else {
        $increment = 'addYear';          // 跨度 ≥ 1年：按年
    }
    
    // 生成期间列表
    while ($begin < $end) {
        $formatted = $begin->format($format);
        $entries[$formatted] = $begin->isoFormat($displayFormat);
        $begin->{$increment}();  // 使用选定的增量方式
    }
    
    return $entries;
}
```

**关键点**：
- 使用 `addMonthsNoOverflow` 而不是 `addMonths`，确保短月不会出现日期溢出
- 阈值使用 `1.0001` 和 `12.001` 而不是整数，避免浮点精度问题

#### 5.2.4 日期差计算的边界

文件：[Navigation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Navigation.php#L141-L183)

```php
public function diffInPeriods(string $period, int $skip, Carbon $beginning, Carbon $end): int
{
    // 先计算原始差值
    $floatDiff = $beginning->{$func}($end, true);
    
    // 季度和半年需要修正
    if ('quarterly' === $period) {
        $floatDiff /= 3;
    }
    if ('half-year' === $period) {
        $floatDiff /= 6;
    }
    
    // 向上取整
    $diff = ceil($floatDiff);
    
    // 跳过周期的修正
    if ($skip > 0) {
        $parameter = $skip + 1;
        $diff = ceil($diff / $parameter) * $parameter;
    }
    
    return (int) $diff;
}
```

**关键点**：
- 使用 `ceil()` 向上取整，确保即使部分期间也算作一个完整期间
- 跳过周期时重新对齐到周期边界

---

## 六、调用关系总图

### 6.1 年度期间（1Y）调用链

```
用户选择"年度视图" (viewRange = '1Y')
    ↓
Range 中间件 setRange()
    ├─ Navigation::updateStartDate('1Y', $today)
    │   └─ FiscalHelper::startOfFiscalYear($today)
    │       └─ Preferences::get('fiscalYearStart', '01-01')
    └─ Navigation::updateEndDate('1Y', $start)
        └─ FiscalHelper::endOfFiscalYear($start)
            └─ FiscalHelper::startOfFiscalYear($start) (内部调用)
                └─ Preferences::get('fiscalYearStart', '01-01')
    ↓
Session: start = 财年开始, end = 财年结束
    ↓
页面显示财年范围的数据
```

### 6.2 月度期间（1M）调用链

```
用户选择"月度视图" (viewRange = '1M')
    ↓
Range 中间件 setRange()
    ├─ Navigation::updateStartDate('1M', $today)
    │   └─ Carbon::startOfMonth() (自然月，不涉及财年)
    └─ Navigation::updateEndDate('1M', $start)
        └─ Carbon::endOfMonth() (自然月，不涉及财年)
    ↓
Session: start = 月初, end = 月末
```

### 6.3 财年快捷链接调用链

```
报表页面 reports/index.twig
    ├─ 显示财年链接：data-start="2023-07-01" data-end="2024-06-30"
    │   (数据来自 ReportHelper::listOfMonths() 中的 fiscal_start/fiscal_end)
    ↓
用户点击链接 → JavaScript 更新表单的日期范围
    ↓
表单提交 → ReportController::postIndex()
    ↓
跳转到报表URL：/reports/default/1,2,3/20230701/20240630
    ↓
路由绑定 Date::routeBinder()
    ├─ 解析 '20230701' → Carbon
    └─ 解析 '20240630' → Carbon
    ↓
ReportController::defaultReport($accounts, $start, $end)
    └─ 使用传入的 $start 和 $end 生成报表
```

---

## 七、容易混淆的点与设计权衡

### 7.1 财年仅在 '1Y' 范围生效

**容易混淆**：用户可能期望在所有期间都考虑财年，但实际上只有当 `viewRange` 为 `'1Y'` 时，`updateStartDate` 和 `updateEndDate` 才会调用财年辅助类。

**设计意图**：
- 月份、季度等期间是基于自然时间的，与财年设置无关
- 只有"年度视图"需要区分是自然年还是财年

### 7.2 配置读取时机不一致

**容易混淆**：
- `customFiscalYear` 在构造函数中一次性读取
- `fiscalYearStart` 在每次 `startOfFiscalYear()` 调用时读取

**设计权衡**：
- 前者是开关，变化后需要重新实例化才会生效
- 后者是具体日期，可能在运行时被修改，需要动态读取
- 这种不一致性可能导致在同一个请求中修改配置后出现意外行为

### 7.3 季度链接的隐藏逻辑

**容易混淆**：在报表页面，当启用自定义财年后，季度链接（Q1-Q4）会消失。

**设计意图**：
- 季度是按自然年划分的（1-3月Q1，4-6月Q2等）
- 在自定义财年场景下，自然季度的意义不明确
- 系统选择隐藏而不是尝试按财年重新划分季度

### 7.4 endOfFiscalYear 的计算依赖

**容易混淆**：`endOfFiscalYear()` 不直接读取配置，而是调用 `startOfFiscalYear()` 后进行日期运算。

```php
public function endOfFiscalYear(Carbon $date): Carbon
{
    $endDate = $this->startOfFiscalYear($date);  // 先算开始
    if ($this->useCustomFiscalYear) {
        $endDate->addYear();
        $endDate->subDay();
    }
    // ...
}
```

**设计优势**：
- 保持跨年逻辑的一致性
- 减少代码重复
- 确保 `startOfFiscalYear + 1年 - 1天 = endOfFiscalYear` 的数学关系始终成立

---

## 八、关键代码位置速查

| 功能 | 文件 | 关键行 |
|------|------|--------|
| 财年辅助类接口 | [FiscalHelperInterface.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Fiscal/FiscalHelperInterface.php) | 全部 |
| 财年辅助类实现 | [FiscalHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php) | 全部 |
| 服务绑定 | [FireflyServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Providers/FireflyServiceProvider.php#L194) | L194 |
| Navigation 集成财年 | [Navigation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Navigation.php#L846-L902) | L846-L902 |
| 财年跨年逻辑 | [FiscalHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php#L85-L88) | L85-L88 |
| 短月无溢出加法 | [Navigation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Navigation.php#L456-L457) | L456-L457 |
| 短月边界测试 | [MonthlyTest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/tests/unit/Support/Calendar/Periodicity/MonthlyTest.php#L48-L64) | L48-L64 |
| 配置保存 | [PreferencesController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Controllers/PreferencesController.php#L296-L303) | L296-L303 |
| 报表月份分组 | [ReportHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Report/ReportHelper.php#L102-L139) | L102-L139 |
| 路由魔术词 | [Date.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Binder/Date.php#L58-L61) | L58-L61 |
| 会话初始化 | [Range.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Middleware/Range.php#L121-L151) | L121-L151 |
| 报表视图UI | [reports/index.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/resources/views/reports/index.twig#L70-L91) | L70-L91 |
| 预算页面 | [Budget/IndexController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Controllers/Budget/IndexController.php#L95-L163) | L95-L163 |
