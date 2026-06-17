# Report AJAX 子控制器路由映射与 Generator 双模式深度分析

本文档完整梳理 Firefly III 报表系统中：
1. **report-data 模块下所有 8 个 AJAX 子控制器 + 40 个子路由**的精确映射关系
2. **Generator\Report（视图渲染骨架）** 与 **Support\Report（业务聚合计算）** 两套 Generator 体系的差异
3. **模式 A（视图骨架 + AJAX 二次取数）** 与 **模式 B（Generator 内直接调 GroupCollector）** 在 GroupCollector 调用时间点上的本质区别

---

## 一、report-data 子路由完整映射表

所有路由定义在 [web.php L942-L1200](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/routes/web.php#L942-L1200)。每个路由组都挂在 `FireflyIII\Http\Controllers\Report` 命名空间下，都用 `user-full-auth` 中间件，日期参数都约束为 `DATEFORMAT`（`[0-9]{8}` 即 YYYYMMDD）。

### 1.1 report-data/account（AccountController）

| 路由名 | URL | 控制器方法 | 参数 | 视图/返回 |
|-------|-----|-----------|-----|----------|
| `report-data.account.general` | `GET report-data/account/general/{accountList}/{start_date}/{end_date}` | [AccountController::general()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/AccountController.php#L45-L74) | accounts, start, end | `reports.partials.accounts` |

### 1.2 report-data/bill（BillController）

| 路由名 | URL | 控制器方法 | 参数 | 视图/返回 |
|-------|-----|-----------|-----|----------|
| `report-data.bills.overview` | `GET report-data/bill/overview/{accountList}/{start_date}/{end_date}` | [BillController::overview()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BillController.php#L46-L74) | accounts, start, end | `reports.partials.bills` |

### 1.3 report-data/double（DoubleController）

| 路由名 | URL | 控制器方法 | 参数 | 视图/返回 |
|-------|-----|-----------|-----|----------|
| `report-data.double.operations` | `GET report-data/double/operations/{accountList}/{doubleList}/{start_date}/{end_date}` | [DoubleController::operations()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/DoubleController.php#L170-L279) | accounts, doubles, start, end | `reports.double.partials.accounts` |
| `report-data.double.ops-asset` | `GET report-data/double/ops-asset/{accountList}/{doubleList}/{start_date}/{end_date}` | [DoubleController::operationsPerAsset()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/DoubleController.php#L284-L374) | accounts, doubles, start, end | `reports.double.partials.accounts-per-asset` |
| `report-data.double.top-expenses` | `GET report-data/double/top-expenses/{accountList}/{doubleList}/{start_date}/{end_date}` | [DoubleController::topExpenses()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/DoubleController.php#L381-L422) | accounts, doubles, start, end | `reports.double.partials.top-expenses` |
| `report-data.double.avg-expenses` | `GET report-data/double/avg-expenses/{accountList}/{doubleList}/{start_date}/{end_date}` | [DoubleController::avgExpenses()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/DoubleController.php#L70-L113) | accounts, doubles, start, end | `reports.double.partials.avg-expenses` |
| `report-data.double.top-income` | `GET report-data/double/top-income/{accountList}/{doubleList}/{start_date}/{end_date}` | [DoubleController::topIncome()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/DoubleController.php#L429-L470) | accounts, doubles, start, end | `reports.double.partials.top-income` |
| `report-data.double.avg-income` | `GET report-data/double/avg-income/{accountList}/{doubleList}/{start_date}/{end_date}` | [DoubleController::avgIncome()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/DoubleController.php#L120-L163) | accounts, doubles, start, end | `reports.double.partials.avg-income` |

### 1.4 report-data/operations（OperationsController）

| 路由名 | URL | 控制器方法 | 参数 | 视图/返回 |
|-------|-----|-----------|-----|----------|
| `report-data.operations.operations` | `GET report-data/operations/operations/{accountList}/{start_date}/{end_date}` | [OperationsController::operations()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/OperationsController.php#L135-L180) | accounts, start, end | `reports.partials.operations` |
| `report-data.operations.income` | `GET report-data/operations/income/{accountList}/{start_date}/{end_date}` | [OperationsController::income()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/OperationsController.php#L99-L126) | accounts, start, end | `reports.partials.income-expenses` |
| `report-data.operations.expenses` | `GET report-data/operations/expenses/{accountList}/{start_date}/{end_date}` | [OperationsController::expenses()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/OperationsController.php#L65-L92) | accounts, start, end | `reports.partials.income-expenses` |

### 1.5 report-data/category（CategoryController）

| 路由名 | URL | 控制器方法 | 参数 | 视图/返回 |
|-------|-----|-----------|-----|----------|
| `report-data.category.operations` | `GET report-data/category/operations/{accountList}/{start_date}/{end_date}` | [CategoryController::operations()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/CategoryController.php#L603-L634) | accounts, start, end | `reports.partials.categories` |
| `report-data.category.income` | `GET report-data/category/income/{accountList}/{start_date}/{end_date}` | [CategoryController::income()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/CategoryController.php#L531-L596) | accounts, start, end | `reports.partials.category-period` |
| `report-data.category.expenses` | `GET report-data/category/expenses/{accountList}/{start_date}/{end_date}` | [CategoryController::expenses()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/CategoryController.php#L457-L524) | accounts, start, end | `reports.partials.category-period` |
| `report-data.category.accounts` | `GET report-data/category/accounts/{accountList}/{categoryList}/{start_date}/{end_date}` | [CategoryController::accounts()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/CategoryController.php#L154-L244) | accounts, categories, start, end | `reports.category.partials.accounts` |
| `report-data.category.categories` | `GET report-data/category/categories/{accountList}/{categoryList}/{start_date}/{end_date}` | [CategoryController::categories()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/CategoryController.php#L351-L448) | accounts, categories, start, end | `reports.category.partials.categories` |
| `report-data.category.account-per-category` | `GET report-data/category/account-per-category/{accountList}/{categoryList}/{start_date}/{end_date}` | [CategoryController::accountPerCategory()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/CategoryController.php#L70-L147) | accounts, categories, start, end | `reports.category.partials.account-per-category` |
| `report-data.category.top-expenses` | `GET report-data/category/top-expenses/{accountList}/{categoryList}/{start_date}/{end_date}` | [CategoryController::topExpenses()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/CategoryController.php#L641-L682) | accounts, categories, start, end | `reports.category.partials.top-expenses` |
| `report-data.category.avg-expenses` | `GET report-data/category/avg-expenses/{accountList}/{categoryList}/{start_date}/{end_date}` | [CategoryController::avgExpenses()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/CategoryController.php#L251-L294) | accounts, categories, start, end | `reports.category.partials.avg-expenses` |
| `report-data.category.top-income` | `GET report-data/category/top-income/{accountList}/{categoryList}/{start_date}/{end_date}` | [CategoryController::topIncome()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/CategoryController.php#L689-L730) | accounts, categories, start, end | `reports.category.partials.top-income` |
| `report-data.category.avg-income` | `GET report-data/category/avg-income/{accountList}/{categoryList}/{start_date}/{end_date}` | [CategoryController::avgIncome()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/CategoryController.php#L301-L344) | accounts, categories, start, end | `reports.category.partials.avg-income` |

### 1.6 report-data/tag（TagController）

| 路由名 | URL | 控制器方法 | 参数 | 视图/返回 |
|-------|-----|-----------|-----|----------|
| `report-data.tag.accounts` | `GET report-data/tag/accounts/{accountList}/{tagList}/{start_date}/{end_date}` | [TagController::accounts()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/TagController.php#L142-L232) | accounts, tags, start, end | `reports.tag.partials.accounts` |
| `report-data.tag.tags` | `GET report-data/tag/tags/{accountList}/{tagList}/{start_date}/{end_date}` | [TagController::tags()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/TagController.php#L339-L440) | accounts, tags, start, end | `reports.tag.partials.tags` |
| `report-data.tag.account-per-tag` | `GET report-data/tag/account-per-tag/{accountList}/{tagList}/{start_date}/{end_date}` | [TagController::accountPerTag()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/TagController.php#L64-L135) | accounts, tags, start, end | `reports.tag.partials.account-per-tag` |
| `report-data.tag.top-expenses` | `GET report-data/tag/top-expenses/{accountList}/{tagList}/{start_date}/{end_date}` | [TagController::topExpenses()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/TagController.php#L447-L488) | accounts, tags, start, end | `reports.tag.partials.top-expenses` |
| `report-data.tag.avg-expenses` | `GET report-data/tag/avg-expenses/{accountList}/{tagList}/{start_date}/{end_date}` | [TagController::avgExpenses()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/TagController.php#L239-L282) | accounts, tags, start, end | `reports.tag.partials.avg-expenses` |
| `report-data.tag.top-income` | `GET report-data/tag/top-income/{accountList}/{tagList}/{start_date}/{end_date}` | [TagController::topIncome()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/TagController.php#L495-L536) | accounts, tags, start, end | `reports.tag.partials.top-income` |
| `report-data.tag.avg-income` | `GET report-data/tag/avg-income/{accountList}/{tagList}/{start_date}/{end_date}` | [TagController::avgIncome()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/TagController.php#L289-L332) | accounts, tags, start, end | `reports.tag.partials.avg-income` |

### 1.7 report-data/balance（BalanceController）

| 路由名 | URL | 控制器方法 | 参数 | 视图/返回 |
|-------|-----|-----------|-----|----------|
| `report-data.balance.general` | `GET report-data/balance/general/{accountList}/{start_date}/{end_date}` | [BalanceController::general()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BalanceController.php#L67-L149) | accounts, start, end | `reports.partials.balance` |

### 1.8 report-data/budget（BudgetController）

| 路由名 | URL | 控制器方法 | 参数 | 视图/返回 |
|-------|-----|-----------|-----|----------|
| `report-data.budget.general` | `GET report-data/budget/general/{accountList}/{start_date}/{end_date}/` | [BudgetController::general()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BudgetController.php#L256-L270) | accounts, start, end | `reports.partials.budgets` |
| `report-data.budget.period` | `GET report-data/budget/period/{accountList}/{start_date}/{end_date}` | [BudgetController::period()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BudgetController.php#L279-L347) | accounts, start, end | `reports.partials.budget-period` |
| `report-data.budget.accounts` | `GET report-data/budget/accounts/{accountList}/{budgetList}/{start_date}/{end_date}` | [BudgetController::accounts()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BudgetController.php#L91-L133) | accounts, budgets, start, end | `reports.budget.partials.accounts` |
| `report-data.budget.budgets` | `GET report-data/budget/budgets/{accountList}/{budgetList}/{start_date}/{end_date}` | [BudgetController::budgets()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BudgetController.php#L189-L247) | accounts, budgets, start, end | `reports.budget.partials.budgets` |
| `report-data.budget.account-per-budget` | `GET report-data/budget/account-per-budget/{accountList}/{budgetList}/{start_date}/{end_date}` | [BudgetController::accountPerBudget()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BudgetController.php#L71-L86) | accounts, budgets, start, end | `reports.budget.partials.account-per-budget` |
| `report-data.budget.top-expenses` | `GET report-data/budget/top-expenses/{accountList}/{budgetList}/{start_date}/{end_date}` | [BudgetController::topExpenses()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BudgetController.php#L354-L395) | accounts, budgets, start, end | `reports.budget.partials.top-expenses` |
| `report-data.budget.avg-expenses` | `GET report-data/budget/avg-expenses/{accountList}/{budgetList}/{start_date}/{end_date}` | [BudgetController::avgExpenses()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BudgetController.php#L140-L184) | accounts, budgets, start, end | `reports.budget.partials.avg-expenses` |

> **共计 40 条 report-data 子路由**，分布在 8 个子控制器中。

---

## 二、两套 Generator 体系的本质差异

Firefly III 报表系统存在 **命名相近但职责完全不同** 的两套 Generator：

### 2.1 Generator\Report（视图骨架渲染器）

位于 `app/Generator/Report/`，实现 [ReportGeneratorInterface](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/ReportGeneratorInterface.php)：

```php
interface ReportGeneratorInterface
{
    public function setStartDate(Carbon $start): ReportGeneratorInterface;
    public function setEndDate(Carbon $end): ReportGeneratorInterface;
    public function setAccounts(Collection $accounts): ReportGeneratorInterface;
    public function generate(): string;  // 返回渲染好的 HTML 字符串
}
```

核心方法 `generate()`：**只渲染骨架 HTML**，日期以 `Ymd` 格式嵌入 JS 变量，数据通过 AJAX 异步获取。6 个报表类型 × 3 个期间粒度 = 18 个实现类。

**示例**：[Standard\MonthReportGenerator::generate()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Standard/MonthReportGenerator.php#L52-L70)：

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

**不调 GroupCollector**。

### 2.2 Support\Report（业务数据聚合器）

位于 `app/Support/Report/`，**不实现统一接口**，每个类型独立实现。职责：接收 Carbon 日期，调用 Repository 层聚合报表数据数组。

**示例**：[BudgetReportGenerator::general()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Report/Budget/BudgetReportGenerator.php#L92-L99)：

```php
public function general(): void
{
    $this->report = ['budgets' => [], 'sums' => []];
    $this->generalBudgetReport();  // 内部调 $this->opsRepository->listExpenses()
    $this->noBudgetReport();       // 内部调 $this->nbRepository->listExpenses()
    $this->percentageReport();
}
```

**不直接调 GroupCollector，通过 Repository 间接调用**。

---

## 三、子控制器的 4 条 GroupCollector 调用路径

所有 8 个 AJAX 子控制器最终都会到达 `GroupCollector::setRange()`，但路径不同，按从控制器到 GroupCollector 的中间层数分为 **4 种路径**：

### 路径 1：控制器 → 直接 new GroupCollector（最短路径）

**适用**：BalanceController

```
BalanceController::general()
  └─ app(GroupCollectorInterface::class)
     └─ ->setRange($start, $end)  → SQL
```

代码位置：[BalanceController.php L89-L97](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BalanceController.php#L89-L97)：

```php
$collector = app(GroupCollectorInterface::class);
$journals  = $collector
    ->setRange($start, $end)
    ->setSourceAccounts($accounts)
    ->setTypes([TransactionTypeEnum::WITHDRAWAL->value])
    ->setBudget($budget)
    ->getExtractedJournals();
```

**GroupCollector 调用时间点**：控制器方法执行期间（AJAX 请求到达后）。

### 路径 2：控制器 → AccountTasker → GroupCollector

**适用**：OperationsController、AccountController

```
OperationsController::income()
  └─ $this->tasker->getIncomeReport($start, $end, $accounts)
     └─ AccountTasker::getIncomeReport()
        └─ app(GroupCollectorInterface::class)
           └─ ->setRange($start, $end) → SQL
```

代码位置：[AccountTasker.php L125-L128](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Repositories/Account/AccountTasker.php#L125-L128)：

```php
$collector = app(GroupCollectorInterface::class);
$collector->setSourceAccounts($accounts)->setRange($start, $end);
$collector->setTypes([TransactionTypeEnum::WITHDRAWAL->value, TransactionTypeEnum::TRANSFER->value]);
```

**GroupCollector 调用时间点**：AJAX 请求到达 → 控制器方法 → AccountTasker 方法执行期间。

### 路径 3：控制器 → Support\Report Generator → Repository → GroupCollector

**适用**：BudgetController::general()、BudgetController::accountPerBudget()、CategoryController::operations()

```
BudgetController::general()
  └─ app(BudgetReportGenerator::class)
     └─ ->general()
        └─ ->generalBudgetReport()
           └─ ->processBudget($budget)
              └─ $this->opsRepository->listExpenses($this->start, $this->end, ...)
                 └─ Budget\OperationsRepository::listExpenses()
                    └─ app(GroupCollectorInterface::class)
                       └─ ->setRange($start, $end) → SQL
```

代码位置：[Budget\OperationsRepository.php L48-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Repositories/Budget/OperationsRepository.php#L48-L52)：

```php
public function listExpenses(Carbon $start, Carbon $end, ?Collection $accounts = null, ?Collection $budgets = null): array
{
    $collector = app(GroupCollectorInterface::class);
    $collector->setUser($this->user)->setRange($start, $end)->setTypes([TransactionTypeEnum::WITHDRAWAL->value]);
```

**GroupCollector 调用时间点**：AJAX 请求到达 → 控制器方法 → Support\Report Generator 方法 → Repository 方法执行期间。

### 路径 4：控制器 → Repository → GroupCollector（无 Support\Report Generator）

**适用**：其余大多数方法（Budget::accounts/budgets/topExpenses/avgExpenses/period；Category 除 operations 外的所有方法；Tag 全部；Double 全部；Bill）

```
CategoryController::accounts()
  └─ $this->opsRepository->listExpenses($start, $end, $accounts, $categories)
     └─ Category\OperationsRepository::listExpenses()
        └─ app(GroupCollectorInterface::class)
           └─ ->setRange($start, $end) → SQL
```

代码位置（Tag 为例）：[Tag\OperationsRepository.php L48-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Repositories/Tag/OperationsRepository.php#L48-L52)：

```php
public function listExpenses(Carbon $start, Carbon $end, ?Collection $accounts = null, ?Collection $tags = null): array
{
    $collector = app(GroupCollectorInterface::class);
    $collector->setUser($this->user)->setRange($start, $end)->setTypes([TransactionTypeEnum::WITHDRAWAL->value]);
```

**GroupCollector 调用时间点**：AJAX 请求到达 → 控制器方法 → Repository 方法执行期间。

### 路径对比总结表

| 路径 | 层数 | 中间件 | 适用控制器/方法 | GroupCollector 调用时机 |
|-----|-----|-------|---------------|----------------------|
| 1 | 1 | 无 | BalanceController | 控制器方法内直接 |
| 2 | 2 | AccountTasker | OperationsController, AccountController | Tasker 方法内 |
| 3 | 3 | Support\Report Generator + Repository | Budget::general/accountPerBudget, Category::operations | Generator 调 Repository |
| 4 | 2 | Repository | Budget 其余, Category 其余, Tag, Double, Bill | 控制器直接调 Repository |

---

## 四、两种报表主 Generator 模式的 GroupCollector 时间点差异

报表主流程（ReportController → ReportGeneratorFactory → Generator\Report）存在两种截然不同的模式，**决定了 GroupCollector 在整个请求生命周期中被调用的时间点**：

### 模式 A：Standard / Tag / Account / Budget / Category / Double（视图骨架 + AJAX 二次取数）

**请求时序**：

```
T0 浏览器请求: GET /reports/default/1,2,3/currentFiscalYearStart/currentFiscalYearEnd
  ↓
T1 Binder 中间件解析魔术词为 Carbon
  ↓
T2 ReportController::defaultReport()
    └─ ReportGeneratorFactory::reportGenerator('Standard', Carbon, Carbon)
       └─ app(Standard\MonthReportGenerator)
          └─ generate() → view('reports.default.month')->render()
             → 返回 HTML 骨架 + 嵌入的 JS 变量 start='20250701', end='20260630'
  ↓
T3 浏览器收到 HTML，JS 变量生效
  ↓
T4 浏览器 AJAX 请求（并行 7-8 个）:
   GET /report-data/account/general/1,2,3/20250701/20260630
   GET /report-data/category/operations/1,2,3/20250701/20260630
   GET /report-data/budget/general/1,2,3/20250701/20260630
   GET /report-data/balance/general/1,2,3/20250701/20260630
   GET /report-data/operations/income/1,2,3/20250701/20260630
   GET /report-data/operations/expenses/1,2,3/20250701/20260630
   GET /report-data/bills/overview/1,2,3/20250701/20260630
  ↓
T5 每个 AJAX 请求的 Binder 将 Ymd 字符串解析为 Carbon
  ↓
T6 各子控制器按 4 条路径之一调用 GroupCollector::setRange()
  ↓
T7 SQL 执行，返回部分 HTML
  ↓
T8 浏览器将 HTML 片段插入 DOM
```

**关键点**：
- **首次请求（T0-T3）不调 GroupCollector**，只返回视图骨架
- **GroupCollector 调用发生在 T6（AJAX 请求中）**
- 日期经历了 **两次 Date Binder 解析**：
  - 第一次（T1）：魔术词 → Carbon → 传给 Generator
  - 第二次（T5）：Generator 视图中格式化为 `Ymd` 字符串 → 经 AJAX URL → Binder 再次解析为 Carbon

### 模式 B：Audit（Generator 内直接调 GroupCollector，无 AJAX）

**请求时序**：

```
T0 浏览器请求: GET /reports/audit/1,2,3/currentFiscalYearStart/currentFiscalYearEnd
  ↓
T1 Binder 中间件解析魔术词为 Carbon
  ↓
T2 ReportController::auditReport()
    └─ ReportGeneratorFactory::reportGenerator('Audit', Carbon, Carbon)
       └─ app(Audit\MonthReportGenerator)
          └─ generate()
             └─ foreach ($accounts as $account) {
                   $this->getAuditReport($account, $this->start)
                   └─ app(GroupCollectorInterface::class)
                      └─ ->setRange($this->start, $this->end) → SQL
                }
             → 返回完整 HTML（包含所有数据）
  ↓
T3 浏览器收到完整 HTML，无需 AJAX
```

代码位置：[Audit\MonthReportGenerator::getAuditReport()](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php#L132-L153)：

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
```

**关键点**：
- **首次请求（T2）直接调 GroupCollector**，Generator 生成完整 HTML
- **无 AJAX 二次请求**，浏览器一次收到完整页面
- 日期只经历 **一次 Date Binder 解析**（T1）

### 模式 A vs 模式 B：时间点差异对照表

| 维度 | 模式 A（Standard 等 6 种） | 模式 B（Audit） |
|-----|--------------------------|----------------|
| **Generator 命名空间** | `Generator\Report\Standard\*` | `Generator\Report\Audit\*` |
| **首次请求是否调 GroupCollector** | ❌ 否，只渲染骨架 | ✅ 是，Generator::generate() 内直接调 |
| **GroupCollector 调用时机** | AJAX 请求到达后（T6） | 主请求 Generator::generate() 内（T2） |
| **Date Binder 解析次数** | 2 次（魔术词→Carbon，Ymd→Carbon） | 1 次（魔术词→Carbon） |
| **HTTP 请求数** | 1 次主请求 + 7-8 次 AJAX = 8-9 次 | 1 次 |
| **日期传递路径** | Carbon → view → Ymd 字符串 → AJAX URL → Binder → Carbon → GroupCollector | Carbon → Generator 属性 → GroupCollector |
| **性能特点** | 首屏快，分块加载，骨架先出 | 首屏慢，一次返回完整数据 |
| **涉及的 Generator 文件** | Standard/Tag/Account/Budget/Category/Double × Month/Year/MultiYear = 18 个类 | Audit × Month/Year/MultiYear = 3 个类 |

---

## 五、财年日期在各路径中的保真度

所有 4 条子路径中，财年日期一旦被 Date Binder 转为 Carbon 对象后，**不再丢失财年语义**——因为 Carbon 对象携带了具体日期值（如 `2025-07-01`），而不是期间类型（如 `'1Y'`）。GroupCollector::setRange() 只需要具体日期，不需要知道是不是财年。

**保真链路验证**：

```
魔术词 currentFiscalYearStart
  → Date Binder → Carbon('2025-07-01')
  → 模式 A: Carbon → format('Ymd')='20250701' → AJAX URL → Date Binder → Carbon('2025-07-01') → GroupCollector
  → 模式 B: Carbon → Generator 属性 → GroupCollector
  → TimeCollection::setRange() → WHERE date >= '2025-07-01 00:00:00'
```

财年语义只存在于 **Date Binder 魔术词解析** 这一步。后续所有链路传递的都是具体 Carbon 日期，与财年无关。

---

## 六、关键代码索引

| 类别 | 文件 | 关键位置 |
|------|------|---------|
| 路由定义 | [web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/routes/web.php) | L942-L1200 |
| 主控制器 | [ReportController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/ReportController.php) | defaultReport() L164, auditReport() L186 |
| Generator Factory | [ReportGeneratorFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/ReportGeneratorFactory.php) | reportGenerator() L39 |
| 模式 A Generator | [Standard\MonthReportGenerator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Standard/MonthReportGenerator.php) | generate() L52 |
| 模式 B Generator | [Audit\MonthReportGenerator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Generator/Report/Audit/MonthReportGenerator.php) | getAuditReport() L132 |
| Support\Report Generator | [BudgetReportGenerator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Report/Budget/BudgetReportGenerator.php) | general() L92 |
| Support\Report Generator | [CategoryReportGenerator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Support/Report/Category/CategoryReportGenerator.php) | operations() L63 |
| 路径1: Balance 直接调 | [BalanceController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Http/Controllers/Report/BalanceController.php) | general() L89 |
| 路径2: AccountTasker | [AccountTasker.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Repositories/Account/AccountTasker.php) | getExpenseReport() L125 |
| 路径3/4: Budget Repository | [Budget\OperationsRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Repositories/Budget/OperationsRepository.php) | listExpenses() L48 |
| 路径4: Tag Repository | [Tag\OperationsRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Repositories/Tag/OperationsRepository.php) | listExpenses() L48 |
| SQL 终端 | [TimeCollection.php](file:///d:/fz/0601-2/solo-dogfeeding/code/13-firefly-iii/app/Helpers/Collector/Extensions/TimeCollection.php) | setRange() L570 |
