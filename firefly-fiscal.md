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

## 八、深入专题分析

### 8.1 自定义财年季度隐藏的根本原因

#### 8.1.1 现象描述
在报表页面 [reports/index.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/resources/views/reports/index.twig#L77-L82) 中，当 `customFiscalYear == 1` 时，季度链接（Q1-Q4）会被隐藏。

```twig
{% if customFiscalYear == 0 %}
    (Q1, Q2, Q3, Q4 链接)
{% endif %}
```

#### 8.1.2 技术层面的原因

**原因一：季度的定义锚定自然年**

系统中所有季度相关计算都是基于自然年的：

| 组件 | 实现 | 文件/位置 |
|------|------|-----------|
| QTD 开始日期 | `firstOfQuarter()` (Carbon 原生方法，自然季度) | [Navigation.php#L293](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Navigation.php#L293) |
| 季度加法 | `addQuarters()` (Carbon 原生方法，按自然季度) | [Navigation.php#L214](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Navigation.php#L214) |
| 日期差计算 | `diffInMonths() / 3` (按自然月推算) | [Navigation.php#L148-L164](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Navigation.php#L148-L164) |
| 周期枚举 | `Periodicity::Quarterly` | [Navigation.php#L61-L63](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Navigation.php#L61-L63) |

**原因二：财年季度没有统一标准**

不同国家/行业对"财年季度"的划分存在差异：
- 有的按财年均分4个等长季度（每季度3个月，但起始月偏移）
- 有的按实际月份对齐（Q1 是财年头3个月）
- 有的公司按周数划分（13周为一季度）

Firefly III 选择不预设任何一种定义，避免误导用户。

**原因三：ReportHelper 月份分组逻辑的限制**

[ReportHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Report/ReportHelper.php#L102-L139) 中的 `listOfMonths()` 方法按财年对月份分组，但只生成了年度级别的 `fiscal_start` / `fiscal_end`，没有季度级别的数据。如果要支持财年季度，需要在该方法中增加财年季度的计算逻辑。

#### 8.1.3 设计选择的权衡

| 方案 | 优点 | 缺点 |
|------|------|------|
| **隐藏季度链接（当前方案）** | 不会产生歧义，实现简单 | 用户无法快速选择季度范围 |
| 显示自然季度 | 实现简单，用户熟悉 | 与财年语境矛盾，容易混淆 |
| 显示财年季度 | 与财年概念一致 | 需要额外配置，增加复杂度 |
| 两者都显示 | 最灵活 | UI 拥挤，用户困惑 |

Firefly III 选择了最简单且不会出错的方案：直接隐藏。

---

### 8.2 闰日财年起点的跨年回退机制

#### 8.2.1 问题场景
如果用户将财年开始日期设置为 `02-29`（2月29日，闰日），在非闰年时会发生什么？

#### 8.2.2 代码分析

财年开始日期的设置代码在 [FiscalHelper.php#L82-L83](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php#L82-L83)：

```php
[$mth, $day]  = explode('-', $prefStartStr);
$startDate->day((int) $day)->month((int) $mth);
```

**Carbon 的日期溢出行为**：

当设置的日期在目标月份不存在时，Carbon 会自动"溢出"到下个月。例如：
- 输入年份 2023（平年），设置 day=29, month=2
- 2023年2月只有28天，29日不存在
- Carbon 会自动推进到 2023-03-01

#### 8.2.3 跨年判定的连锁影响

跨年判断在 [FiscalHelper.php#L86-L88](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php#L86-L88)：

```php
if ($startDate > $date) {
    $startDate->subYear();
}
```

**平年中闰日财年起点的行为推演**（配置为 `02-29`）：

| 输入日期 | 计算过程 | 溢出后日期 | 比较结果 | 最终结果 |
|----------|----------|------------|----------|----------|
| 2023-02-15 | 设置 day=29, month=2 → 溢出 | 2023-03-01 | 03-01 > 02-15 → 是 | 2022-03-01 |
| 2023-03-01 | 设置 day=29, month=2 → 溢出 | 2023-03-01 | 03-01 = 03-01 → 否 | 2023-03-01 |
| 2024-02-15 (闰年) | 设置 day=29, month=2 | 2024-02-29 | 02-29 > 02-15 → 是 | 2023-02-29 |
| 2024-03-01 (闰年) | 设置 day=29, month=2 | 2024-02-29 | 02-29 < 03-01 → 否 | 2024-02-29 |

**关键发现**：
- 在平年，闰日财年起点会"漂移"到3月1日
- 漂移导致2月15日的输入会回退到上一年的3月1日，而不是2月29日
- 闰年和平年的财年开始日期不一致，每年会有1天的偏移

#### 8.2.4 endOfFiscalYear 的连锁影响

`endOfFiscalYear()` 在 [FiscalHelper.php#L49-L63](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php#L49-L63) 中通过 `startOfFiscalYear + 1年 - 1天` 计算。

平年时，如果 `startOfFiscalYear` 是 2023-03-01（溢出结果），那么：
- endDate = 2023-03-01 + 1年 = 2024-03-01
- endDate = 2024-03-01 - 1天 = 2024-02-29

这正好是闰年的2月29日，形成了一种"偶然的正确"。

#### 8.2.5 设计选择分析

Firefly III 没有对闰日财年起点做特殊处理，原因可能是：
1. **使用场景极少**：实际中几乎没有企业将财年开始设在2月29日
2. **Carbon 行为可预测**：溢出行为虽然反直觉，但结果一致
3. **避免过度设计**：为极端边缘情况增加复杂逻辑性价比低

**潜在风险**：
- 平年和闰年的财年起止日期不一致
- 财年长度可能不是精确的365/366天
- 用户可能意外设置闰日而不自知

---

### 8.3 时区配置与跨年判定的关联

#### 8.3.1 时区配置的读取点

系统时区配置在 [config/app.php#L43](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/config/app.php#L43)：

```php
'timezone' => env_default_when_empty(env('TZ'), 'UTC'),
```

默认值为 `UTC`，可通过环境变量 `TZ` 修改。

#### 8.3.2 时区使用的不一致性

**使用配置时区的地方**：

| 位置 | 代码 | 说明 |
|------|------|------|
| 魔术词计算 | `today(config('app.timezone'))` | [Date.php#L48-L61](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Binder/Date.php#L48-L61) |
| 会话初始化 | `today(config('app.timezone'))` | [Range.php#L133](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Middleware/Range.php#L133) |
| 配置保存 | `Carbon::parse(..., config('app.timezone'))` | [PreferencesController.php#L301](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Controllers/PreferencesController.php#L301) |
| 报表首页 | `today(config('app.timezone'))` | [ReportController.php#L223](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Controllers/ReportController.php#L223) |

**未显式指定时区的地方**：

| 位置 | 代码 | 说明 |
|------|------|------|
| 路由日期解析 | `new Carbon($value)` | [Date.php#L71](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Binder/Date.php#L71) |
| 财年日期设置 | `$startDate->day()->month()` | [FiscalHelper.php#L83](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php#L83) |

#### 8.3.3 跨年判定与时区的关系

跨年判断的核心在 [FiscalHelper.php#L86](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php#L86)：

```php
if ($startDate > $date) {
```

这个比较是基于**日期时间对象**的完整比较（包括时间部分）。

**时区影响分析**：

假设：
- 应用时区：`Asia/Shanghai` (UTC+8)
- 当前时间：2024-01-01 01:00:00 (北京时间)
- 财年开始：`01-01`
- PHP 默认时区：`UTC`

| 场景 | 日期值（含时间） | 比较结果 |
|------|------------------|----------|
| `today(config('app.timezone'))` → startOfFiscalYear | 2024-01-01 00:00:00 (Asia/Shanghai) = 2023-12-31 16:00:00 UTC | 起点正确 |
| `new Carbon('2024-01-01')` (默认 UTC) | 2024-01-01 00:00:00 UTC = 2024-01-01 08:00:00 Asia/Shanghai | 日期相同但时区不同 |

**关键问题**：
- Carbon 的日期比较会考虑时区
- 如果 `$date` 和 `$startDate` 时区不同，即使"同一天"也可能比较出意外结果
- 跨年边界（12月31日 vs 1月1日）附近的日期，时区差异可能导致回退判断错误

#### 8.3.4 实际影响评估

**影响有限的原因**：

1. **PHP 默认时区通常与应用一致**：Laravel 会在启动时设置默认时区
2. **日期比较通常在同一天内**：财年开始日期通常与输入日期年份相同
3. **跨年判断是日期级别的**：虽然使用 `>` 比较，但通常涉及整日的差异

**边界情况（可能出错）**：

当输入日期正好是财年开始日期的午夜附近，且两个日期对象时区不同时：
- 输入日期：2024-07-01 00:30:00 (Asia/Shanghai)
- 计算的开始日期：2024-07-01 00:00:00 (UTC) = 2024-07-01 08:00:00 (Asia/Shanghai)
- 比较：输入日期 < 开始日期？→ 如果不转换时区，可能得到错误结果

#### 8.3.5 设计选择分析

Firefly III 在关键路径（魔术词、会话初始化）上显式使用 `config('app.timezone')`，但在某些地方依赖 PHP 默认时区。这种不一致性是常见的技术债务：

- **优点**：代码简洁，大部分场景下工作正常
- **缺点**：边界情况下可能出现难以调试的时区问题

---

### 8.4 魔术词路由后的完整消费链路

#### 8.4.1 链路总览

```
URL: /reports/default/1,2,3/currentFiscalYearStart/currentFiscalYearEnd
    ↓
路由匹配: reports.report.default
    ↓
Binder 中间件遍历路由参数
    ├─ accountList → AccountList::routeBinder()
    ├─ start_date  → Date::routeBinder('currentFiscalYearStart')
    │   └─ 命中魔术词 → FiscalHelper::startOfFiscalYear(today()) → Carbon 对象
    └─ end_date    → Date::routeBinder('currentFiscalYearEnd')
        └─ 命中魔术词 → FiscalHelper::endOfFiscalYear(today()) → Carbon 对象
    ↓
控制器方法参数注入: defaultReport(Collection $accounts, Carbon $start, Carbon $end)
    ↓
ReportGeneratorFactory::reportGenerator('Standard', $start, $end)
    ├─ 计算日期差: $start->diffInMonths($end, true)
    ├─ > 12个月 → MultiYearReportGenerator
    ├─ > 1个月  → YearReportGenerator
    └─ ≤ 1个月  → MonthReportGenerator
    ↓
生成器渲染视图 → 返回 HTML
```

#### 8.4.2 路由绑定配置

路由参数名与绑定类的映射在 [config/bindables.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/config/bindables.php#L100-L103)：

```php
'start_date' => Date::class,
'end_date'   => Date::class,
'date'       => Date::class,
```

路由定义在 [routes/web.php#L929](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/routes/web.php#L929)：

```php
Route::get('default/{accountList}/{start_date}/{end_date}', [
    'uses' => 'ReportController@defaultReport', 
    'as' => 'report.default'
]);
```

#### 8.4.3 Binder 中间件工作流程

[Binder.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Middleware/Binder.php) 的处理流程：

```php
foreach ($request->route()->parameters() as $key => $value) {
    if (array_key_exists($key, $this->binders)) {
        $boundObject = $this->performBinding($key, $value, $request->route());
        $request->route()->setParameter($key, $boundObject);
    }
}
```

**关键点**：
- 遍历所有路由参数
- 检查参数名是否在 `bindables` 配置中
- 调用对应类的 `routeBinder()` 静态方法
- 将结果替换回路由参数中

#### 8.4.4 Date 绑定器的魔术词解析

[Date.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Binder/Date.php#L42-L80) 的处理逻辑：

```
输入 value → 检查是否为魔术词
    ├─ 是 → 返回预计算的 Carbon 对象
    └─ 否 → new Carbon($value) 解析日期字符串
            ├─ 成功 → 返回 Carbon 对象
            └─ 失败 → 抛出 NotFoundHttpException
```

**全部魔术词清单**：

| 魔术词 | 含义 | 计算方式 |
|--------|------|----------|
| `currentMonthStart` | 本月初 | `today()->startOfMonth()` |
| `currentMonthEnd` | 本月末 | `today()->endOfMonth()` |
| `currentYearStart` | 本年初 | `today()->startOfYear()` |
| `currentYearEnd` | 本年末 | `today()->endOfYear()` |
| `previousMonthStart` | 上月初 | 本月初-1天→月初 |
| `previousMonthEnd` | 上月末 | 本月初-1天→月末 |
| `previousYearStart` | 上年初 | 本年初-1天→年初 |
| `previousYearEnd` | 上年末 | 本年初-1天→年末 |
| `currentFiscalYearStart` | 本财年初 | `FiscalHelper::startOfFiscalYear(today())` |
| `currentFiscalYearEnd` | 本财年末 | `FiscalHelper::endOfFiscalYear(today())` |
| `previousFiscalYearStart` | 上财年初 | 本财年初 `->subYear()` |
| `previousFiscalYearEnd` | 上财年末 | 本财年末 `->subYear()` |

**注意**：`previousFiscalYearStart` 是直接对 `startOfFiscalYear(today())` 调用 `subYear()`，而不是调用 `startOfFiscalYear(today()->subYear())`。对于非闰日财年起点，两者结果相同；但对于闰日起点，可能有细微差异。

#### 8.4.5 控制器层消费

报表控制器在 [ReportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Controllers/ReportController.php) 中直接接收 Carbon 对象：

```php
public function defaultReport(Collection $accounts, Carbon $start, Carbon $end)
{
    if ($end < $start) {
        return view('errors.error')->with(...);
    }
    
    $generator = ReportGeneratorFactory::reportGenerator('Standard', $start, $end);
    $generator->setAccounts($accounts);
    
    return $generator->generate();
}
```

**关键点**：
- 参数类型声明为 `Carbon`，由 Laravel 服务容器自动注入
- 控制器不关心日期是魔术词解析的还是字符串解析的
- 只做基本的起止日期校验（结束日期不能早于开始日期）

#### 8.4.6 报表生成器工厂的周期判定

[ReportGeneratorFactory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Generator/Report/ReportGeneratorFactory.php#L39-L63) 根据日期跨度选择生成器：

```php
$period = 'Month';
if ($start->diffInMonths($end, true) > 1) {
    $period = 'Year';
}
if ($start->diffInMonths($end, true) > 12) {
    $period = 'MultiYear';
}
```

**财年日期跨度的结果**：
- 财年跨度为 12 个月左右（精确为 365/366 天）
- `diffInMonths` 结果接近 12
- `> 1` → 触发 Year 报告
- `> 12` → 不触发（因为正好约12个月）
- 最终选择 **YearReportGenerator**

#### 8.4.7 视图层消费

[YearReportGenerator.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Generator/Report/Standard/YearReportGenerator.php#L59-L64) 将日期传递给视图：

```php
$result = view('reports.default.year', [
    'accountIds' => $accountIds,
    'reportType' => $reportType,
    'start'      => $this->start,
    'end'        => $this->end,
])->render();
```

视图再使用这些日期进行数据查询和图表渲染。

#### 8.4.8 设计亮点

1. **透明性**：控制器和下游代码完全不需要知道日期来自魔术词还是硬编码
2. **可扩展性**：新增魔术词只需在 `Date::routeBinder()` 中添加条目
3. **一致性**：所有日期入口都经过同一绑定器，行为统一
4. **SEO 友好**：魔术词 URL 具有可读性和可分享性

---

## 九、关键代码位置速查

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
| 路由魔术词 | [Date.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Binder/Date.php#L42-L80) | L42-L80 |
| 会话初始化 | [Range.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Middleware/Range.php#L121-L151) | L121-L151 |
| 报表视图UI | [reports/index.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/resources/views/reports/index.twig#L70-L91) | L70-L91 |
| 预算页面 | [Budget/IndexController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Controllers/Budget/IndexController.php#L95-L163) | L95-L163 |
| Binder 中间件 | [Binder.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Middleware/Binder.php#L61-L71) | L61-L71 |
| 绑定配置 | [bindables.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/config/bindables.php#L100-L103) | L100-L103 |
| 报表生成工厂 | [ReportGeneratorFactory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Generator/Report/ReportGeneratorFactory.php#L39-L63) | L39-L63 |
| 年度报表生成 | [YearReportGenerator.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Generator/Report/Standard/YearReportGenerator.php#L52-L74) | L52-L74 |
| 时区配置 | [config/app.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/config/app.php#L43) | L43 |
| 报表控制器入口 | [ReportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Controllers/ReportController.php#L164-L183) | L164-L183 |

---

## 十、深度专题：三个关键细节路径分析

### 10.1 previousFiscalYear 闰日差异的深度推演

#### 10.1.1 两种计算方式的代码对比

在 [Date.php#L58-L61](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Binder/Date.php#L58-L61) 中，上一财年魔术词的实现是：

```php
'currentFiscalYearStart'  => $fiscalHelper->startOfFiscalYear(today(config('app.timezone'))),
'currentFiscalYearEnd'    => $fiscalHelper->endOfFiscalYear(today(config('app.timezone'))),
'previousFiscalYearStart' => $fiscalHelper->startOfFiscalYear(today(config('app.timezone')))->subYear(),
'previousFiscalYearEnd'   => $fiscalHelper->endOfFiscalYear(today(config('app.timezone')))->subYear(),
```

**方式 A（当前实现）**：先算当前财年起止，再减一年
- `startOfFiscalYear(today())->subYear()`
- `endOfFiscalYear(today())->subYear()`

**方式 B（对比实现）**：先减一年，再算财年起止
- `startOfFiscalYear(today()->subYear())`
- `endOfFiscalYear(today()->subYear())`

#### 10.1.2 previousFiscalYearStart 的等价性验证（财年起点 = 02-29）

**场景 1：当前日期为 2024-03-15（闰年，起点后）**

| 步骤 | 方式 A（先算后减） | 方式 B（先减后算） |
|------|-------------------|-------------------|
| 1 | startOfFiscalYear(2024-03-15) = 2024-02-29 | today()->subYear() = 2023-03-15 |
| 2 | subYear() → 2023-02-29 → 溢出到 2023-03-01 | startOfFiscalYear(2023-03-15) → 溢出到 2023-03-01 |
| 结果 | **2023-03-01** | **2023-03-01** |

差异：**0 天**

**场景 2：当前日期为 2024-02-15（闰年，起点前）**

| 步骤 | 方式 A | 方式 B |
|------|--------|--------|
| 1 | startOfFiscalYear(2024-02-15) → 回退 → 2023-02-29 → 溢出 → 2023-03-01 | today()->subYear() = 2023-02-15 |
| 2 | subYear() → 2022-03-01 | startOfFiscalYear(2023-02-15) → 溢出 → 2023-03-01 → 回退 → 2022-03-01 |
| 结果 | **2022-03-01** | **2022-03-01** |

差异：**0 天**

**场景 3：当前日期为 2023-03-15（平年，起点后，平年→闰年反例）**

| 步骤 | 方式 A | 方式 B |
|------|--------|--------|
| 1 | startOfFiscalYear(2023-03-15) → 溢出 → 2023-03-01 | today()->subYear() = 2022-03-15 |
| 2 | subYear() → 2022-03-01 | startOfFiscalYear(2022-03-15) → 溢出 → 2022-03-01 |
| 结果 | **2022-03-01** | **2022-03-01** |

差异：**0 天**

**场景 4：当前日期为 2023-02-15（平年，起点前）**

| 步骤 | 方式 A | 方式 B |
|------|--------|--------|
| 1 | startOfFiscalYear(2023-02-15) → 溢出 → 2023-03-01 → 回退 → 2022-03-01 | today()->subYear() = 2022-02-15 |
| 2 | subYear() → 2021-03-01 | startOfFiscalYear(2022-02-15) → 溢出 → 2022-03-01 → 回退 → 2021-03-01 |
| 结果 | **2021-03-01** | **2021-03-01** |

差异：**0 天**

**previousFiscalYearStart 结论**：两种方式**完全等价**，差异恒为 **0 天**。

**原因**：
- 方式 A 的溢出发生在 `subYear()` 步骤（从闰年2月29日减到平年）
- 方式 B 的溢出发生在 `startOfFiscalYear()` 步骤（平年设置2月29日）
- 两者的 Carbon 溢出行为完全一致，最终都指向 3月1日
- 跨年回退逻辑也保持一致（都基于溢出后的日期进行比较）

#### 10.1.3 previousFiscalYearEnd 的不等价性发现（财年起点 = 02-29）

`endOfFiscalYear` 的实现是 `startOfFiscalYear() + 1年 - 1天`，引入了额外的日期运算，打破了对称性。

**场景 1：当前日期为 2024-03-15（闰年，起点后，平年→闰年反例）**

**方式 A（当前实现）**：
```
1. startOfFiscalYear(2024-03-15) = 2024-02-29（闰年，存在）
2. endOfFiscalYear = 2024-02-29 + 1年 = 2025-02-29 → 平年溢出到 2025-03-01
3. 减1天 = 2025-02-28
4. subYear() = 2024-02-28
结果：2024-02-28
```

**方式 B（先减后算）**：
```
1. today()->subYear() = 2023-03-15
2. startOfFiscalYear(2023-03-15) → 溢出到 2023-03-01
3. endOfFiscalYear = 2023-03-01 + 1年 = 2024-03-01
4. 减1天 = 2024-02-29（闰年，存在！）
结果：2024-02-29
```

差异：**2024-02-28 vs 2024-02-29 → 相差 1 天！**

**场景 2：当前日期为 2024-02-15（闰年，起点前）**

**方式 A**：
```
1. startOfFiscalYear(2024-02-15) = 2024-02-29 → 回退 → 2023-02-29 → 溢出 → 2023-03-01
2. endOfFiscalYear = 2023-03-01 + 1年 = 2024-03-01
3. 减1天 = 2024-02-29（闰年，存在！）
4. subYear() = 2023-02-29 → 溢出 → 2023-03-01
结果：2023-03-01
```

**方式 B**：
```
1. today()->subYear() = 2023-02-15
2. startOfFiscalYear(2023-02-15) → 溢出 → 2023-03-01 → 回退 → 2022-03-01
3. endOfFiscalYear = 2022-03-01 + 1年 = 2023-03-01
4. 减1天 = 2023-02-28（平年）
结果：2023-02-28
```

差异：**2023-03-01 vs 2023-02-28 → 相差 1 天！**

**场景 3：当前日期为 2023-03-15（平年，起点后）**

**方式 A**：
```
1. startOfFiscalYear(2023-03-15) → 溢出 → 2023-03-01
2. endOfFiscalYear = 2023-03-01 + 1年 = 2024-03-01
3. 减1天 = 2024-02-29（闰年，存在！）
4. subYear() = 2023-02-29 → 溢出 → 2023-03-01
结果：2023-03-01
```

**方式 B**：
```
1. today()->subYear() = 2022-03-15
2. startOfFiscalYear(2022-03-15) → 溢出 → 2022-03-01
3. endOfFiscalYear = 2022-03-01 + 1年 = 2023-03-01
4. 减1天 = 2023-02-28（平年）
结果：2023-02-28
```

差异：**1 天**

#### 10.1.4 差异对比总表

| 魔术词 | 当前日期（财年起点=02-29） | 方式A结果 | 方式B结果 | 差异天数 |
|--------|--------------------------|-----------|-----------|----------|
| previousFiscalYearStart | 2024-03-15 (闰年,起点后) | 2023-03-01 | 2023-03-01 | 0天 |
| previousFiscalYearStart | 2024-02-15 (闰年,起点前) | 2022-03-01 | 2022-03-01 | 0天 |
| previousFiscalYearStart | 2023-03-15 (平年,起点后) | 2022-03-01 | 2022-03-01 | 0天 |
| previousFiscalYearStart | 2023-02-15 (平年,起点前) | 2021-03-01 | 2021-03-01 | 0天 |
| **previousFiscalYearEnd** | **2024-03-15 (闰年,起点后)** | **2024-02-28** | **2024-02-29** | **1天** |
| **previousFiscalYearEnd** | **2024-02-15 (闰年,起点前)** | **2023-03-01** | **2023-02-28** | **1天** |
| **previousFiscalYearEnd** | **2023-03-15 (平年,起点后)** | **2023-03-01** | **2023-02-28** | **1天** |

**核心结论**：
- `previousFiscalYearStart`：两种方式**完全等价**，差异恒为 0 天
- `previousFiscalYearEnd`：两种方式**不等价**，闰日财年起点下差异为 **1 天**

**不等价的根本原因**：
`endOfFiscalYear` 引入了 `startOfFiscalYear() + 1年 - 1天` 的两步运算，其中：
- 方式 A：`+1年` 发生在闰年（可能溢出），`-1天` 在溢出后，最后 `subYear()` 再次溢出
- 方式 B：`+1年` 发生在平年（可能溢出到不同日期），`-1天` 在不同的基准上
- 溢出位置的不同导致最终结果差 1 天

#### 10.1.5 非闰日起点的对比验证

对于非闰日财年起点（如 07-01），Start 和 End 两种方式都完全一致：

| 魔术词 | 当前日期 | 方式A结果 | 方式B结果 | 差异天数 |
|--------|----------|-----------|-----------|----------|
| previousFiscalYearStart | 2024-03-15 | 2022-07-01 | 2022-07-01 | 0天 |
| previousFiscalYearStart | 2024-08-15 | 2023-07-01 | 2023-07-01 | 0天 |
| previousFiscalYearEnd | 2024-03-15 | 2023-06-30 | 2023-06-30 | 0天 |
| previousFiscalYearEnd | 2024-08-15 | 2024-06-30 | 2024-06-30 | 0天 |

**设计选择分析**：当前实现选择"先算财年起止，再减一年"的方式，代码更简洁。对于 99.9% 的非闰日财年起点场景，两种方式完全等价。闰日财年起点的 1 天差异属于极端边缘情况，实际影响可忽略。

---

### 10.2 ReportHelper 自然季度分组的路径

#### 10.2.1 季度链接的生成路径

ReportHelper 本身**没有**季度分组方法。季度链接是在**视图层**直接硬编码的：

文件：[reports/index.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/resources/views/reports/index.twig#L77-L82)

```twig
{% if customFiscalYear == 0 %}
    (
        <a href="#" class="date-select" data-start="{{ year }}-01-01" data-end="{{ year }}-03-31">Q1</a>,
        <a href="#" class="date-select" data-start="{{ year }}-04-01" data-end="{{ year }}-06-30">Q2</a>,
        <a href="#" class="date-select" data-start="{{ year }}-07-01" data-end="{{ year }}-09-30">Q3</a>,
        <a href="#" class="date-select" data-start="{{ year }}-10-01" data-end="{{ year }}-12-31">Q4</a>
    )
{% endif %}
```

**季度数据来源**：`year` 变量来自 [ReportHelper.php#L113](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Report/ReportHelper.php#L113) 的 `listOfMonths()` 方法返回的数组键。

#### 10.2.2 完整的季度数据流动路径

```
ReportController::index() [L220-L225]
    ↓
$this->helper->listOfMonths($start)  [ReportHelper.php#L102-L139]
    ├─ 按财年对月份分组（键为财年结束年份）
    ├─ 每个分组包含：
    │   ├─ fiscal_start / fiscal_end（财年起止）
    │   ├─ start / end（自然年起止，用于季度链接）
    │   └─ months（月份列表）
    └─ 返回 $months 数组
    ↓
视图 reports/index.twig 接收 $months
    ├─ 遍历 $months，$year 为数组键（自然年份）
    ├─ 显示自然年链接：data-start="{{ data.start }}" = {{ year }}-01-01
    ├─ 显示财年链接（如果启用）：data-start="{{ data.fiscal_start }}"
    └─ 显示季度链接（如果未启用自定义财年）：
        ├─ Q1: {{ year }}-01-01 到 {{ year }}-03-31
        ├─ Q2: {{ year }}-04-01 到 {{ year }}-06-30
        ├─ Q3: {{ year }}-07-01 到 {{ year }}-09-30
        └─ Q4: {{ year }}-10-01 到 {{ year }}-12-31
```

#### 10.2.3 季度日期的后端消费路径

当用户点击季度链接后，日期通过以下路径处理：

```
用户点击 Q1 链接 → JavaScript 更新表单隐藏域
    ↓
表单提交 → ReportController::postIndex() [L291-L347]
    ├─ $request->getStartDate() 解析 "2024-01-01"
    ├─ $request->getEndDate() 解析 "2024-03-31"
    └─ 重定向到：/reports/default/1/20240101/20240331
    ↓
路由匹配：reports.report.default
    ├─ start_date 参数 = "20240101" → Date::routeBinder()
    │   └─ new Carbon("20240101") → 2024-01-01
    └─ end_date 参数 = "20240331" → Date::routeBinder()
        └─ new Carbon("20240331") → 2024-03-31
    ↓
控制器注入：defaultReport(Collection $accounts, Carbon $start, Carbon $end)
    ↓
ReportGeneratorFactory::reportGenerator('Standard', $start, $end) [L39-L63]
    ├─ 计算日期差：2024-01-01 到 2024-03-31 = 约 2.9 个月
    ├─ diffInMonths > 1 → true
    ├─ diffInMonths > 12 → false
    └─ 选择 YearReportGenerator（而非 MonthReportGenerator）
    ↓
YearReportGenerator 渲染季度报表视图
```

#### 10.2.4 Navigation 中的季度计算

当 `viewRange` 设置为 `'3M'` 时，Navigation 提供季度级别的计算。

**准确行号与代码**：

`updateEndDate()` 方法（第 821-863 行）：
- 第 **824 行**：定义 `$functionMap` 数组，包含季度映射
- 第 **827-829 行**：检查 `$range` 是否在映射中，存在则调用对应方法

文件：[Navigation.php#L821-L834](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Navigation.php#L821-L834)

```php
// 第 821 行：方法签名
public function updateEndDate(string $range, Carbon $start): Carbon
{
    // 第 824 行：季度映射在这里！
    $functionMap = ['1D' => 'endOfDay', '1W' => 'endOfWeek', '1M' => 'endOfMonth', 
                    '3M' => 'lastOfQuarter', 'custom' => 'startOfMonth'];
    $end         = clone $start;

    // 第 827-829 行：调用映射的方法
    if (array_key_exists($range, $functionMap)) {
        $function = $functionMap[$range];
        $end->{$function}();  // '3M' 会调用 $end->lastOfQuarter()
        return $end;
    }
    // ... 后续处理 6M、1Y 等
}
```

`updateStartDate()` 方法（第 868-933 行）：
- 第 **871-877 行**：定义 `$functionMap` 数组，包含季度映射
- 第 **875 行**：季度映射具体位置 `'3M' => 'firstOfQuarter'`
- 第 **878-884 行**：检查并调用对应方法

文件：[Navigation.php#L868-L884](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Support/Navigation.php#L868-L884)

```php
// 第 868 行：方法签名
public function updateStartDate(string $range, Carbon $start): Carbon
{
    // 第 871-877 行：季度映射在这里！
    $functionMap = [
        '1D'     => 'startOfDay',
        '1W'     => 'startOfWeek',
        '1M'     => 'startOfMonth',
        // 第 875 行：季度映射具体位置
        '3M'     => 'firstOfQuarter',
        'custom' => 'startOfMonth',
    ];
    // 第 878-884 行：调用映射的方法
    if (array_key_exists($range, $functionMap)) {
        $function = $functionMap[$range];
        $start->{$function}();  // '3M' 会调用 $start->firstOfQuarter()
        return $start;
    }
    // ... 后续处理 6M、1Y 等
}
```

**Carbon 原生方法说明**：
- `firstOfQuarter()`：将日期修改为所在自然季度的第一天（1月/4月/7月/10月的1日）
- `lastOfQuarter()`：将日期修改为所在自然季度的最后一天（3月/6月/9月/12月的最后一天）

**关键注意**：这些方法都是 Carbon 原生的自然季度计算，**与财年设置完全无关**。即使财年从7月1日开始，`viewRange='3M'` 仍然会使用 1-3月、4-6月等自然季度划分。

#### 10.2.5 ReportHelper 与季度的关系总结

| 功能 | ReportHelper 参与 | 实现位置 |
|------|-------------------|----------|
| 生成年份分组（含自然年起止） | ✅ 参与 | [ReportHelper.php#L118-L119](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Report/ReportHelper.php#L118-L119) |
| 生成季度起止日期 | ❌ 不参与 | 视图层硬编码 |
| 解析季度日期 | ❌ 不参与 | Date 绑定器 + ReportFormRequest |
| 按季度计算报表 | ❌ 不参与 | YearReportGenerator（按日期跨度） |

ReportHelper 的 `listOfMonths()` 只为季度链接提供了 `year` 变量和 `data.start` / `data.end` 作为参考，实际的季度起止日期完全是视图层通过字符串拼接生成的。

---

### 10.3 Range middleware setRange 的触发时机与时区错位复现

#### 10.3.1 中间件注册顺序

文件：[bootstrap/app.php#L129-L135](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/bootstrap/app.php#L129-L135)

```php
$middleware->appendToGroup('user-full-auth', [
    Authenticate::class,      // 1. 用户认证
    MFAMiddleware::class,     // 2. 双因素认证
    Range::class,             // 3. 范围设置（setRange 在这里调用）
    InterestingMessage::class,// 4. 消息提示
]);
```

**web 组中间件顺序**（在 user-full-auth 之前执行）：
```
web 组：
  1. EncryptCookies
  2. AddQueuedCookiesToResponse
  3. StartFireflyIIISession  ← 会话启动
  4. ShareErrorsFromSession
  5. VerifyCsrfToken
  6. Binder                   ← 路由参数绑定（魔术词解析）
  7. CreateFreshApiToken

然后才是 user-full-auth 组：
  1. Authenticate
  2. MFAMiddleware
  3. Range::handle() → setRange()
  4. InterestingMessage
```

#### 10.3.2 setRange 的触发条件

文件：[Range.php#L121-L151](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Middleware/Range.php#L121-L151)

```php
private function setRange(): void
{
    // 仅当会话中没有 start 和 end 时才执行
    if (!app('session')->has('start') && !app('session')->has('end')) {
        $viewRange = Preferences::get('viewRange', '1M')->data;
        $today     = today(config('app.timezone'));              // 显式指定时区
        $start     = Navigation::updateStartDate($viewRange, $today);
        $end       = Navigation::updateEndDate($viewRange, $start);
        
        app('session')->put('start', $start);
        app('session')->put('end', $end);
    }
    
    // 仅当会话中没有 first 时才执行
    if (!app('session')->has('first')) {
        // 设置 first 日期（最早交易日期或本年初）
    }
}
```

**触发条件**：
1. 用户已认证（通过 Authenticate 中间件）
2. 会话中没有 `start` 和 `end` 变量
3. 路由使用了 `user-full-auth` 或 `admin` 中间件组

**不触发的情况**：
- 会话已存在 `start` / `end`（大部分请求）
- 用户未登录
- 路由不包含 `user-full-auth` 中间件（如 API 路由、公开页面）
- 保存偏好设置后（会清除会话，下次请求重新触发）

#### 10.3.3 时区错位问题的复现路径

要复现时区错位导致的跨年判断错误，需要满足以下**所有条件**：

##### 条件 1：应用时区与 PHP 默认时区不一致

```env
# .env
TZ=Asia/Shanghai  # 应用时区 = UTC+8
```

但 PHP 默认时区仍为 UTC（Laravel 通常会统一设置，但如果被其他代码修改）。

##### 条件 2：财年开始日期为 01-01，且当前日期接近跨年边界

当前时间：**2023-12-31 23:30:00 (Asia/Shanghai)** = **2023-12-31 15:30:00 (UTC)**

##### 条件 3：两个日期对象来自不同的创建方式

```php
// 方式 A：显式指定时区（来自 setRange）
$today = today(config('app.timezone'));  
// 结果：2023-12-31 00:00:00 Asia/Shanghai 
//      = 2023-12-30 16:00:00 UTC

// 方式 B：不显式指定时区（来自魔术词解析的日期字符串）
$date = new Carbon('2023-12-31');  
// 结果：2023-12-31 00:00:00 UTC 
//      = 2023-12-31 08:00:00 Asia/Shanghai

// 在 FiscalHelper::startOfFiscalYear 中比较
$startDate = clone $date;  // 继承 $date 的时区
$startDate->day(1)->month(1);  // 2023-01-01 UTC

if ($startDate > $date) {  // 2023-01-01 UTC > 2023-12-31 UTC ? → false
    $startDate->subYear(); // 不执行
}
```

##### 真实复现场景（跨年边界）

**配置**：
- 应用时区：`Asia/Shanghai` (UTC+8)
- 财年开始：`01-01`
- PHP 默认时区：`UTC`（假设未被正确设置）

**场景**：用户在北京时间 2024-01-01 01:00:00 访问页面

```
请求时间：2024-01-01 01:00:00 Asia/Shanghai = 2023-12-31 17:00:00 UTC

1. Range 中间件 setRange() 触发
   ├─ $today = today(config('app.timezone')) 
   │  → 2024-01-01 00:00:00 Asia/Shanghai
   │  = 2023-12-31 16:00:00 UTC
   ├─ viewRange = '1Y'
   └─ Navigation::updateStartDate('1Y', $today)
       └─ FiscalHelper::startOfFiscalYear($today)
           ├─ $startDate = clone $today  // 2024-01-01 Asia/Shanghai
           ├─ 设置 day=1, month=1      // 2024-01-01 Asia/Shanghai
           └─ 比较：2024-01-01 > 2024-01-01？→ false，不回退
           └─ 结果：2024-01-01 Asia/Shanghai ✓

2. 同时，用户点击了一个日期链接：20231231
   └─ Date::routeBinder('20231231', $route)
       └─ new Carbon('20231231')  // 2023-12-31 00:00:00 UTC
                                   = 2023-12-31 08:00:00 Asia/Shanghai
       ↓
   控制器收到：$start = 2023-12-31 UTC
   
3. 报表生成时调用 FiscalHelper::startOfFiscalYear($start)
   ├─ $startDate = clone $start  // 2023-12-31 UTC
   ├─ 设置 day=1, month=1      // 2023-01-01 UTC
   └─ 比较：2023-01-01 UTC > 2023-12-31 UTC？→ false ✓
```

**实际复现困难的原因**：

1. **Laravel 统一设置时区**：在 `bootstrap/app.php` 或 `AppServiceProvider` 中通常会调用 `date_default_timezone_set(config('app.timezone'))`，确保 PHP 默认时区与应用一致。

2. **Carbon 比较会转换时区**：当比较两个不同时区的 Carbon 对象时，Carbon 会先转换为相同时区再比较。

3. **日期通常在同一天内**：跨年判断通常比较的是日期部分，而非时间部分。

4. **会话缓存**：setRange 只在第一次请求时执行，后续请求使用会话中已缓存的日期。

##### 理论上可复现的极端场景

**配置**：
- 应用时区：`Pacific/Apia` (UTC+13, 跨国际日期变更线)
- 财年开始：`01-01`
- PHP 默认时区：`UTC`
- 手动修改 PHP 默认时区（通过代码注入）

**复现步骤**：
1. 在 `AppServiceProvider` 之后但 `Range` 中间件之前，执行 `date_default_timezone_set('UTC')`
2. 用户在当地时间 2024-01-01 01:00:00 (Pacific/Apia) 访问
   - UTC 时间：2023-12-31 12:00:00
3. setRange 中 `today(config('app.timezone'))` 创建 2024-01-01 Pacific/Apia
4. 另一个日期通过 `new Carbon('2023-12-31')` 创建，使用 UTC 时区
5. 在财年判断中可能出现预期外的回退行为

**结论**：时区错位问题在理论上存在，但在实际部署中由于 Laravel 的时区统一机制，**很难自然复现**。只有在时区配置被显式破坏的极端情况下才可能发生。

#### 10.3.4 清除会话强制重新触发 setRange

文件：[PreferencesController.php#L264-L266](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Http/Controllers/PreferencesController.php#L264-L266)

```php
// 保存偏好设置后清除日期会话
session()->forget('start');
session()->forget('end');
session()->forget('range');
```

这确保用户修改财年设置后，下次请求会重新计算日期范围。

---

### 10.4 ReportHelper 月份分组键导致的自然年与财年链接错位

#### 10.4.1 分组键的定义

文件：[ReportHelper.php#L113-L121](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/app/Helpers/Report/ReportHelper.php#L113-L121)

```php
while ($start <= $end) {
    $year = $fiscalHelper->endOfFiscalYear($start)->year; // 财年结束年份作为分组键
    if (!array_key_exists($year, $months)) {
        $months[$year] = [
            'fiscal_start' => $fiscalHelper->startOfFiscalYear($start)->format('Y-m-d'),
            'fiscal_end'   => $fiscalHelper->endOfFiscalYear($start)->format('Y-m-d'),
            'start'        => Carbon::createFromDate($year, 1, 1)->format('Y-m-d'),   // 自然年开始
            'end'          => Carbon::createFromDate($year, 12, 31)->format('Y-m-d'), // 自然年结束
            'months'       => [],
        ];
    }
    // ... 添加月份到 $months[$year]['months']
}
```

**关键点**：
- 数组键 `$year` = `endOfFiscalYear($start)->year`（财年结束年份）
- `fiscal_start` / `fiscal_end` = 真正的财年起止日期
- `start` / `end` = 该年份的**自然年**起止（1月1日到12月31日）
- `months` 数组 = 按财年分组的月份列表

#### 10.4.2 错位现象分析（财年起点 = 07-01）

假设财年从 7月1日开始，遍历日期从 2024年1月到 2025年6月：

| 遍历月份 | 财年结束年份（分组键） | 该分组下累计的月份 |
|----------|----------------------|-------------------|
| 2024-01 | 2024（endOfFiscalYear = 2024-06-30） | 2024-01 |
| 2024-02 | 2024 | 2024-01, 2024-02 |
| ... | ... | ... |
| 2024-06 | 2024 | 2024-01 ~ 2024-06（共6个月） |
| 2024-07 | 2025（endOfFiscalYear = 2025-06-30） | 2024-07 |
| 2024-08 | 2025 | 2024-07, 2024-08 |
| ... | ... | ... |
| 2024-12 | 2025 | 2024-07 ~ 2024-12（共6个月） |
| 2025-01 | 2025 | 2024-07 ~ 2025-01（共7个月） |
| ... | ... | ... |
| 2025-06 | 2025 | 2024-07 ~ 2025-06（共12个月） |

**错位表现**（以 "2025" 分组为例）：

| 项目 | 实际值 | 标签/期望 | 错位程度 |
|------|--------|-----------|----------|
| 分组标签 | "2025" | 自然年2025 | 标签一致 |
| 财年链接范围 | 2024-07-01 至 2025-06-30 | "2025 (fiscal year)" | 符合财年命名惯例（以结束年命名） |
| 自然年链接范围 | 2025-01-01 至 2025-12-31 | "2025" | 标签一致，但与月份列表不匹配 |
| 月份列表 | 2024-07 ~ 2025-06（12个月） | 2025年全年的月份 | **严重错位** |

#### 10.4.3 视图层的呈现错位

文件：[reports/index.twig#L70-L91](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/resources/views/reports/index.twig#L70-L91)

```twig
{% for year, data in months %}
    {# 自然年链接 #}
    <a href="#" data-start="{{ data.start }}" data-end="{{ data.end }}">{{ year }}</a>
    
    {# 财年链接（如果启用） #}
    {% if customFiscalYear == 1 %}
        <a href="#" data-start="{{ data.fiscal_start }}" data-end="{{ data.fiscal_end }}">
            {{ year }} (fiscal year)
        </a>
    {% endif %}
    
    {# 季度链接（仅非自定义财年时显示） #}
    {% if customFiscalYear == 0 %}
        (Q1, Q2, Q3, Q4)
    {% endif %}
    
    {# 月份列表 #}
    <ul class="list-inline">
        {% for month in data.months %}
            <li><a data-start="{{ month.start }}" data-end="{{ month.end }}">{{ month.formatted }}</a></li>
        {% endfor %}
    </ul>
{% endfor %}
```

**用户视角的错位感受**（财年起点 = 07-01，启用自定义财年）：

用户看到的 "2025" 分组下：
```
2025           ← 点击跳转到 2025-01-01 至 2025-12-31（自然年）
2025 (fiscal year)  ← 点击跳转到 2024-07-01 至 2025-06-30（财年）
· 7月 · 8月 · 9月 · 10月 · 11月 · 12月   ← 这些是 2024 年的月份！
· 1月 · 2月 · 3月 · 4月 · 5月 · 6月      ← 这些是 2025 年的月份
```

**错位问题清单**：

| # | 问题 | 影响 |
|---|------|------|
| 1 | "2025" 标题下显示 2024 年 7-12 月的月份 | 用户困惑，以为自己看错了年份 |
| 2 | 自然年链接（2025全年）与下方月份列表（2024-07~2025-06）不匹配 | 点击年份链接看到的范围和下方展示的月份不一致 |
| 3 | 财年链接与月份列表匹配，但标签只有年份没有"财年"强调 | 用户可能混淆两个链接的含义 |
| 4 | 月份按时间顺序排列，7月在前1月在后，不符合日历直觉 | 用户需要适应"从财年开始月到结束月"的排列 |

#### 10.4.4 季度链接的隐含错位（非自定义财年时）

当 `customFiscalYear == 0` 时，季度链接使用 `{{ year }}-01-01` 等硬编码格式：

文件：[reports/index.twig#L78-L81](file:///d:/fz/0601-1/solo-dogfeeding/code/98-firefly-iii/resources/views/reports/index.twig#L78-L81)

```twig
<a href="#" data-start="{{ year }}-01-01" data-end="{{ year }}-03-31">Q1</a>
<a href="#" data-start="{{ year }}-04-01" data-end="{{ year }}-06-30">Q2</a>
...
```

在非自定义财年模式下（`viewRange` 为自然年），分组键 `$year` 仍然是财年结束年份。但因为财年就是自然年（01-01到12-31），所以：
- 分组键 = 自然年份
- 月份列表 = 1月到12月
- 季度链接 = 该年的自然季度

**这种情况下是完全对齐的，没有错位。**

但如果用户虽然启用了自定义财年（比如 07-01），但页面上仍然显示自然季度（实际不会，因为 `customFiscalYear == 1` 时季度链接被隐藏了），就会有严重错位。

**这也是为什么自定义财年模式下要隐藏季度链接的另一个原因**：季度链接是按自然年硬编码的，与按财年分组的月份列表不匹配。

#### 10.4.5 设计选择的权衡

| 方案 | 优点 | 缺点 |
|------|------|------|
| **当前方案（财年结束年为键）** | 财年链接与月份列表完全匹配 | 自然年链接与月份列表错位，用户困惑 |
| 自然年为键 | 自然年链接与月份匹配 | 财年链接被拆分到两个年份分组，不直观 |
| 两种分组并行展示 | 最清晰 | UI复杂，信息重复 |
| 月份按日历顺序排列 | 符合用户直觉 | 财年概念被弱化 |

**当前设计的核心理由**：
1. **财年优先**：Firefly III 的报表设计以财年为核心组织单位，月份按财年分组是合理的
2. **标签简洁**：用财年结束年份作为标签，符合会计惯例（FY2025 表示 2024-07 至 2025-06）
3. **财年链接准确**：财年链接与下方月份列表完全一致，点击财年链接看到的就是下方展示的那些月份
4. **自然年链接作为补充**：自然年链接只是额外提供的快捷入口，与月份列表不完全匹配是可以接受的

#### 10.4.6 代码层面的因果链

```
ReportHelper::listOfMonths() [L102-L139]
    ↓
$year = endOfFiscalYear($start)->year  [L113]
    ↓  （决定了分组键 = 财年结束年份）
    ↓
$fiscal_start / $fiscal_end = 财年实际起止  [L116-L117]
    ↓  （与分组键的财年概念一致）
    ↓
$start / $end = Carbon::createFromDate($year, 1, 1) ...  [L118-L119]
    ↓  （用分组键年份直接生成自然年起止，不考虑财年偏移）
    ↓
$months[] = 当月数据  [L126-L132]
    ↓  （月份归属由财年决定，与分组键年份不完全重叠）
    ↓
视图 reports/index.twig
    ├─ year 变量 = 分组键（财年结束年）
    ├─ 自然年链接 → data.start / data.end（自然年）
    ├─ 财年链接 → data.fiscal_start / fiscal_end（财年）
    └─ 月份列表 → data.months（按财年分组）
        → 自然年链接与月份列表错位
        → 财年链接与月份列表一致
```

**错位的直接原因**：`start` / `end`（自然年）直接从分组键 `$year` 生成，没有考虑财年开始月份的偏移，而 `months` 数组是按财年分组的。两者基于不同的时间基准。
