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

## 附：核心文件索引

| 文件 | 作用 |
|------|------|
| [Bill.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Models/Bill.php) | 账单模型，定义字段和关系 |
| [BillDateCalculator.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/Models/BillDateCalculator.php) | 账单日期计算器 |
| [BillRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Repositories/Bill/BillRepository.php) | 账单仓储，业务逻辑 |
| [SubscriptionEnrichment.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/JsonApi/Enrichments/SubscriptionEnrichment.php) | 账单数据富集 |
| [LinkToBill.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/TransactionRules/Actions/LinkToBill.php) | 链接账单规则动作 |
| [WarnAboutBills.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Jobs/WarnAboutBills.php) | 账单提醒任务 |
| [Navigation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Support/Navigation.php) | 周期导航工具 |
| [UpgradesBillsToRules.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Console/Commands/Upgrade/UpgradesBillsToRules.php) | 账单规则迁移（理解匹配规则的关键） |
| [BillTransformer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/99-firefly-iii/app/Transformers/BillTransformer.php) | 账单数据转换器 |
