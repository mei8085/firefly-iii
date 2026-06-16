# Firefly III 账单系统代码分析

## 一、账单匹配规则的判定

### 1.1 匹配规则实现机制

账单的自动匹配通过 **规则引擎（Rule Engine）** 实现，而非账单模型自身的匹配逻辑。历史上账单有独立的 `match` 字段，现已迁移到规则系统中。

**关键文件：**
- [UpgradesBillsToRules.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Console/Commands/Upgrade/UpgradesBillsToRules.php) - 账单匹配规则迁移逻辑
- [LinkToBill.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/TransactionRules/Actions/LinkToBill.php) - 链接账单的规则动作
- [UpdatesRulesForChangedBill.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Listeners/Model/Bill/UpdatesRulesForChangedBill.php) - 账单变更时同步更新规则

### 1.2 匹配规则构成

每个账单对应一条或多条规则，规则由 **触发器（Triggers）** 和 **动作（Actions）** 组成：

**触发器（判断条件）：**
1. `description_contains` - 交易描述包含账单匹配关键词
2. 金额条件（二选一）：
   - `amount_exactly` - 金额精确匹配（当 amount_min == amount_max 时）
   - `amount_less` + `amount_more` - 金额在范围内（当 amount_min != amount_max 时）

**动作（执行结果）：**
- `link_to_bill` - 将交易链接到指定账单

### 1.3 匹配规则代码逻辑

