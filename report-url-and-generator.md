# 报表 URL 到 SQL 时间过滤：头尾链路全链路分析

本文档梳理 Firefly III 报表系统中两条关键链路的代码事实：
- **头部链路**：Twig 模板拼接报表 URL → Binder 中间件触发 Date 类解析魔术词 → 控制器收到 Carbon 对象
- **尾部链路**：ReportController 交给 ReportGeneratorFactory → Generator 渲染视图 → 视图 AJAX 调 report-data 端点 → Report 子控制器调用 GroupCollector.setRange() → SQL WHERE 子句

---

## 一、头部链路：Twig 模板 → URL → Date Binder → Carbon

### 1.1 Twig 快捷链接的 URL 拼接

报表入口页 [reports/index.twig](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/resources/views/reports/index.twig) 中有两处拼报表 URL：

**快捷链接区域** [L141-L177](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/resources/views/reports/index.twig#L141-L177)：

```twig
{# 自然年快捷链接 #}
<a href="{{ route('reports.report.default',[accountList, 'currentYearStart','currentYearEnd']) }}">
    {{ 'report_this_year_quick'|_ }}
</a>

{# 财年快捷链接 —— 仅当 customFiscalYear == 1 时渲染 #}
{% if customFiscalYear == 1 %}
<a href="{{ route('reports.report.default',[accountList, 'currentFiscalYearStart','currentFiscalYearEnd']) }}">
    {{ 'report_this_fiscal_year_quick'|_ }}
</a>
{% endif %}
```

**预设日期选择区域** [L70-L91](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/resources/views/reports/index.twig#L70-L91)：

```twig
{% for year, data in months %}
    {# 自然年 #}
    <a href="#" class="date-select" data-start="{{ data.start }}" data-end="{{ data.end }}">{{ year }}</a>

    {# 财年 —— 仅当 customFiscalYear == 1 时渲染 #}
    {% if customFiscalYear == 1 %}
    <a href="#" class="date-select" data-start="{{ data.fiscal_start }}" data-end="{{ data.fiscal_end }}">
        {{ year }} ({{ 'fiscal_year'|_|lower }})
    </a>
    {% endif %}

    {# 季度 —— 仅当 customFiscalYear == 0 时渲染（因为财年季度≠自然季度） #}
    {% if customFiscalYear == 0 %}
    (<a href="#" class="date-select" data-start="{{ year }}-01-01" data-end="{{ year }}-03-31">Q1</a>, ...)
    {% endif %}
{% endfor %}
```

两种机制的行为差异：
- **快捷链接**：直接调用 `route()` 生成 URL，日期参数使用魔术词字符串（如 `currentFiscalYearStart`）
- **预设日期选择**：`data-start`/`data-end` 是实际日期字符串（如 `2025-07-01`），由前端 JS 写入 `#inputDateRange` 输入框，随表单 POST 提交

### 1.2 路由定义与参数绑定

[web.php L924-L941](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/routes/web.php#L924-L941)：

```php
Route::group(
    ['middleware' => 'user-full-auth', 'namespace' => 'FireflyIII\Http\Controllers', 'prefix' => 'reports', 'as' => 'reports.'],
    static function (): void {
        Route::get('', ['uses' => 'ReportController@index', 'as' => 'index']);
        Route::get('options/{reportType}', ['uses' => 'ReportController@options', 'as' => 'options']);
        Route::get('default/{accountList}/{start_date}/{end_date}', ['uses' => 'ReportController@defaultReport', 'as' => 'report.default']);
        Route::get('audit/{accountList}/{start_date}/{end_date}', ['uses' => 'ReportController@auditReport', 'as' => 'report.audit']);
        Route::get('category/{accountList}/{categoryList}/{start_date}/{end_date}', ['uses' => 'ReportController@categoryReport', 'as' => 'report.category']);
        Route::get('budget/{accountList}/{budgetList}/{start_date}/{end_date}', ['uses' => 'ReportController@budgetReport', 'as' => 'report.budget']);
        Route::get('tag/{accountList}/{tagList}/{start_date}/{end_date}', ['uses' => 'ReportController@tagReport', 'as' => 'report.tag']);
        Route::get('double/{accountList}/{doubleList}/{start_date}/{end_date}', ['uses' => 'ReportController@doubleReport', 'as' => 'report.double']);
        Route::post('', ['uses' => 'ReportController@postIndex', 'as' => 'index.post']);
    }
);
```

关键点：路由参数名 `start_date` / `end_date` 在 bindables 配置中被映射到 Date binder。

### 1.3 Binder 中间件：参数名 → Binder 类

[Binder.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Middleware/Binder.php) 作为全局中间件，在请求处理前遍历所有路由参数：

```php
public function handle($request, Closure $next)
{
    foreach ($request->route()->parameters() as $key => $value) {
        if (array_key_exists($key, $this->binders)) {
            $boundObject = $this->performBinding($key, $value, $request->route());
            $request->route()->setParameter($key, $boundObject);
        }
    }
    return $next($request);
}
```

`$this->binders` 来自 [bindables.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/config/bindables.php#L100-L103)：

```php
// dates
'start_date' => Date::class,
'end_date'   => Date::class,
'date'       => Date::class,
```

当路由参数名是 `start_date` 或 `end_date` 时，其原始字符串值被传给 [Date::routeBinder()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Binder/Date.php#L42-L80)。

### 1.4 Date Binder 魔术词解析

[Date.php L42-L80](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Binder/Date.php#L42-L80) 的核心逻辑：

```php
public static function routeBinder(string $value, Route $route): Carbon
{
    $fiscalHelper = app(FiscalHelperInterface::class);
    $magicWords   = [
        'currentMonthStart'       => today()->startOfMonth(),
        'currentMonthEnd'         => today()->endOfMonth(),
        'currentYearStart'        => today()->startOfYear(),
        'currentYearEnd'          => today()->endOfYear(),
        'previousMonthStart'      => today()->startOfMonth()->subDay()->startOfMonth(),
        'previousMonthEnd'        => today()->startOfMonth()->subDay()->endOfMonth(),
        'previousYearStart'       => today()->startOfYear()->subDay()->startOfYear(),
        'previousYearEnd'         => today()->startOfYear()->subDay()->endOfYear(),
        // 财年魔术词：
        'currentFiscalYearStart'  => $fiscalHelper->startOfFiscalYear(today()),
        'currentFiscalYearEnd'    => $fiscalHelper->endOfFiscalYear(today()),
        'previousFiscalYearStart' => $fiscalHelper->startOfFiscalYear(today())->subYear(),
        'previousFiscalYearEnd'   => $fiscalHelper->endOfFiscalYear(today())->subYear(),
    ];
    if (array_key_exists($value, $magicWords)) {
        return $magicWords[$value];       // 魔术词 → Carbon
    }
    return new Carbon($value);             // 日期字符串 → Carbon（如 "20260701"）
}
```

**完整链路示例**：

```
Twig: route('reports.report.default', [accountList, 'currentFiscalYearStart', 'currentFiscalYearEnd'])
  ↓ 生成 URL: /reports/default/1,2,3/currentFiscalYearStart/currentFiscalYearEnd
  ↓
Binder 中间件: 参数 start_date='currentFiscalYearStart'
  ↓ bindables.php: start_date => Date::class
  ↓ Date::routeBinder('currentFiscalYearStart')
  ↓ FiscalHelper->startOfFiscalYear(today()) → Carbon('2025-07-01')
  ↓
ReportController::defaultReport(Collection $accounts, Carbon $start, Carbon $end)
  // $start = Carbon('2025-07-01'), $end = Carbon('2026-06-30')
```

### 1.5 postIndex 分支：表单提交的日期解析

[ReportController::postIndex()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/ReportController.php#L291-L348) 处理表单 POST：

```php
$start = $request->getStartDate()->format('Ymd');  // 如 "20260701"
$end   = $request->getEndDate()->format('Ymd');
$url   = match ($reportType) {
    default    => route('reports.report.default', [$accounts, $start, $end]),
    'category' => route('reports.report.category', [$accounts, $categories, $start, $end]),
    // ...
};
return redirect($url);
```

这里 `$start`/`$end` 是 `Ymd` 格式的**数字字符串**（如 `20260701`），Date Binder 会走 `new Carbon($value)` 分支解析。

---

## 二、中间层：ReportController → ReportGeneratorFactory → Generator

### 2.1 ReportController 各报表方法

以 [defaultReport()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/ReportController.php#L164-L183) 为例：

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

所有 6 种报表方法（default/audit/budget/category/tag/double）结构一致：收到 Carbon → 传给 Factory → 调 generator。

### 2.2 ReportGeneratorFactory 期间选择

[ReportGeneratorFactory::reportGenerator()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/ReportGeneratorFactory.php#L39-L63)：

```php
public static function reportGenerator(string $type, Carbon $start, Carbon $end): ReportGeneratorInterface
{
    $period = 'Month';
    if ($start->diffInMonths($end, true) > 1) {
        $period = 'Year';
    }
    if ($start->diffInMonths($end, true) > 12) {
        $period = 'MultiYear';
    }
    $class = sprintf('FireflyIII\Generator\Report\%s\%sReportGenerator', $type, $period);
    $obj   = app($class);
    $obj->setStartDate($start);
    $obj->setEndDate($end);
    return $obj;
}
```

期间判定规则：

| 起止月份差 | 期间名 | 类名后缀 |
|-----------|--------|---------|
| ≤ 1 | Month | MonthReportGenerator |
| 2–12 | Year | YearReportGenerator |
| > 12 | MultiYear | MultiYearReportGenerator |

### 2.3 各类型 × 期间的 Generator 矩阵

| | Month | Year | MultiYear |
|---|---|---|---|
| **Standard** | [Standard\Month](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Standard/MonthReportGenerator.php) | [Standard\Year](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Standard/YearReportGenerator.php) | [Standard\MultiYear](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Standard/MultiYearReportGenerator.php) |
| **Audit** | [Audit\Month](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php) | [Audit\Year](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/YearReportGenerator.php) | [Audit\MultiYear](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MultiYearReportGenerator.php) |
| **Budget** | [Budget\Month](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Budget/MonthReportGenerator.php) | [Budget\Year](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Budget/YearReportGenerator.php) | [Budget\MultiYear](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Budget/MultiYearReportGenerator.php) |
| **Category** | [Category\Month](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Category/MonthReportGenerator.php) | [Category\Year](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Category/YearReportGenerator.php) | [Category\MultiYear](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Category/MultiYearReportGenerator.php) |
| **Tag** | [Tag\Month](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Tag/MonthReportGenerator.php) | [Tag\Year](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Tag/YearReportGenerator.php) | [Tag\MultiYear](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Tag/MultiYearReportGenerator.php) |
| **Account** | [Account\Month](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Account/MonthReportGenerator.php) | [Account\Year](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Account/YearReportGenerator.php) | [Account\MultiYear](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Account/MultiYearReportGenerator.php) |

---

## 三、Generator 的两种日期透传模式

所有 Generator 的 `generate()` 方法将 `$this->start` / `$this->end` 透传给下游，但**透传路径分两种模式**：

### 3.1 模式 A：渲染视图 + AJAX 拉数据（Standard / Tag / Account / Budget / Category）

以 [Standard\MonthReportGenerator::generate()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Standard/MonthReportGenerator.php#L52-L70) 为代表：

```php
public function generate(): string
{
    $accountIds = implode(',', $this->accounts->pluck('id')->toArray());
    $reportType = 'default';
    return view('reports.default.month', ['accountIds' => $accountIds, 'reportType' => $reportType])
        ->with('start', $this->start)
        ->with('end', $this->end)
        ->render();
}
```

Generator 只负责**渲染骨架 HTML**，日期通过视图变量 `$start` / `$end` 传给 Twig/Blade。

视图模板 [reports/default/month.twig](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/resources/views/reports/default/month.twig#L153-L173) 中，日期被编入 JavaScript 变量，再 AJAX 调用 report-data 端点：

```twig
<script type="text/javascript" nonce="{{ JS_NONCE }}">
    var startDate = '{{ start.format('Ymd') }}';
    var endDate = '{{ end.format('Ymd') }}';
    var accountIds = '{{ accountIds }}';

    var accountReportUrl  = '{{ route('report-data.account.general',  [accountIds, start.format('Ymd'), end.format('Ymd')]) }}';
    var categoryReportUrl = '{{ route('report-data.category.operations', [accountIds, start.format('Ymd'), end.format('Ymd')]) }}';
    var budgetReportUrl   = '{{ route('report-data.budget.general',  [accountIds, start.format('Ymd'), end.format('Ymd')]) }}';
    var balanceReportUrl   = '{{ route('report-data.balance.general',  [accountIds, start.format('Ymd'), end.format('Ymd')]) }}';
    var incomeReportUrl    = '{{ route('report-data.operations.income',  [accountIds, start.format('Ymd'), end.format('Ymd')]) }}';
    var expenseReportUrl  = '{{ route('report-data.operations.expenses', [accountIds, start.format('Ymd'), end.format('Ymd')]) }}';
    var incExpReportUrl   = '{{ route('report-data.operations.operations', [accountIds, start.format('Ymd'), end.format('Ymd')]) }}';
    var billReportUrl     = '{{ route('report-data.bills.overview',  [accountIds, start.format('Ymd'), end.format('Ymd')]) }}';
</script>
```

> **关键代码事实**：日期以 `Ymd` 格式（如 `20260701`）被嵌入 URL，再次经过 Date Binder 解析为 Carbon。

### 3.2 模式 B：Generator 直接调用 GroupCollector（Audit）

[Audit\MonthReportGenerator](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php#L132-L153) 不走 AJAX，直接在 `generate()` 中调用 GroupCollector：

```php
public function getAuditReport(Account $account, Carbon $date): array
{
    $collector = app(GroupCollectorInterface::class);
    $collector
        ->setAccounts(new Collection()->push($account))
        ->setRange($this->start, $this->end)
        ->withAccountInformation()
        ->withBudgetInformation()
        ->withCategoryInformation()
        ->withBillInformation()
        ->withNotes();
    $journals = $collector->getExtractedJournals();
    // ...
}
```

这是日期最短路径：`Carbon $this->start/$this->end → GroupCollector::setRange()`。

---

## 四、report-data 子控制器：AJAX → GroupCollector

### 4.1 report-data 路由组

[web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/routes/web.php) 定义了多组 report-data 路由，统一格式：

```
report-data/{section}/{action}/{accountList}/.../{start_date}/{end_date}
```

如：
- `report-data/budget/general/{accountList}/{start_date}/{end_date}` → `BudgetController@general`
- `report-data/category/operations/{accountList}/{start_date}/{end_date}` → `CategoryController@operations`
- `report-data/balance/general/{accountList}/{start_date}/{end_date}` → `BalanceController@general`
- `report-data/operations/income/{accountList}/{start_date}/{end_date}` → `OperationsController@income`
- `report-data/double/operations/{accountList}/{doubleList}/{start_date}/{end_date}` → `DoubleController@operations`

**`start_date` / `end_date` 再次经过 Date Binder 解析**——即使从模板中传来的是 `Ymd` 格式的数字字符串，Binder 也会将它们转换为 Carbon 对象。

### 4.2 report-data 子控制器的两种数据获取路径

**路径 1：子控制器直接调 GroupCollector**

如 [BalanceController](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BalanceController.php#L89-L97)：

```php
$collector = app(GroupCollectorInterface::class);
$journals  = $collector
    ->setRange($start, $end)
    ->setSourceAccounts($accounts)
    ->setTypes([TransactionTypeEnum::WITHDRAWAL->value])
    ->setBudget($budget)
    ->getExtractedJournals();
```

**路径 2：子控制器调 Support\Report 下的 ReportGenerator，由 Generator 调 Repository**

如 [BudgetController::general()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BudgetController.php#L256-L270)：

```php
$generator = app(BudgetReportGenerator::class);
$generator->setUser(auth()->user());
$generator->setAccounts($accounts);
$generator->setStart($start);
$generator->setEnd($end);
$generator->general();
$report = $generator->getReport();
```

[BudgetReportGenerator](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Report/Budget/BudgetReportGenerator.php) 内部调 [OperationsRepository::listExpenses()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Repositories/Tag/OperationsRepository.php#L48-L53)：

```php
public function listExpenses(Carbon $start, Carbon $end, ...): array
{
    $collector = app(GroupCollectorInterface::class);
    $collector->setUser($this->user)->setRange($start, $end)->setTypes([TransactionTypeEnum::WITHDRAWAL->value]);
    // ...
}
```

Repository 内部最终也是 `GroupCollector::setRange($start, $end)`。

### 4.3 GroupCollector → SQL

[TimeCollection::setRange()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Helpers/Collector/Extensions/TimeCollection.php#L570-L585) 是 SQL 过滤的终端：

```php
public function setRange(?Carbon $start, ?Carbon $end): GroupCollectorInterface
{
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

---

## 五、完整链路图

```
┌─────────────────────────── 头部链路 ───────────────────────────┐
│                                                                 │
│  Twig: route('reports.report.default',                          │
│        [accountList, 'currentFiscalYearStart',                  │
│         'currentFiscalYearEnd'])                                 │
│    ↓                                                            │
│  URL: /reports/default/1,2/currentFiscalYearStart/             │
│       currentFiscalYearEnd                                      │
│    ↓                                                            │
│  Binder 中间件: start_date='currentFiscalYearStart'             │
│    ↓  bindables.php: 'start_date' => Date::class                │
│    ↓  Date::routeBinder('currentFiscalYearStart')                │
│    ↓  FiscalHelper->startOfFiscalYear(today())                  │
│    ↓  → Carbon('2025-07-01')                                    │
│    ↓                                                            │
│  ReportController::defaultReport($accounts, Carbon, Carbon)     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────── 中间层 ─────────────────────────────┐
│                                                                 │
│  ReportGeneratorFactory::reportGenerator('Standard', $s, $e)   │
│    ↓ diffInMonths 判定: Month / Year / MultiYear               │
│    ↓ app('FireflyIII\Generator\Report\Standard\Month...')       │
│    ↓ $obj->setStartDate($start) / setEndDate($end)             │
│    ↓                                                            │
│  Generator::generate() → view('reports.default.month')          │
│    ↓  ->with('start', $this->start)                             │
│    ↓  ->with('end', $this->end)                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────── 尾部链路 ───────────────────────────┐
│                                                                 │
│  模式 A (Standard/Tag/Account/Budget/Category):                 │
│    Twig 模板 → JS 变量 (Ymd 格式)                               │
│    → AJAX: report-data/budget/general/{ids}/{Ymd}/{Ymd}        │
│    → Binder 再次将 Ymd 字符串解析为 Carbon                      │
│    → Report\BudgetController::general($accounts, Carbon, Carbon)│
│    → BudgetReportGenerator → OperationsRepository               │
│    → GroupCollector::setRange($start, $end)                     │
│                                                                 │
│  模式 B (Audit):                                                │
│    Generator::generate() → getAuditReport()                    │
│    → GroupCollector::setRange($this->start, $this->end)        │
│                                                                 │
│    ↓↓↓ 最终归宿 ↓↓↓                                            │
│                                                                 │
│  TimeCollection::setRange()                                     │
│    → WHERE transaction_journals.date >= 'YYYY-MM-DD 00:00:00'  │
│    → AND transaction_journals.date <= 'YYYY-MM-DD 23:59:59'    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、关键代码索引

| 环节 | 文件 | 关键方法/行 |
|------|------|-----------|
| Twig 快捷链接 | [reports/index.twig](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/resources/views/reports/index.twig) | L141-L177 |
| Twig 预设日期 | [reports/index.twig](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/resources/views/reports/index.twig) | L70-L91 |
| 报表路由 | [web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/routes/web.php) | L924-L941 |
| Binder 中间件 | [Binder.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Middleware/Binder.php) | `handle()` L60 |
| bindables 配置 | [bindables.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/config/bindables.php) | L100-L103 |
| Date 魔术词 | [Date.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Binder/Date.php) | `routeBinder()` L42 |
| Factory | [ReportGeneratorFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/ReportGeneratorFactory.php) | `reportGenerator()` L39 |
| Standard Month Generator | [MonthReportGenerator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Standard/MonthReportGenerator.php) | `generate()` L52 |
| Audit Month Generator | [Audit\MonthReportGenerator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php) | `getAuditReport()` L132 |
| 报表视图（AJAX URL） | [reports/default/month.twig](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/resources/views/reports/default/month.twig) | L153-L173 |
| Balance 控制器 | [BalanceController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BalanceController.php) | L89-L97 |
| Budget 控制器 | [BudgetController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BudgetController.php) | `general()` L256 |
| BudgetReportGenerator | [BudgetReportGenerator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Report/Budget/BudgetReportGenerator.php) | `setStart()/setEnd()` L116-L124 |
| Tag OperationsRepository | [OperationsRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Repositories/Tag/OperationsRepository.php) | `listExpenses()` L48 |
| TimeCollection | [TimeCollection.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Helpers/Collector/Extensions/TimeCollection.php) | `setRange()` L570 |
| postIndex | [ReportController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/ReportController.php) | `postIndex()` L291 |
