# 财年与自定义期间：代码链路全链路分析

本文档梳理 Firefly III 系统中"财年（Fiscal Year）与自定义期间如何从用户偏好、辅助方法、到报表查询时间范围的完整代码渗透路径。

---

## 一、用户偏好层：财年设置的存储与读取

### 1.1 两个关键偏好项

系统涉及两个核心偏好，由 [Preferences.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Preferences.php) 管理：

| 偏好键名 | 类型 | 默认值 | 含义 |
|---------|------|-------|-----|
| `customFiscalYear` | bool | `false` | 是否启用自定义财年 |
| `fiscalYearStart` | string | `'01-01'` | 财年起始月日（格式 `MM-DD`） |

### 1.2 偏好的读取方法 [Preferences::get()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Preferences.php#L105-L117)

```php
public function get(string $name, array|bool|int|string|null $default = null): ?Preference
{
    $user = auth()->user();
    if (null === $user) {
        $preference       = new Preference();
        $preference->data = $default;
        return $preference;
    }
    return $this->getForUser($user, $name, $default);
}
```

核心逻辑：
- 未登录用户返回带默认值的虚拟 Preference 对象
- 已登录用户走 `getForUser()` 从数据库 `preferences` 表查询
- 若找不到且 `$default` 非空，自动调用 `setForUser()` 写入默认值并返回

### 1.3 偏好的设置方法 [Preferences::set()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Preferences.php#L276-L290)

偏好写入通过 `Preferences::set()`，最终走 `setForUser()`：
- 写入 `preferences` 表，字段结构：`user_id` + `name` + `data`（加密与否取决于调用方）
- 写入后通过 `Cache::forever()` 做缓存

### 1.4 用户偏好页面的保存入口 [PreferencesController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/PreferencesController.php#L296-L303)

```php
// custom fiscal year
$customFiscalYear  = 1 === (int) $request->input('customFiscalYear');
Preferences::set('customFiscalYear', $customFiscalYear);
$fiscalYearString  = (string) $request->input('fiscalYearStart');
if ('' !== $fiscalYearString) {
    $fiscalYearStart = Carbon::parse($fiscalYearString, config('app.timezone'))->format('m-d');
    Preferences::set('fiscalYearStart', $fiscalYearStart);
}
```

关键点：`fiscalYearStart` 被格式化为 `MM-DD` 字符串（如 `'07-01'`）后存入。