在 [UpgradesBillsToRules.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Console/Commands/Upgrade/UpgradesBillsToRules.php#L99-L145) 的 `migrateBill()` 方法中，规则生成逻辑如下：

```php
$newRule = [
    'trigger'  => 'store-journal',    // 触发时机：交易存储时
    'triggers' => [
        ['type' => 'description_contains', 'value' => $match],  // 描述包含匹配词
    ],
    'actions'  => [
        ['type' => 'link_to_bill', 'value' => $bill->name],     // 链接到账单
    ],
];

// 金额条件：
if ($bill->amount_max === $bill->amount_min) {
    $newRule['triggers'][] = ['type' => 'amount_exactly', 'value' => $bill->amount_min];
} else {
    $newRule['triggers'][] = ['type' => 'amount_less', 'value' => $bill->amount_max];
    $newRule['triggers'][] = ['type' => 'amount_more', 'value' => $bill->amount_min];
}
```

### 1.4 LinkToBill 动作执行

在 [LinkToBill.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/TransactionRules/Actions/LinkToBill.php#L48-L99) 的 `actOnArray()` 方法中：

1. 根据账单名称查找账单
2. 验证交易类型必须是 **支出（withdrawal）**
3. 检查是否已链接到该账单（避免重复）
4. 更新 `transaction_journals` 表的 `bill_id` 字段
5. 记录审计日志

```php
if (null !== $bill && TransactionTypeEnum::WITHDRAWAL->value === $type) {
    DB::table('transaction_journals')
        ->where('id', '=', $journal['transaction_journal_id'])
        ->update(['bill_id' => $bill->id]);
    // 触发审计日志事件
    event(new TransactionGroupRequestsAuditLogEntry(...));
    return true;
}
```

---

## 二、下一笔到期日的计算算法

### 2.1 核心计算类

账单日期计算由专门的计算器类负责，包含两种实现：

**关键文件：**
- [BillDateCalculator.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/Models/BillDateCalculator.php) - 新版账单日期计算器
- [BillRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Repositories/Bill/BillRepository.php) - 旧版仓储中的日期计算
- [Navigation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/Navigation.php) - 导航/周期工具类

### 2.2 核心方法：getPayDates()

在 [BillDateCalculator.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/Models/BillDateCalculator.php#L43-L141) 中，`getPayDates()` 方法计算指定时间范围内的所有应付日期：

**参数：**
- `$earliest` - 最早日期
- `$latest` - 最晚日期
- `$billStart` - 账单起始日期
- `$period` - 重复周期（weekly/monthly/quarterly/yearly 等）
- `$skip` - 跳过次数（每 N 次周期付一次）
- `$lastPaid` - 上次付款日期（可选）

**算法流程：**

```
1. 初始化当前日期为 earliest（向前推1天）
2. 循环直到当前日期超过 latest：
   a. 计算下一个预期匹配日期 nextExpectedMatch
   b. 如果 nextExpectedMatch 超过 latest：
      - 如果已有结果，停止
      - 如果没有结果，至少添加一个日期
   c. 日期满足以下条件则加入结果集：
      - 日期 >= earliest
      - 日期 > lastPaid（如果有上次付款日期）
   d. 月末日期修正：如果账单日接近月末（<4天），处理小月/二月边界问题
   e. 当前日期推进到 nextExpectedMatch + 1天
3. 返回日期字符串数组
```

### 2.3 核心方法：nextDateMatch()

计算给定最早日期后的下一个账单日：

```php
protected function nextDateMatch(Carbon $earliest, Carbon $billStartDate, string $period, int $skip): Carbon
{
    // 如果最早日期在账单开始日期之前，直接返回账单开始日期
    if ($earliest->lt($billStartDate)) {
        return $billStartDate;
    }
    
    // 计算两个日期之间的周期数
    $steps = Navigation::diffInPeriods($period, $skip, $earliest, $billStartDate);
    
    // 修正：如果周期数与记录的相同，再加1（避免重复同一日期）
    if ($steps === $this->diffInMonths) {
        ++$steps;
    }
    $this->diffInMonths = $steps;
    
    // 从账单起始日期加上 steps-1 个周期（因为 addPeriod 本身加1个周期）
    $result = clone $billStartDate;
    if ($steps > 0) {
        --$steps;
        $result = Navigation::addPeriod($billStartDate, $period, $steps);
    }
    
    return $result;
}
```

### 2.4 nextExpectedMatch() - 考虑已付款的情况

在 [BillRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Repositories/Bill/BillRepository.php#L521-L559) 中，`nextExpectedMatch()` 方法会检查当前周期是否已付款：

```php
public function nextExpectedMatch(Bill $bill, Carbon $date): Carbon
{
    // 1. 先找到 >= date 的下一个账单日 start
    $start = clone $bill->date;
    while ($start < $date) {
        $start = Navigation::addPeriod($start, $bill->repeat_freq, $bill->skip);
    }
    
    // 2. 计算该周期的结束日期 end
    $end = Navigation::addPeriod($start, $bill->repeat_freq, $bill->skip);
    
    // 3. 检查这个周期内是否已有交易（已付款）
    $journalCount = $bill->transactionJournals()->before($end)->after($start)->count();
    
    // 4. 如果已付款，跳到下一个周期
    if ($journalCount > 0) {
        $start = clone $end;
        $end   = Navigation::addPeriod($start, $bill->repeat_freq, $bill->skip);
    }
    
    return $start;
}
```

### 2.5 月末边界修正

在 [BillDateCalculator.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/Models/BillDateCalculator.php#L103-L115) 中处理特殊情况：

```php
// 如果账单日距离月末不到4天（如30号），检查下月是否有这一天
if ($daysUntilEOM < 4) {
    $nextUntilEOM = Navigation::daysUntilEndOfMonth($nextExpectedMatch);
    $diffEOM      = $daysUntilEOM - $nextUntilEOM;
    if ($diffEOM > 0) {
        // 比如1月30号，2月只有28天，就往前推几天
        $nextExpectedMatch->subDays($diffEOM);
    }
}
```

### 2.6 周期增加：addPeriod()

在 [Navigation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/Navigation.php#L49-L87) 中，通过 `Periodicity` 枚举和日历计算器实现周期增加：

```
周期映射：
- daily / 1D         → 每日
- weekly / 1W / week → 每周
- monthly / 1M / month → 每月
- quarterly / 3M / quarter → 每季度
- half-year / 6M → 每半年
- yearly / 1Y / year → 每年
```

---

## 三、预计交易虚拟记录在收支预测中的呈现位置

### 3.1 核心概念说明

Firefly III 中 **没有真正的"虚拟交易"或"预计交易"数据库记录**。账单的"预计"是通过 **计算应付日期（pay_dates）与已付日期（paid_dates）的差值** 动态呈现的。

**关键数据结构：**
- `pay_dates` - 应付日期列表（计算得出）
- `paid_dates` - 已付日期列表（从交易表查询得出）
- `next_expected_match` - 下一笔预期付款日

### 3.2 数据富集层

在 [SubscriptionEnrichment.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/JsonApi/Enrichments/SubscriptionEnrichment.php) 中统一计算账单的元数据：

**富集流程：**
```
1. collectSubscriptionIds() - 收集账单ID
2. collectNotes() - 收集备注
3. collectObjectGroups() - 收集对象组
4. collectPaidDates() - 收集已付日期（从交易表查询）
5. collectPayDates() - 计算应付日期（调用 BillDateCalculator）
```

**输出的 meta 数据：**
```php
$meta = [
    'last_paid_date' => ...,     // 最近一次付款日期
    'paid_dates'     => [...],   // 已付日期数组（含交易详情）
    'pay_dates'      => [...],   // 应付日期数组
    'nem'            => ...,     // next expected match 下一笔预期日
    'nem_diff'       => ...,     // 下一笔预期日的相对描述（如"3天后"）
];
```

### 3.3 首页仪表板呈现

**位置1：汇总盒子（Summary Boxes）**

在 [BasicController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Api/V1/Controllers/Summary/BasicController.php#L515-L652) 的 `getSubscriptionInformation()` 方法中：

返回两种汇总数据：
- `bills-paid-in-{currency}` - 已付账单金额
- `bills-unpaid-in-{currency}` - 未付账单金额

计算方式：
```php
$paidAmount   = $this->billRepository->sumPaidInRange($start, $end);
$unpaidAmount = $this->billRepository->sumUnpaidInRange($start, $end);
```

**位置2：订阅卡片（Dashboard Subscriptions）**

在 [subscriptions.blade.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/resources/views/v2/partials/dashboard/subscriptions.blade.php) 中：
- 进度条显示已付/未付比例
- 账单列表显示每笔账单的状态（已付/未付）
- 未付账单显示预期金额和预期次数

**位置3：账单饼图**

在 [BillController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Http/Controllers/Chart/BillController.php#L55-L93) 的 `frontpage()` 方法中：
- 饼图展示已付 vs 未付的金额比例

### 3.4 账单列表页面呈现

在 [bills.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/resources/views/list/bills.twig) 中，每个账单显示：

- **"本期已付"列**：显示已付日期链接（绿色文字）
- **"下一笔预期"列**：显示应付日期
- 三种状态展示：
  1. 本期无预期 → 灰色文字 "not expected period"
  2. 本期有预期但未付 → 黄色警告文字
  3. 本期已付 → 绿色成功文字，显示付款交易链接

底部汇总行：
- 活跃账单总金额
- 待支付总金额（left_to_pay）
- 每周期金额汇总

### 3.5 报表页面呈现

在 [bills.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/resources/views/reports/partials/bills.twig) 的报表账单部分：

表格列：
- 账单名称
- 最小金额 / 最大金额
- 预期日期（可能多个）
- 已付交易（带链接）

如果未支付显示 "not charged"（未扣款）。

### 3.6 未付金额计算

在 [BillRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Repositories/Bill/BillRepository.php#L665-L703) 的 `sumUnpaidInRange()` 方法中：

```php
foreach ($bills as $bill) {
    $dates = $this->getPayDatesInRange($bill, $start, $end);  // 应付日期数
    $count = $bill->transactionJournals()->after($start)->before($end)->count();  // 已付笔数
    $total = $dates->count() - $count;  // 未付笔数
    
    if ($total > 0) {
        $average = bcdiv(bcadd($bill->amount_max, $bill->amount_min), '2');  // 取平均金额
        $sum = bcmul($average, (string)$total);  // 未付总金额
    }
}
```

---

## 四、被实际交易匹配后的状态切换

### 4.1 关联机制

账单与交易的关联通过 `transaction_journals` 表的 `bill_id` 外键字段实现：

- 当 `bill_id = NULL` 时：交易未关联任何账单
- 当 `bill_id = {账单ID}` 时：交易已关联到该账单

**关键模型关系：**
- [Bill.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Models/Bill.php#L148-L151) 中 `transactionJournals()` 一对多关系

```php
public function transactionJournals(): HasMany
{
    return $this->hasMany(TransactionJournal::class);
}
```

### 4.2 关联方式

交易可以通过以下方式关联到账单：

**方式1：规则自动匹配**
- 触发时机：交易存储时（store-journal）
- 执行动作：`LinkToBill` 规则动作
- 更新字段：`transaction_journals.bill_id`

**方式2：手动关联**
- 通过表单编辑交易时选择账单
- 在 [JournalUpdateService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Services/Internal/Update/JournalUpdateService.php) 中处理

**方式3：批量操作**
- 通过 [BillRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Repositories/Bill/BillRepository.php#L483-L492) 的 `linkCollectionToBill()` 方法

### 4.3 状态切换判定

账单没有显式的"状态字段"，状态是 **动态计算** 得出的：

**单个账单在某周期的状态：**

| 状态 | 判定条件 | 视觉表现 |
|------|----------|----------|
| 未预期 | `pay_dates` 为空 | 灰色文字 |
| 待支付 | `paid_dates` 数量 < `pay_dates` 数量 | 黄色/橙色警告 |
| 已支付 | `paid_dates` 数量 >= `pay_dates` 数量 | 绿色成功 |

**计算位置：**
- 在 [SubscriptionEnrichment.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/JsonApi/Enrichments/SubscriptionEnrichment.php) 中分别收集 `paid_dates` 和 `pay_dates`
- 前端视图根据数量对比判断状态

### 4.4 状态切换的影响

当一笔交易关联到账单后（bill_id 被设置），会产生以下连锁反应：

**1. 已付日期列表更新**
- `collectPaidDates()` 查询时会包含这笔交易
- `paid_dates` 数组增加一条记录

**2. 下一笔预期日重新计算**
- 在 `nextExpectedMatch()` 中，如果当前周期已有交易，会跳到下一个周期
- `next_expected_match` 指向下一个未付款周期

**3. 未付金额减少**
- `sumUnpaidInRange()` 中 `total = pay_dates_count - paid_count` 减少
- 未付总金额相应减少

**4. 首页/仪表板数据更新**
- "已付账单"金额增加
- "未付账单"金额减少
- 进度条百分比变化

### 4.5 逾期提醒判定

在 [WarnAboutBills.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Jobs/WarnAboutBills.php) 的 `needsOverdueAlert()` 方法中：

```php
private function needsOverdueAlert(array $dates): bool
{
    // 未付笔数 = 应付笔数 - 已付笔数
    $count = count($dates['pay_dates']) - count($dates['paid_dates']);
    if (0 === $count || 0 === count($dates['pay_dates'])) {
        return false;
    }
    
    // 最早的应付日期距今 >= 6 天，则视为逾期
    $earliest = new Carbon($dates['pay_dates'][0]);
    $diff = $earliest->diffInDays($this->date);
    
    return $diff >= 6;  // FIXME: 硬编码值
}
```

触发逾期事件 `SubscriptionsAreOverdueForPayment`，发送提醒通知。

### 4.6 关联的反向操作

取消关联（解绑）通过设置 `bill_id = NULL` 实现：

- [BillRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Repositories/Bill/BillRepository.php#L706-L708) 的 `unlinkAll()` 方法
- 账单删除时会清除所有关联

---

## 五、store-journal → RuleEngine → LinkToBill 主路径事件链

### 5.1 事件链总览

交易从创建到规则引擎执行再到账单关联的完整事件链如下：

```
用户提交交易
    ↓
TransactionGroupRepository::store()
    ↓
event(CreatedSingleTransactionGroup)
    ↓
ProcessesNewTransactionGroup::handle()  (异步队列监听器)
    ↓
SupportsGroupProcessingTrait::processRules()
    ↓
RuleEngineInterface::fire()  (SearchRuleEngine 实现)
    ↓
fireRule() → fireStrictRule() / fireNonStrictRule()
    ↓
执行 RuleActions → LinkToBill::actOnArray()
    ↓
UPDATE transaction_journals SET bill_id = ?
    ↓
状态切换（已付日期列表更新、下一笔预期日后移）
```

### 5.2 第一步：交易存储触发事件

在 [TransactionGroupRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php#L341-L372) 的 `store()` 方法中：

```php
public function store(array $data): TransactionGroup
{
    // 1. 通过工厂创建交易组
    $transactionGroup = $factory->create($data);
    
    // 2. 收集事件对象（journals, groups, accounts 等）
    $objects = TransactionGroupEventObjects::collectFromTransactionGroup($transactionGroup);
    
    // 3. 构造事件标志位
    $flags = new TransactionGroupEventFlags();
    $flags->applyRules      = $data['apply_rules'] ?? true;      // 是否执行规则
    $flags->fireWebhooks    = $data['fire_webhooks'] ?? true;    // 是否触发 webhook
    $flags->batchSubmission = $data['batch_submission'] ?? false;
    
    // 4. 触发事件！这是整条链的起点
    event(new CreatedSingleTransactionGroup($flags, $objects));
    
    return $transactionGroup;
}
```

**其他触发 `CreatedSingleTransactionGroup` 的位置：**
- [GroupCloneService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Services/Internal/Update/GroupCloneService.php#L59) - 交易克隆时
- [AccountServiceTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Services/Internal/Support/AccountServiceTrait.php#L479) - 账户服务创建交易时

### 5.3 第二步：监听器接收事件并处理规则

[ProcessesNewTransactionGroup.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesNewTransactionGroup.php) 实现了 `ShouldQueue` 接口，说明处理会进入**异步队列**：

```php
class ProcessesNewTransactionGroup implements ShouldQueue
{
    use SupportsGroupProcessingTrait;

    public function handle(CreatedSingleTransactionGroup|UserRequestedBatchProcessing $event): void
    {
        // 收集所有待处理的 journals
        $journals = $event->objects->transactionJournals
            ->merge($repository->getAllUncompletedJournals());

        // 如果标记允许，执行规则引擎
        if ($event->flags->applyRules) {
            $this->processRules($journals, 'store-journal');  // ← 触发类型关键字！
        }
        
        // 其他后续操作：
        if ($event->flags->recalculateCredit) { $this->recalculateCredit(...); }
        if ($event->flags->fireWebhooks)    { $this->createWebhookMessages(...); }
        $this->removePeriodStatistics($event->objects);
        $this->recalculateRunningBalance($event->objects);
        $repository->markAsCompleted($journals);
    }
}
```

### 5.4 第三步：processRules() 启动规则引擎

在 [SupportsGroupProcessingTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Listeners/Model/TransactionGroup/SupportsGroupProcessingTrait.php#L29-L66) 中：

```php
protected function processRules(Collection $set, string $type): void
{
    // 1. 从数据库获取用户的所有规则组（按触发类型过滤）
    $ruleGroupRepository = app(RuleGroupRepositoryInterface::class);
    $groups = $ruleGroupRepository->getRuleGroupsWithRules($type);  // $type = 'store-journal'

    // 2. 创建规则引擎实例
    $newRuleEngine = app(RuleEngineInterface::class);  // → SearchRuleEngine
    $newRuleEngine->setUser($user);
    $newRuleEngine->setRuleGroups($groups);

    // 3. 对每个 journal 分别执行规则（通过 operator 限定范围）
    foreach ($array as $journalId) {
        $newRuleEngine->removeOperator('journal_id');
        $newRuleEngine->addOperator(['type' => 'journal_id', 'value' => $journalId]);
        $newRuleEngine->fire();  // ← 启动规则引擎
    }
}
```

**关键点：** 规则组是**按触发类型过滤**的。`'store-journal'` 类型只匹配账单规则中 `trigger = 'store-journal'` 的规则，这正是 UpgradesBillsToRules 迁移时设置的值。

### 5.5 第四步：SearchRuleEngine::fire() 执行

在 [SearchRuleEngine.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L95-L130) 中：

```php
public function fire(): void
{
    // 两种执行模式：
    if (0 !== $this->rules->count()) {
        // 模式A：独立规则，逐条执行
        foreach ($this->rules as $rule) {
            $result = $this->fireRule($rule);
        }
    }
    if (0 !== $this->groups->count()) {
        // 模式B：规则组，按组顺序执行（可中途停止）
        foreach ($this->groups as $group) {
            $this->fireRuleGroup($group);
        }
    }
}
```

对于账单规则，使用的是**模式B**（规则组）。`fireRule()` → `fireStrictRule()` 会先通过搜索运算符找到匹配的交易，然后执行所有规则动作。

### 5.6 第五步：LinkToBill 动作执行

在 [SearchRuleEngine.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L514-L600) 的 `executeActionsOnJournal()` 中：

```php
protected function executeActionsOnJournal(Rule $rule, array $journal): bool
{
    foreach ($rule->ruleActions as $action) {
        // 通过 ActionFactory 创建动作处理器
        $actionProcessor = ActionFactory::getAction($action->action_type);
        // → 如果是 link_to_bill，返回 LinkToBill 实例
        
        $result = $actionProcessor->actOnArray($journal, $action->action_value, ...);
    }
}
```

`LinkToBill::actOnArray()` 最终执行 `UPDATE transaction_journals SET bill_id = ?`，完成关联。

---

## 六、UpdatesRulesForChangedBill 同步规则步骤

### 6.1 同步机制总览

当账单的**名称发生变更**时，所有引用该账单的规则（触发器中的关键词匹配、动作中的账单名称）都需要同步更新。

```
用户修改账单名称
    ↓
BillUpdateService::update()
    ↓
event(new UpdatedExistingBill($bill, $oldData))
    ↓
UpdatesRulesForChangedBill::handle()  (异步队列监听器)
    ↓
遍历用户所有规则
    ↓
updateRuleTriggers()  - 更新 bill_is / bill_contains / bill_starts / bill_ends 触发器
updateRuleActions()   - 更新 link_to_bill 动作
    ↓
数据库中的规则触发器和动作被 UPDATE
```

### 6.2 第一步：BillUpdateService 触发事件

在 [BillUpdateService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Services/Internal/Update/BillUpdateService.php#L114) 中：

```php
// 更新账单数据库记录后：
event(new UpdatedExistingBill($bill, $oldData));
```

`$oldData` 保存修改前的字段值（包括旧的 `name`），用于匹配需要更新的规则。

### 6.3 第二步：监听器检查名称变更

在 [UpdatesRulesForChangedBill.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Listeners/Model/Bill/UpdatesRulesForChangedBill.php#L36-L80) 中：

```php
class UpdatesRulesForChangedBill implements ShouldQueue  // 异步队列
{
    public function handle(UpdatedExistingBill $event): void
    {
        // 只有名称变了才需要处理（金额等变化不需要改规则）
        if ($event->bill->name !== $event->oldData['name']) {
            $this->updateBillTriggersAndActions($event->bill, $event->oldData);
        }
    }
```

### 6.4 第三步：遍历规则并更新

```php
private function updateBillTriggersAndActions(Bill $bill, array $oldData): void
{
    $repository = app(RuleRepositoryInterface::class);
    $repository->setUser($bill->user);
    $rules = $repository->getAll();  // 获取用户所有规则

    foreach ($rules as $rule) {
        $this->updateRule($bill, $rule, $oldData);
    }
}

private function updateRule(Bill $bill, Rule $rule, array $oldData): void
{
    $triggerTypes = ['bill_is', 'bill_ends', 'bill_starts', 'bill_contains'];

    // 1. 更新触发器：如果触发器值等于旧账单名，替换为新名称
    foreach ($rule->ruleTriggers as $trigger) {
        if (in_array($trigger->trigger_type, $triggerTypes, true) 
            && $trigger->trigger_value === $oldData['name']) {
            $trigger->trigger_value = $bill->name;  // ← 替换为新名称
            $trigger->save();
        }
    }

    // 2. 更新动作：如果 link_to_bill 动作值等于旧名称，替换为新名称
    foreach ($rule->ruleActions as $action) {
        if ('link_to_bill' === $action->action_type 
            && $action->action_value === $oldData['name']) {
            $action->action_value = $bill->name;  // ← 替换为新名称
            $action->save();
        }
    }
}
```

**注意：** 这种同步只更新规则中**名称完全匹配**的记录。如果用户手动修改了规则中的名称，这种自动同步不会生效。

---

## 七、BillDateCalculator 与 nextExpectedMatch 的调用位置总览

### 7.1 两套日期计算体系

Firefly III 中有两套日期计算体系，处于**新旧交替**阶段：

| 体系 | 类 | 特点 | 使用场景 |
|------|----|------|----------|
| 新版 | `BillDateCalculator` | 独立服务类，通过 DI 使用，月末边界处理更完善 | 数据富集层（API v1）、提醒任务 |
| 旧版 | `BillRepository::nextDateMatch()` / `nextExpectedMatch()` | 仓储层方法，有数据库缓存 | 老页面（v2 blade）、报表、图表控制器 |

### 7.2 BillDateCalculator 调用链

**调用入口：SubscriptionEnrichment 富集层**

```
API 请求账单列表 / 详情
    ↓
JsonApi V1 路由 → BillTransformer
    ↓
SubscriptionEnrichment::enrich() / enrichSingle()
    ↓
app(BillDateCalculator::class)  // 注入
    ↓
BillDateCalculator::getPayDates(...)
    ├── nextDateMatch()      // 计算单个账单日
    └── 月末边界修正逻辑
    ↓
输出 meta.pay_dates、meta.nem、meta.nem_diff
```

**具体调用位置：**
- [SubscriptionEnrichment.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/JsonApi/Enrichments/SubscriptionEnrichment.php#L48-L71) - 构造函数注入
- [SubscriptionEnrichment.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/JsonApi/Enrichments/SubscriptionEnrichment.php#L165-L200) 的 `collectPayDates()` 方法中调用 `getPayDates()`
- [WarnAboutBills.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Jobs/WarnAboutBills.php#L117-L132) 的 `getDates()` 方法中通过 `SubscriptionEnrichment` 间接调用

### 7.3 BillRepository::nextExpectedMatch 调用链

**旧仓储方法被以下页面/控制器调用：**

| 调用方 | 文件 | 用途 |
|--------|------|------|
| 首页汇总 API | [BasicController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Api/V1/Controllers/Summary/BasicController.php#L515-L523) | 计算 bills-paid-in / bills-unpaid-in 金额 |
| 账单图表控制器 | [BillController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Http/Controllers/Chart/BillController.php#L68-L69) | 生成首页饼图数据 |
| 账单列表控制器 | [IndexController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Http/Controllers/Bill/IndexController.php) | 计算本期应付款、下一笔预期日 |
| 账单详情控制器 | [ShowController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Http/Controllers/Bill/ShowController.php) | 单账单详情展示 |
| 报表助手 | [ReportHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Helpers/Report/ReportHelper.php#L63) | 报表中预期日期列 |
| 搜索运算符 | [OperatorQuerySearch.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/Search/OperatorQuerySearch.php#L99) | 账单相关搜索条件 |
| 交易流水工厂 | [TransactionJournalFactory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Factory/TransactionJournalFactory.php#L93) | 创建流水时验证账单 |
| 流水更新服务 | [JournalUpdateService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Services/Internal/Update/JournalUpdateService.php#L105) | 更新流水时查找账单 |

### 7.4 核心方法调用关系

```
sumUnpaidInRange($start, $end)
    └── getPayDatesInRange($bill, $start, $end)
            └── nextDateMatch($bill, $currentStart)  // 循环累加周期

sumPaidInRange($start, $end)
    └── 直接 SQL 查询 transaction_journals 表

nextExpectedMatch($bill, $date)
    ├── 先用 nextDateMatch 找到 >= date 的账单日
    └── 再查 transaction_journals 是否已付，已付则跳到下一周期

SubscriptionEnrichment::collectPayDates()
    └── BillDateCalculator::getPayDates()
            └── nextDateMatch($earliest, $billStart, $period, $skip)
```

---

## 八、WarnAboutBills 调度入口与任务链

### 8.1 调度入口三层结构

WarnAboutBills 任务通过三个独立的入口触发，都汇聚到同一个 Job：

```
┌─────────────────────────────────────────────────────────────┐
│  调度入口（三层）                                            │
├─────────────────────────────────────────────────────────────┤
│  ① API 入口：/api/v1/cron                                    │
│     CronController::cron()                                   │
│     └── CronRunner::billWarningCronJob()                     │
│                                                             │
│  ② CLI 入口：php artisan firefly-iii:cron                    │
│     Cron::handle() (--send-subscription-warnings 参数)        │
│     └── billWarningCronJob()                                 │
│                                                             │
│  ③ HTTP 入口（兼容旧版）：/cron                               │
│     其他 Http Controller 使用 CronRunner trait               │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
        BillWarningCronjob::fire()
                       ↓
        WarnAboutBills::handle()  (Job 主体)
```

### 8.2 第一层：CronController (API)

在 [CronController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Api/V1/Controllers/System/CronController.php#L45-L61) 中：

```php
public function cron(CronRequest $request): JsonResponse
{
    $config = $request->getAll();
    $return = [];
    
    // 五个 cron 任务依次执行：
    $return['recurring_transactions'] = $this->runRecurring(...);
    $return['auto_budgets']           = $this->runAutoBudget(...);
    $return['exchange_rates']         = $this->exchangeRatesCronJob(...);
    $return['bill_notifications']     = $this->billWarningCronJob(...);  // ← 我们关心的
    $return['webhooks']               = $this->webhookCronJob(...);
    
    return response()->api($return);
}
```

### 8.3 第一层：Cron Artisan 命令

在 [Cron.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Console/Commands/Tools/Cron.php#L58-L140) 中：

```php
// 签名定义：
protected $signature = 'firefly-iii:cron
    {--F|force : 强制执行}
    {--date= : 指定运行日期}
    {--send-subscription-warnings : 只运行订阅提醒}  // ← 可单独触发
    ...
';

// 无参数时执行所有任务
$doAll = !$this->option('download-cer') 
      && !$this->option('send-subscription-warnings')
      && ...;

if ($doAll || $this->option('send-subscription-warnings')) {
    $this->billWarningCronJob($force, $date);  // ← 与 API 调用同一个方法
}
```

### 8.4 第二层：BillWarningCronjob（节流控制）

在 [BillWarningCronjob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/Cronjobs/BillWarningCronjob.php#L43-L100) 中主要做**频率节流**：

```php
public function fire(): void
{
    // 1. 读取上次运行时间（firefly_configurations 表）
    $config = FireflyConfig::get('last_bw_job', 0);
    $lastTime = (int) $config->data;
    $diff = now()->timestamp - $lastTime;
    
    // 2. 43_200 秒 = 12 小时内不重复运行（除非 --force）
    if ($lastTime > 0 && $diff <= 43_200 && !$this->force) {
        $this->jobFired = false;
        $this->message = 'It has been %s since the cron-job has fired...';
        return;  // 直接跳过
    }
    
    // 3. 真正调用 Job
    $this->fireWarnings();
    
    // 4. 记录本次运行时间戳
    FireflyConfig::set('last_bw_job', (int) $this->date->format('U'));
}
```

**设计意图：** 防止用户频繁调用 API 导致重复提醒。12小时内只会实际执行一次。

### 8.5 节流阈值的由来与各 Cronjob 对比

节流阈值定义在基类 [AbstractCronjob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/Cronjobs/AbstractCronjob.php#L38) 中：

```php
public int $timeBetweenRuns = 43_200;  // 12 小时 = 12 * 3600
```

各 Cronjob 的节流配置对比：

| Cronjob | 配置键名 | 节流阈值 | 说明 |
|---------|----------|----------|------|
| BillWarningCronjob | `last_bw_job` | 43_200 秒（12h） | 账单提醒 |
| RecurringCronjob | `last_rt_job` | 43_200 秒（12h） | 定期交易生成 |
| AutoBudgetCronjob | `last_ab_job` | 43_200 秒（12h） | 自动预算 |
| ExchangeRatesCronjob | `last_cer_job` | 43_200 秒（12h） | 汇率下载 |
| WebhookCronjob | `last_webhook_job` | 600 秒（10min） | Webhook 发送（频率更高） |

**注意：** 虽然基类定义了 `$timeBetweenRuns`，但每个具体 Cronjob 是**各自硬编码**实现节流逻辑的，没有复用基类变量（除 Webhook 外都是 43200）。

### 8.6 Laravel Schedule 与外部 Cron 的说明

**Firefly III 不使用 Laravel 内置的 Schedule 调度器。**

代码中**没有** `app/Console/Kernel.php` 的 `schedule()` 方法注册，也没有在任何 ServiceProvider 中使用 `Schedule` facade。

**实际调度方式：** 依赖**外部系统级 cron** 触发，有两种调用入口：

```
系统 cron（Linux crontab / Docker cron）
    │
    ├── 调用 CLI：  php artisan firefly-iii:cron
    │               └── Cron::handle()
    │
    └── 调用 HTTP： GET /api/v1/cron?token=xxx
                    └── CronController::cron()
```

**推荐的 cron 表达式（官方文档）：** 每 5 分钟运行一次

```
*/5 * * * * cd /path-to-firefly && php artisan firefly-iii:cron >> /dev/null 2>&1
```

配合内部 12 小时节流机制，效果是：系统每 5 分钟触发一次检查，但账单提醒任务实际每 12 小时最多执行一次。

### 8.7 队列执行情况

[WarnAboutBills.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Jobs/WarnAboutBills.php#L44) 类实现了 `ShouldQueue` 接口：

```php
class WarnAboutBills implements ShouldQueue
{
    use Dispatchable;
    use InteractsWithQueue;
    use Queueable;
    use SerializesModels;
    // 没有自定义 $connection / $queue / $tries / $backoff
}
```

**队列配置：**
- **连接（connection）**：使用默认连接 `QUEUE_CONNECTION`（默认 `sync`，可配 database/redis/beanstalkd 等）
- **队列（queue）**：使用默认队列名 `default`
- **重试策略**：未显式配置 `$tries`，依赖连接的 `retry_after`（database/redis 默认为 90 秒）
- **同步模式**：当 `QUEUE_CONNECTION=sync` 时（默认），Job 会**同步执行**，不进入队列

> 这意味着在默认配置下，虽然 WarnAboutBills 实现了 ShouldQueue，但实际是同步执行的。只有配置了外部队列驱动（如 redis）才会真正异步。

### 8.5 第三层：WarnAboutBills Job 主体

在 [WarnAboutBills.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Jobs/WarnAboutBills.php#L77-L103) 的 `handle()` 方法中：

```php
public function handle(): void
{
    // 遍历所有用户
    foreach (User::all() as $user) {
        $bills = $user->bills()->where('active', true)->get();
        $overdue = [];

        foreach ($bills as $bill) {
            // 1. 通过富集层获取本周期的应付/已付日期
            $dates = $this->getDates($bill);
            
            // 2. 检查是否逾期（最早应付日距今 >= 6天 且 未付）
            if ($this->needsOverdueAlert($dates)) {
                $overdue[] = ['bill' => $bill, 'dates' => $dates];
            }
            
            // 3. 检查到期/续期提醒（非逾期，提前通知）
            if ($this->hasDateFields($bill)) {
                if ($this->needsWarning($bill, 'end_date')) {
                    $this->sendWarning($bill, 'end_date');      // → SubscriptionNeedsExtensionOrRenewal
                }
                if ($this->needsWarning($bill, 'extension_date')) {
                    $this->sendWarning($bill, 'extension_date');
                }
            }
        }
        
        // 4. 发送逾期提醒（每个用户最多一次批量通知）
        $this->sendOverdueAlerts($user, $overdue);  // → SubscriptionsAreOverdueForPayment
    }
}
```

### 8.6 两种提醒类型

| 类型 | 触发条件 | 对应事件 | 适用场景 |
|------|----------|----------|----------|
| **到期提醒** | `end_date` / `extension_date` 距今差在 `bill_reminder_periods` 配置列表中 | `SubscriptionNeedsExtensionOrRenewal` | 订阅即将到期、需要续期时（如 Netflix 年付到期前7天） |
| **逾期提醒** | 未付笔数 > 0 且 最早应付日距今 >= 6 天 | `SubscriptionsAreOverdueForPayment` | 账单超过6天未付款 |

---

## 九、SubscriptionsAreOverdueForPayment Listener 与通知通道

### 9.1 事件触发链

```
WarnAboutBills::handle() 检测到逾期账单
    ↓
sendOverdueAlerts($user, $overdue)
    ↓
event(new SubscriptionsAreOverdueForPayment($user, $overdue))
    ↓
NotifiesAboutOverdueSubscriptions::handle()  (监听器，异步队列)
    ├── 去重检查（Preferences）
    ├── 用户偏好检查（notification_bill_reminder）
    └── NotificationSender::send($user, new SubscriptionsOverdueReminder(...))
        ↓
    NotificationFacade::send()  (Laravel 通知系统)
        ↓
    via() 方法返回可用通道
        ├── mail       → toMail()       → emails.subscriptions-overdue-warning
        ├── pushover   → toPushover()   → Pushover 推送
        └── slack      → toSlack()      → Slack Webhook
```

### 9.2 事件定义

在 [SubscriptionsAreOverdueForPayment.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Events/Model/Subscription/SubscriptionsAreOverdueForPayment.php#L31-L38) 中：

```php
class SubscriptionsAreOverdueForPayment extends Event
{
    use SerializesModels;

    public function __construct(
        public User $user,       // 接收提醒的用户
        public array $overdue    // 逾期账单数组：[['bill' => Bill, 'dates' => ['pay_dates'=>[], 'paid_dates'=>[]]], ...]
    ) {}
}
```

### 9.3 监听器：去重与偏好检查

在 [NotifiesAboutOverdueSubscriptions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Listeners/Model/Subscription/NotifiesAboutOverdueSubscriptions.php#L35-L82) 中：

```php
class NotifiesAboutOverdueSubscriptions implements ShouldQueue  // 异步队列
{
    public function handle(SubscriptionsAreOverdueForPayment $event): void
    {
        $overdue = $event->overdue;
        $user = $event->user;
        $toBeWarned = [];

        // 1. 去重检查：防止同一笔逾期被重复提醒
        foreach ($overdue as $item) {
            $bill = $item['bill'];
            // 生成唯一 key：bill_id + 应付日期哈希的前10位
            // 只要应付日期变了（进入下一周期），就认为是新的逾期需要提醒
            $key = sprintf(
                'bill_overdue_%s_%s',
                $bill->id,
                substr(hash('sha256', json_encode($item['dates']['pay_dates'])), 0, 10)
            );
            
            $pref = Preferences::getForUser($bill->user, $key, false);
            if (true === $pref->data) {
                continue;  // 已提醒过，跳过
            }
            $toBeWarned[] = $item;
        }

        // 2. 用户偏好检查：是否关闭了账单提醒
        $sendNotification = Preferences::getForUser($user, 'notification_bill_reminder', true)->data;
        if (false === $sendNotification || 0 === count($toBeWarned)) {
            return;
        }

        // 3. 标记已提醒（写 preferences）
        foreach ($toBeWarned as $item) {
            $key = ...;  // 同样的 key 生成逻辑
            Preferences::setForUser($bill->user, $key, true);
        }

        // 4. 发送通知
        NotificationSender::send($user, new SubscriptionsOverdueReminder($toBeWarned));
    }
}
```

**去重机制的关键设计：** 使用 `bill_id + pay_dates 哈希` 作为 key。如果用户在同一周期内多次运行 cron，不会重复收到提醒；但到了下一个周期（pay_dates 变化），会生成新的 key 触发新提醒。

### 9.4 通知发送器：NotificationSender

在 [NotificationSender.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Notifications/NotificationSender.php#L36-L68) 中：

```php
class NotificationSender
{
    public static function send(OwnerNotifiable|User $user, Notification $notification): void
    {
        // 获取用户语言偏好
        $lang = Preferences::getForUser($user, 'language', config('firefly.default_language'))->data;

        try {
            // Laravel 原生通知发送器，按 via() 返回的通道分发
            NotificationFacade::locale($lang)->send($user, $notification);
        } catch (ClientException $e) {
            Log::error('[a] Error sending notification: ' . $e->getMessage());
        } 
        // 邮件配置错误处理（Bcc/RFC 2822）...
    }
}
```

### 9.5 通知类与可用通道

#### SubscriptionsOverdueReminder (逾期提醒)

在 [SubscriptionsOverdueReminder.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Notifications/User/SubscriptionsOverdueReminder.php#L36-L126) 中：

```php
class SubscriptionsOverdueReminder extends Notification
{
    use Queueable;

    // 通道选择（动态）
    public function via(User $notifiable): array
    {
        // 根据用户偏好返回可用通道：mail, pushover, slack, ntfy(注释中)
        return ReturnsAvailableChannels::returnChannels('user', $notifiable);
    }

    // 邮件通道
    public function toMail(User $notifiable): MailMessage
    {
        return new MailMessage()
            ->markdown('emails.subscriptions-overdue-warning', [...])
            ->subject($this->getSubject());  // 单条/多条逾期的不同标题
    }

    // Pushover 通道
    public function toPushover(User $notifiable): PushoverMessage
    {
        return PushoverMessage::create(trans('email.bill_warning_please_action'))
            ->title($this->getSubject());
    }

    // Slack 通道
    public function toSlack(User $notifiable): SlackMessage
    {
        return (new SlackMessage())
            ->warning()
            ->attachment(...)  // 附加"查看账单列表"链接
            ->content($this->getSubject());
    }
}
```

#### BillReminder (到期/续期提醒)

在 [BillReminder.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Notifications/User/BillReminder.php#L39-L120) 中用于 `SubscriptionNeedsExtensionOrRenewal` 事件，通道与逾期提醒相同。

### 9.6 可用通道对比

| 通道 | 对应方法 | 依赖包 | 说明 |
|------|----------|--------|------|
| **mail** | `toMail()` | Laravel 内置 | 使用 `emails.subscriptions-overdue-warning` markdown 模板，格式最完整 |
| **pushover** | `toPushover()` | `laravel-notification-channels/pushover` | 移动端推送，只显示标题和简单文本 |
| **slack** | `toSlack()` | `laravel/slack-notification-channel` | Slack 频道消息，带附件链接 |
| **ntfy** | `toNtfy()` | (注释中) | 开源推送服务，代码已写但被注释 |

通道的实际启用由 `ReturnsAvailableChannels::returnChannels()` 根据用户数据库中的通知配置动态返回。

---

## 附：核心文件索引

| 文件 | 作用 |
|------|------|
| **账单核心模型与计算** | |
| [Bill.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Models/Bill.php) | 账单模型，定义字段和关系 |
| [BillDateCalculator.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/Models/BillDateCalculator.php) | 新版账单日期计算器（富集层使用） |
| [BillRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Repositories/Bill/BillRepository.php) | 旧版账单仓储（含 nextDateMatch/nextExpectedMatch） |
| [Navigation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/Navigation.php) | 周期导航工具（addPeriod/diffInPeriods） |
| [SubscriptionEnrichment.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/JsonApi/Enrichments/SubscriptionEnrichment.php) | 账单数据富集层（pay_dates/paid_dates 计算） |
| [BillTransformer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Transformers/BillTransformer.php) | 账单数据转换器 |
| **规则引擎（store-journal → LinkToBill）** | |
| [TransactionGroupRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php#L341-L372) | 交易存储入口，触发 CreatedSingleTransactionGroup 事件 |
| [ProcessesNewTransactionGroup.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesNewTransactionGroup.php) | 新交易监听器（异步队列） |
| [SupportsGroupProcessingTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Listeners/Model/TransactionGroup/SupportsGroupProcessingTrait.php#L29-L66) | processRules() 启动规则引擎 |
| [SearchRuleEngine.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php) | 规则引擎实现，执行触发器+动作 |
| [LinkToBill.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/TransactionRules/Actions/LinkToBill.php) | link_to_bill 规则动作（设置 bill_id） |
| [UpgradesBillsToRules.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Console/Commands/Upgrade/UpgradesBillsToRules.php) | 账单规则迁移（理解匹配规则构成的关键） |
| **账单变更同步规则** | |
| [BillUpdateService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Services/Internal/Update/BillUpdateService.php#L114) | 账单更新服务，触发 UpdatedExistingBill 事件 |
| [UpdatesRulesForChangedBill.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Listeners/Model/Bill/UpdatesRulesForChangedBill.php) | 账单名称变更时同步更新规则触发器和动作 |
| **Cron 调度入口** | |
| [CronController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Api/V1/Controllers/System/CronController.php) | API 调度入口：/api/v1/cron |
| [Cron.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Console/Commands/Tools/Cron.php) | Artisan 命令入口：php artisan firefly-iii:cron |
| [CronRunner.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/HttpControllers/CronRunner.php) | 通用 Cron 运行 Trait，各入口共用 |
| [BillWarningCronjob.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/Cronjobs/BillWarningCronjob.php) | 账单提醒 Cronjob（12小时节流控制） |
| [WarnAboutBills.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Jobs/WarnAboutBills.php) | 账单提醒 Job 主体（检测逾期+到期，发送事件） |
| **逾期事件与通知通道** | |
| [SubscriptionsAreOverdueForPayment.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Events/Model/Subscription/SubscriptionsAreOverdueForPayment.php) | 账单逾期事件定义 |
| [NotifiesAboutOverdueSubscriptions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Listeners/Model/Subscription/NotifiesAboutOverdueSubscriptions.php) | 逾期监听器（去重+偏好检查+发送通知） |
| [NotificationSender.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Notifications/NotificationSender.php) | 统一通知发送器（错误处理+语言设置） |
| [SubscriptionsOverdueReminder.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Notifications/User/SubscriptionsOverdueReminder.php) | 逾期提醒通知类（mail/pushover/slack 通道） |
| [BillReminder.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Notifications/User/BillReminder.php) | 到期/续期提醒通知类（相同通道） |
| **收支预测呈现** | |
| [BasicController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Api/V1/Controllers/Summary/BasicController.php#L515-L523) | 首页汇总 API（bills-paid-in / bills-unpaid-in） |
| [BillController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Http/Controllers/Chart/BillController.php#L68-L69) | 账单饼图控制器 |
| [IndexController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Http/Controllers/Bill/IndexController.php) | 账单列表页控制器 |
| [ReportHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Helpers/Report/ReportHelper.php#L63) | 报表中预期日期列生成 |