用户偏好的展示 [index() 方法](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/PreferencesController.php#L120-L126) 读取：
```php
$customFiscalYear               = Preferences::get('customFiscalYear', 0)->data;
$fiscalYearStartStr             = Preferences::get('fiscalYearStart', '01-01')->data;
```

偏好页面模板 [preferences/index.twig](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/resources/views/preferences/index.twig#L111-L113) 暴露了两个控件：
- `customFiscalYear` 复选框
- `fiscalYearStart` 日期选择器

---

## 二、财年辅助层：FiscalHelper 计算财年起止

### 2.1 接口定义 [FiscalHelperInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Helpers/Fiscal/FiscalHelperInterface.php)

```php
interface FiscalHelperInterface
{
    public function endOfFiscalYear(Carbon $date): Carbon;
    public function startOfFiscalYear(Carbon $date): Carbon;
}
```

### 2.2 实现类 [FiscalHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php)

**构造函数** [L41-L44](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php#L41-L44)：
```php
public function __construct()
{
    $this->useCustomFiscalYear = (bool) Preferences::get('customFiscalYear', false)->data;
}
```
构造时即读取用户偏好，决定后续计算分支。

**startOfFiscalYear()** [L72-L97](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php#L72-L97)：

```php
public function startOfFiscalYear(Carbon $date): Carbon
{
    $startDate = clone $date;
    if ($this->useCustomFiscalYear) {
        $prefStartStr = Preferences::get('fiscalYearStart', '01-01')->data;
        [$mth, $day]  = explode('-', $prefStartStr);
        $startDate->day((int) $day)->month((int) $mth);
        // 如果计算出的起始日期在传入日期之后，说明要回退一年
        if ($startDate > $date) {
            $startDate->subYear();
        }
    }
    if (false === $this->useCustomFiscalYear) {
        $startDate->startOfYear();  // 自然年 1 月 1 日
    }
    return $startDate;
}
```

**endOfFiscalYear()** [L49-L64](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php#L49-L64)：

```php
public function endOfFiscalYear(Carbon $date): Carbon
{
    $endDate = $this->startOfFiscalYear($date);
    if ($this->useCustomFiscalYear) {
        $endDate->addYear();
        $endDate->subDay();  // 起始日 + 1 年 - 1 天
    }
    if (false === $this->useCustomFiscalYear) {
        $endDate->endOfYear();  // 自然年 12 月 31 日
    }
    return $endDate;
}
```

**示例（假设财年从 7 月 1 日开始）：
- 传入 `2026-03-15` → `startOfFiscalYear()` 返回 `2025-07-01`
- 传入 `2026-09-15` → `startOfFiscalYear()` 返回 `2026-07-01`

---

## 三、导航辅助层：Navigation 期间计算

[Navigation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php) 是全局的期间计算核心类，管理从日、周、月、季度、年各种粒度的日期边界。

### 3.1 财年渗透点：updateStartDate / updateEndDate

**[updateStartDate()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php#L868-L945)** 和 **[updateEndDate()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php#L821-L863) 当 `range === '1Y'` 时，会委托给 FiscalHelper：

```php
// updateStartDate L896-L902
if ('1Y' === $range) {
    $fiscalHelper = app(FiscalHelperInterface::class);
    return $fiscalHelper->startOfFiscalYear($start);
}

// updateEndDate L846-L852
if ('1Y' === $range) {
    $fiscalHelper = app(FiscalHelperInterface::class);
    return $fiscalHelper->endOfFiscalYear($end);
}
```

这是财年逻辑"1 年期"在 Navigation 中的关键渗透点。其他范围（1D/1W/1M/3M/6M）走 Carbon 的标准方法。

### 3.2 startOfPeriod / endOfPeriod

**[startOfPeriod()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php#L651-L724)**：
- 通过 `$functionMap` 将期间字符串映射到 Carbon 方法
- 注意：`startOfPeriod('1Y')` 直接走 `startOfYear()`，**不**区分财年

**[endOfPeriod()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php#L185-L384)**：
- 同样通过 functionMap 映射
- `endOfPeriod('1Y')` 走 `addYears()->subDay()，**不**区分财年

> 注意：`startOfPeriod/endOfPeriod` 的 1Y 期间不感知财年；只有 `updateStartDate/updateEndDate` 的 1Y 才感知财年。

### 3.3 periodShow 期间显示格式化

**[periodShow()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php#L499-L538)**：根据期间类型返回本地化显示字符串，如 `'1Y'` 返回年份格式，`'3M'` 返回 `Q%d %d` 格式。

---

## 四、路由绑定层：Date Binder 魔术词解析

报表 URL 形如 `/reports/default/.../currentFiscalYearStart/currentFiscalYearEnd` 中的魔术词由 [Date.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Binder/Date.php) 解析。

### 4.1 魔术词映射表 [L47-L62](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Binder/Date.php#L47-L62)

```php
$magicWords = [
    'currentMonthStart'       => today()->startOfMonth(),
    'currentMonthEnd'         => today()->endOfMonth(),
    'currentYearStart'        => today()->startOfYear(),
    'currentYearEnd'          => today()->endOfYear(),
    'previousMonthStart'      => today()->startOfMonth()->subDay()->startOfMonth(),
    'previousMonthEnd'        => today()->startOfMonth()->subDay()->endOfMonth(),
    'previousYearStart'       => today()->startOfYear()->subDay()->startOfYear(),
    'previousYearEnd'         => today()->startOfYear()->subDay()->endOfYear(),
    // 财年相关：
    'currentFiscalYearStart'  => $fiscalHelper->startOfFiscalYear(today()),
    'currentFiscalYearEnd'    => $fiscalHelper->endOfFiscalYear(today()),
    'previousFiscalYearStart' => $fiscalHelper->startOfFiscalYear(today())->subYear(),
    'previousFiscalYearEnd'   => $fiscalHelper->endOfFiscalYear(today())->subYear(),
];
```

这是财年从 URL 魔术词直接解析为实际 Carbon 日期的入口。`routeBinder()` 方法将魔术词查找表，找不到则尝试 `new Carbon($value)` 解析具体日期字符串。

---

## 五、报表查询层：从控制器 → 表单 → 查询构造器

### 5.1 报表入口 [ReportController::index()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/ReportController.php#L220-L264)

```php
$start            = clone session('first', today(config('app.timezone'));
$months           = $this->helper->listOfMonths($start);
$customFiscalYear = Preferences::get('customFiscalYear', 0)->data;
```

- 传递给视图 [reports/index.twig](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/resources/views/reports/index.twig)
- 视图根据 `customFiscalYear` 决定是否显示财年快速链接（`currentFiscalYearStart` / `currentFiscalYearEnd`）

### 5.2 报表月份列表生成 [ReportHelper::listOfMonths()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Helpers/Report/ReportHelper.php#L101-L138)

```php
public function listOfMonths(Carbon $date): array
{
    $fiscalHelper = app(FiscalHelperInterface::class);
    // ...
    while ($start <= $end) {
        $year = $fiscalHelper->endOfFiscalYear($start)->year; // 财年归属年
        if (!array_key_exists($year, $months)) {
            $months[$year] = [
                'fiscal_start' => $fiscalHelper->startOfFiscalYear($start)->format('Y-m-d'),
                'fiscal_end'   => $fiscalHelper->endOfFiscalYear($start)->format('Y-m-d'),
                'start'        => Carbon::createFromDate($year, 1, 1)->format('Y-m-d'),
                'end'          => Carbon::createFromDate($year, 12, 31)->format('Y-m-d'),
                'months'       => [],
            ];
        }
        // ...
    }
}
```

每个月按财年结束日期的年份分组，每组同时包含自然年和财年起止。

### 5.3 报表提交处理 [ReportController::postIndex()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/ReportController.php#L291-L348)

通过 [ReportFormRequest](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Requests/ReportFormRequest.php) 获取起止日期：

- `getStartDate()` [L181-L212] 解析 `daterange` 表单字段（格式 `YYYY-MM-DD - YYYY-MM-DD`）
- `getEndDate()` [L142-L173] 同上

然后重定向到具体报表路由（URL 中携带 `$start->format('Ymd')` / `$end->format('Ymd')`）。

### 5.4 报表生成器入口（以 defaultReport 为例 [L164-L183](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/ReportController.php#L164-L183)

```php
public function defaultReport(Collection $accounts, Carbon $start, Carbon $end)
{
    $start->endOfDay();
    $end->endOfDay();
    $generator = ReportGeneratorFactory::reportGenerator('Standard', $start, $end);
    $generator->setAccounts($accounts);
    return $generator->generate();
}
```

注意：路由参数 `$start` / `$end` 已经由 Date Binder 将魔术词解析为 Carbon 对象。

---

## 六、数据查询层：GroupCollector 时间范围过滤

时间范围最终落到 [TimeCollection.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Helpers/Collector/Extensions/TimeCollection.php) 的 SQL 构造。

### 6.1 setRange() [L570-L585](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Helpers/Collector/Extensions/TimeCollection.php#L570-L585)

```php
public function setRange(?Carbon $start, ?Carbon $end): GroupCollectorInterface
{
    if ($start instanceof Carbon && $end instanceof Carbon && $end < $start) {
        [$start, $end] = [$end, $start];
    }
    $startStr = $start?->format('Y-m-d 00:00:00');
    $endStr   = $end?->format('Y-m-d 23:59:59');
    if ($start instanceof Carbon) {
        $this->query->where('transaction_journals.date', '>=', $startStr);
    }
    if ($end instanceof Carbon) {
        $this->query->where('transaction_journals.date', '<=', $endStr);
    }
    return $this;
}
```

### 6.2 setAfter() / setBefore() 单独设置单边边界：

```php
// setAfter L435-L441
$this->query->where('transaction_journals.date', '>=', $date->format('Y-m-d 00:00:00'));

// setBefore L446-L452
$this->query->where('transaction_journals.date', '<=', $date->format('Y-m-d 23:59:59'));
```

这是财年期间最终落到 SQL `WHERE` 子句的终端点。查询本身不关心财年语义——只接收上层传进来的 `$start`、`$end` Carbon 对象已经在前面各层完成了财年逻辑的转换。

---

## 七、ParseDateString：日期字符串解析辅助

[ParseDateString.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/ParseDateString.php) 处理用户输入的日期字符串，主要用于搜索和规则引擎等场景。

### 7.1 parseDate() [L77-L120] 支持的格式：
- 关键词：`today`、`yesterday`、`start of this month`、`end of this year` 等
- 标准日期：`YYYY-MM-DD`
- 相对日期：`+1d`、`-2m`、`+3q`、`-1y` 等
- 年份单独：`2026`

### 7.2 parseRange() [L122-L163] 支持的范围格式（通配符 xx/xxxx）：
- `xxxx-xx-DD` → 每年某日
- `xxxx-MM-xx` → 某年某月
- `YYYY-xx-xx` → 某年全年
- `xxxx-MM-DD` → 每年某月某日
- `YYYY-xx-DD` → 某年某日
- `YYYY-MM-xx` → 某年某月

> 注意：ParseDateString 目前不直接感知财年，它处理的是自然日期。

---

## 八、整体链路图

```
用户偏好设置页面
    ↓ (PreferencesController::postIndex())
    ↓ Preferences::set('customFiscalYear')
    ↓ Preferences::set('fiscalYearStart')
    ↓
preferences 表 (user_id, name, data)
    ↑
    ↑ Preferences::get()
    ↑
FiscalHelper (构造时读取 customFiscalYear)
    ├─ startOfFiscalYear(Carbon) → 财年起始日
    └─ endOfFiscalYear(Carbon)   → 财年结束日
        ↑                       ↑
        │                       │
        ├─ Date Binder 魔术词       ├─ Navigation::updateStartDate('1Y')
        │   currentFiscalYearStart  └─ Navigation::updateEndDate('1Y')
        │   currentFiscalYearEnd
        │   previousFiscalYearStart
        │   previousFiscalYearEnd
        ↓                       ↓
Report 路由参数 (Carbon 对象)
        ↓
ReportController::defaultReport/budgetReport/...
        ↓
ReportGeneratorFactory → 具体 Generator
        ↓
GroupCollector::setRange($start, $end)
        ↓
SQL: WHERE transaction_journals.date >= 'YYYY-MM-DD 00:00:00'
     AND transaction_journals.date <= 'YYYY-MM-DD 23:59:59'
```

---

## 九、关键代码索引

| 层级 | 文件 | 关键方法/行 |
|------|------|-----------|
| 偏好存储 | [Preferences.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Preferences.php) | `get()` L105, `set()` L276 |
| 偏好保存 | [PreferencesController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/PreferencesController.php) | L296-L303 |
| 财年计算 | [FiscalHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Helpers/Fiscal/FiscalHelper.php) | `startOfFiscalYear()` L72, `endOfFiscalYear()` L49 |
| 期间导航 | [Navigation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Navigation.php) | `updateStartDate()` L868, `updateEndDate()` L821 |
| 路由绑定 | [Date.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Binder/Date.php) | 魔术词映射 L47-L62 |
| 报表入口 | [ReportController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/ReportController.php) | `index()` L220, `postIndex()` L291 |
| 报表月份 | [ReportHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Helpers/Report/ReportHelper.php) | `listOfMonths()` L101 |
| 查询构造 | [TimeCollection.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Helpers/Collector/Extensions/TimeCollection.php) | `setRange()` L570 |
