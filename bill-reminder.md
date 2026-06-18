# Firefly III 账单提醒与预计交易的协作链路

## 概述

Firefly III 中涉及两个独立但可协作的概念：

- **Bill（周期账单 / 订阅）**：追踪预期中的周期性支出，监控是否已按时支付
- **Recurrence（循环交易）**：自动按计划创建交易的自动化机制

二者通过 `bill_id` 关联——Recurrence 在创建交易时可指定关联某个 Bill，使生成的交易自动被标记为"已支付该账单"。

---

## 一、全局调度入口

所有定时任务通过统一的 Artisan 命令启动：

```
php artisan firefly-iii:cron
```

[Cron.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Console/Commands/Tools/Cron.php#L41-L144)

该命令支持以下与本文相关的选项：

| 选项 | 作用 |
|------|------|
| `--create-recurring` | 仅执行循环交易生成任务 |
| `--send-subscription-warnings` | 仅执行账单到期提醒任务 |
| `--force` | 强制执行，忽略时间间隔限制 |
| `--date=YYYY-MM-DD` | 模拟指定日期执行 |

不加任何选项时，所有定时任务（含上述两个）都会按序执行。

---

## 二、链路一：循环交易（Recurrence）驱动预计交易生成

### 2.1 调用链路总览

```
Cron::handle()
  → Cron::recurringCronJob()
    → RecurringCronjob::fire()           # 频率控制（≥12h 间隔）
      → RecurringCronjob::fireRecurring()
        → CreateRecurringTransactions::handle()   # 核心业务 Job
          → filterRecurrences()          # 过滤无效 Recurrence
          → handleRepetitions()          # 逐个处理重复规则
            → handleOccurrences()        # 逐个处理发生日期
              → handleOccurrence()       # 当天则创建交易
          → event(TransactionGroupsRequestedReporting)  # 事后通知
```

### 2.2 各环节详解

#### 2.2.1 RecurringCronjob — 频率控制

[RecurringCronjob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/Cronjobs/RecurringCronjob.php#L38-L97)

- 读取 `FireflyConfig` 中 `last_rt_job` 配置，判断距上次执行是否超过 **43,200 秒（12 小时）**
- 12 小时内不会重复执行（除非 `--force`）
- 执行完毕后更新 `last_rt_job` 为当前时间戳

#### 2.2.2 CreateRecurringTransactions — 核心业务 Job

[CreateRecurringTransactions.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/CreateRecurringTransactions.php#L50-L464)

**步骤一：获取所有 Recurrence**

```php
$this->recurrences = $this->repository->getAll();  // 从 recurrences 表获取全部记录
```

[RecurringRepository::getAll()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/Recurring/RecurringRepository.php#L136-L144)

**步骤二：过滤 — `filterRecurrences()` → `validRecurrence()`**

[validRecurrence()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/CreateRecurringTransactions.php#L412-L464)

过滤条件（任一不满足即跳过）：

| 条件 | 说明 |
|------|------|
| `active` | Recurrence 必须处于激活状态 |
| `repetitions` 限制 | 若设置了最大执行次数，且已创建的 journal 数 ≥ 此值则跳过 |
| `repeat_until` 已过 | 若设置了终止日期且已过则跳过 |
| `first_date` 尚未到 | 若首次执行日期在未来则跳过 |
| `latest_date` = 今天 | 今天已执行过则跳过（除非 force） |

**步骤三：处理重复规则 — `handleRepetitions()`**

[handleRepetitions()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/CreateRecurringTransactions.php#L352-L378)

- 每个 Recurrence 可有多条 `RecurrenceRepetition`（重复规则）
- 调用 `repository->getOccurrencesInRange()` 计算从 `first_date` 到「今天+2天」范围内的所有发生日期
- 发生日期计算支持 5 种重复类型：`daily`、`weekly`、`monthly`、`ndom`（每月第N个星期X）、`yearly`

[getOccurrencesInRange()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/Recurring/RecurringRepository.php#L264-L291)

**步骤四：按日期创建交易 — `handleOccurrence()`**

[handleOccurrence()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/CreateRecurringTransactions.php#L261-L322)

仅当发生日期等于今天时才执行：

1. 检查当天是否已创建过 journal（`getJournalCount`）
2. 检查是否已有相同 `recurrence_id` + `recurrence_date` 的历史记录（`createdPreviously`）
3. 构建 `getTransactionData()` 数组，其中 **`bill_id` 来自 `RecurrenceTransactionMeta`**：

```php
'bill_id' => $this->repository->getBillId($transaction),
```

[getBillId()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Repositories/Recurring/RecurringRepository.php#L146-L158)

4. 调用 `groupRepository->store($array)` 创建 `TransactionGroup`
5. 更新 Recurrence 的 `latest_date` 为当天

**步骤五：事后事件通知**

```php
event(new TransactionGroupsRequestedReporting($userId, $journals));
```

[TransactionGroupsRequestedReporting](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Events/Model/TransactionGroup/TransactionGroupsRequestedReporting.php#L31-L42)

### 2.3 交易创建后的邮件通知

[MailsNewTransactionsReport](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Listeners/Model/TransactionGroup/MailsNewTransactionsReport.php#L37-L75)

- 监听 `TransactionGroupsRequestedReporting` 事件
- 检查用户偏好 `notification_transaction_creation`（默认关闭）
- 通过 `NotificationSender` 发送 `TransactionCreation` 通知

---

## 三、链路二：账单到期提醒（Bill Warning）

### 3.1 调用链路总览

```
Cron::handle()
  → Cron::subscriptionWarningCronJob()
    → BillWarningCronjob::fire()         # 频率控制（≥12h 间隔）
      → BillWarningCronjob::fireWarnings()
        → WarnAboutBills::handle()       # 核心业务 Job
          ├── 遍历所有用户 → 所有活跃 Bill
          ├── 过期检查：SubscriptionEnrichment → BillDateCalculator → needsOverdueAlert()
          │   └── event(SubscriptionsAreOverdueForPayment)
          │       └── NotifiesAboutOverdueSubscriptions
          │           └── NotificationSender → SubscriptionsOverdueReminder
          └── 到期提醒：end_date / extension_date → needsWarning()
              └── event(SubscriptionNeedsExtensionOrRenewal)
                  └── NotifiesAboutExtensionOrRenewal
                      └── NotificationSender → BillReminder
```

### 3.2 各环节详解

#### 3.2.1 BillWarningCronjob — 频率控制

[BillWarningCronjob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/Cronjobs/BillWarningCronjob.php#L38-L100)

- 读取 `FireflyConfig` 中 `last_bw_job` 配置
- 同样 12 小时内不重复执行（除非 `--force`）
- 执行完毕后更新 `last_bw_job`

#### 3.2.2 WarnAboutBills — 核心业务 Job

[WarnAboutBills.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/WarnAboutBills.php#L44-L199)

遍历所有用户的所有活跃 Bill，执行两类检查：

**检查 A：账单过期（Overdue）**

```php
$dates = $this->getDates($bill);
if ($this->needsOverdueAlert($dates)) {
    $overdue[] = ['bill' => $bill, 'dates' => $dates];
}
```

1. `getDates()` 使用 `SubscriptionEnrichment` 计算当前周期内的 `pay_dates`（应付款日）和 `paid_dates`（已付款日）
2. `needsOverdueAlert()` 判断逻辑：

```php
$count = count($pay_dates) - count($paid_dates);  // 未付款数量
$earliest = new Carbon($pay_dates[0]);
$diff = $earliest->diffInDays($this->date);
return $diff >= 6;  // 最早的应付款日距今 ≥ 6 天则判定过期
```

[needsOverdueAlert()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/WarnAboutBills.php#L158-L172)

**检查 B：到期/续期提醒（End Date / Extension Date）**

```php
if ($this->needsWarning($bill, 'end_date')) { ... }
if ($this->needsWarning($bill, 'extension_date')) { ... }
```

[needsWarning()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Jobs/WarnAboutBills.php#L174-L184)

判断逻辑：

```php
$diff = $this->getDiff($bill, $field);  // 今天与 end_date/extension_date 的天数差
$list = config('firefly.bill_reminder_periods');  // [90, 30, 14, 7, 0]
return in_array($diff, $list, true);
```

即：当到期日距今天数恰好为 **90 / 30 / 14 / 7 / 0 天** 时触发提醒。

配置来源：[firefly.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/config/firefly.php#L226)

```php
'bill_reminder_periods' => [90, 30, 14, 7, 0],
```

#### 3.2.3 日期计算引擎 — BillDateCalculator

[BillDateCalculator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/Models/BillDateCalculator.php#L32-L173)

`getPayDates()` 方法核心逻辑：

1. 从 `earliest` 到 `latest` 逐日循环
2. 每轮调用 `nextDateMatch()` 计算下一个预期付款日
3. `nextDateMatch()` 通过 `Navigation::diffInPeriods()` 计算从账单起始日到当前日期跨了多少个周期
4. 跳过 `skip` 值对应的周期（如每 2 个月付一次则 skip=1）
5. 处理月末边界情况（如 30 号的账单在 2 月的处理）
6. 结果必须晚于 `lastPaid` 日期（排除已付款的周期）

#### 3.2.4 SubscriptionEnrichment — 数据富化

[SubscriptionEnrichment.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Support/JsonApi/Enrichments/SubscriptionEnrichment.php#L46-L481)

为 Bill 对象附加元数据：

| 元数据 | 来源 | 说明 |
|--------|------|------|
| `pay_dates` | `BillDateCalculator::getPayDates()` | 当前周期内预期应付款日列表 |
| `paid_dates` | 数据库查询 `transaction_journals`（`bill_id` 匹配） | 当前周期内已付款日列表 |
| `last_paid_date` | `paid_dates` 中的最晚日期 | 最近一次付款日 |
| `nem` | `pay_dates[0]` | 下一个预期付款日（Next Expected Match） |
| `nem_diff` | `nem` 与今天的人性化差值 | 如"3 天后" |

#### 3.2.5 事件与监听器

**过期事件**：

[SubscriptionsAreOverdueForPayment](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Events/Model/Subscription/SubscriptionsAreOverdueForPayment.php#L31-L38)

- 携带 `User` 和 `$overdue` 数组（每个元素含 `bill` + `dates`）

[NotifiesAboutOverdueSubscriptions](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Listeners/Model/Subscription/NotifiesAboutOverdueSubscriptions.php#L35-L82)

- 去重：通过偏好 `bill_overdue_{id}_{hash}` 防止同一过期状态重复通知
- 检查用户偏好 `notification_bill_reminder`（默认开启）
- 发送 `SubscriptionsOverdueReminder` 通知

**到期/续期事件**：

[SubscriptionNeedsExtensionOrRenewal](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Events/Model/Subscription/SubscriptionNeedsExtensionOrRenewal.php#L31-L39)

- 携带 `Bill`、`field`（`end_date` 或 `extension_date`）、`diff`（天数差）

[NotifiesAboutExtensionOrRenewal](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Listeners/Model/Subscription/NotifiesAboutExtensionOrRenewal.php#L34-L52)

- 检查用户偏好 `notification_bill_reminder`
- 发送 `BillReminder` 通知

#### 3.2.6 通知投递

所有通知通过统一入口发送：

[NotificationSender::send()](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Notifications/NotificationSender.php#L37-L68)

- 自动识别用户语言偏好
- 支持 Mail / Slack / Pushover 渠道
- 渠道选择由 `ReturnsAvailableChannels::returnChannels()` 根据用户配置决定

**两种通知类**：

| 通知类 | 触发场景 | 渠道 |
|--------|----------|------|
| [BillReminder](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Notifications/User/BillReminder.php#L39-L120) | `end_date`/`extension_date` 临近 | Mail、Slack、Pushover |
| [SubscriptionsOverdueReminder](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Notifications/User/SubscriptionsOverdueReminder.php#L36-L126) | 账单逾期未付 | Mail、Slack、Pushover |

---

## 四、Bill 与 Recurrence 的协作关系

### 4.1 概念区分

```
┌─────────────────────────────────────────────────────────┐
│                     Bill（周期账单）                      │
│  作用：追踪预期支出是否按时支付                             │
│  关键字段：name, amount_min/max, repeat_freq, skip,       │
│           date, end_date, extension_date                  │
│  关联交易：transaction_journals.bill_id                   │
└─────────────────────────────────────────────────────────┘
                         ▲
                         │ bill_id（可选关联）
                         │
┌─────────────────────────────────────────────────────────┐
│                  Recurrence（循环交易）                    │
│  作用：自动按周期创建交易                                  │
│  关键字段：title, first_date, repeat_until, latest_date   │
│  子模型：RecurrenceRepetition（重复规则）                  │
│          RecurrenceTransaction（交易模板）                 │
│            └→ RecurrenceTransactionMeta（含 bill_id）     │
└─────────────────────────────────────────────────────────┘
```

### 4.2 协作流程

1. 用户创建一个 Bill，定义预期支出（如"月租 ¥3000，每月 1 日"）
2. 用户创建一个 Recurrence，设置与 Bill 相同的周期，**并在交易模板中指定 `bill_id`**
3. Cron 执行时：
   - **Recurrence 链路**：按计划创建交易，交易自动关联到 Bill → Bill 被标记为该周期"已付"
   - **Bill Warning 链路**：检查 Bill 的 `pay_dates` vs `paid_dates`，如果已付则不触发过期提醒

### 4.3 不使用 Recurrence 的情况

如果用户只创建 Bill 而不创建对应 Recurrence：
- Bill 仍然会计算预期付款日（`pay_dates`）
- 如果用户手动记录了关联该 Bill 的交易（`paid_dates`），过期检查会通过
- 如果没有手动记录，`needsOverdueAlert()` 会在 6 天后触发过期提醒

---

## 五、关键配置与偏好

### 5.1 系统配置

| 配置项 | 位置 | 值 | 说明 |
|--------|------|----|------|
| `bill_reminder_periods` | [config/firefly.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/config/firefly.php#L226) | `[90, 30, 14, 7, 0]` | 到期提醒触发天数 |
| `last_rt_job` | `configuration` 表 | 时间戳 | 循环交易 Job 上次执行时间 |
| `last_bw_job` | `configuration` 表 | 时间戳 | 账单提醒 Job 上次执行时间 |

### 5.2 用户偏好

| 偏好键 | 默认值 | 说明 |
|--------|--------|------|
| `notification_bill_reminder` | `true` | 是否接收账单提醒通知 |
| `notification_transaction_creation` | `false` | 是否接收循环交易创建报告 |
| `bill_overdue_{id}_{hash}` | `false` | 过期通知去重标记（已发送则设为 `true`） |

### 5.3 时间控制

| 参数 | 值 | 说明 |
|------|----|------|
| Cron 最小间隔 | 43,200 秒（12 小时） | 两次 Cron 执行间最小时间 |
| 过期判定天数 | 6 天 | 最早预期付款日距今 ≥ 6 天视为过期 |
| 发生日期搜索范围 | 今天 + 2 天 | 包含周末的缓冲 |

---

## 六、Bill 模型关键字段

[Bill.php](file:///d:/fz/0601-2/solo-dogfeeding/code/30-firefly-iii/app/Models/Bill.php#L50-L212)

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 账单名称 |
| `amount_min` / `amount_max` | string | 预期金额范围 |
| `date` | Carbon | 账单起始日期 |
| `repeat_freq` | string | 重复频率（monthly/weekly/yearly 等） |
| `skip` | int | 跳过周期数（0=每期，1=隔一期） |
| `active` | bool | 是否激活 |
| `end_date` | Carbon? | 账单终止日期 |
| `extension_date` | Carbon? | 账单续期日期 |
| `transaction_currency_id` | int | 货币 ID |

---

## 七、完整链路时序图

```
                        ┌──────────┐
                        │  Cron 命令 │
                        └────┬─────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              │              ▼
    ┌─────────────────┐     │     ┌─────────────────┐
    │ RecurringCronjob│     │     │BillWarningCronjob│
    └────────┬────────┘     │     └────────┬────────┘
             │              │              │
             ▼              │              ▼
  ┌──────────────────────┐  │  ┌──────────────────────┐
  │CreateRecurring       │  │  │WarnAboutBills        │
  │Transactions          │  │  │                      │
  │                      │  │  │  遍历 User → Bill    │
  │ 1.获取所有 Recurrence│  │  │                      │
  │ 2.过滤无效记录       │  │  │ ┌──────────────────┐ │
  │ 3.计算发生日期       │  │  │ │过期检查          │ │
  │ 4.当天则创建交易     │  │  │ │Subscription      │ │
  │   └→ bill_id 关联   │  │  │ │Enrichment        │ │
  │ 5.更新 latest_date   │  │  │ │  → BillDate      │ │
  │                      │  │  │ │    Calculator    │ │
  └──────────┬───────────┘  │  │ │  → pay_dates    │ │
             │              │  │ │  → paid_dates   │ │
             ▼              │  │ └──────┬───────────┘ │
  ┌──────────────────────┐  │  │        │ ≥6天        │
  │TransactionGroups     │  │  │        ▼              │
  │RequestedReporting    │  │  │ ┌──────────────────┐ │
  │   (Event)            │  │  │ │SubscriptionsAre  │ │
  └──────────┬───────────┘  │  │ │OverdueForPayment │ │
             │              │  │ │   (Event)        │ │
             ▼              │  │ └──────┬───────────┘ │
  ┌──────────────────────┐  │  │        ▼              │
  │MailsNewTransactions  │  │  │ ┌──────────────────┐ │
  │Report (Listener)     │  │  │ │NotifiesAbout     │ │
  │                      │  │  │ │OverdueSubs       │ │
  │ 检查偏好 → 发送通知   │  │  │ │  (Listener)      │ │
  └──────────────────────┘  │  │ └──────┬───────────┘ │
                            │  │        │              │
                            │  │ ┌──────────────────┐ │
                            │  │ │到期/续期检查      │ │
                            │  │ │end_date          │ │
                            │  │ │extension_date    │ │
                            │  │ │  diff ∈ [90,30,  │ │
                            │  │ │   14,7,0]        │ │
                            │  │ └──────┬───────────┘ │
                            │  │        ▼              │
                            │  │ ┌──────────────────┐ │
                            │  │ │SubscriptionNeeds │ │
                            │  │ │ExtensionOrRenewal│ │
                            │  │ │   (Event)        │ │
                            │  │ └──────┬───────────┘ │
                            │  │        ▼              │
                            │  │ ┌──────────────────┐ │
                            │  │ │NotifiesAbout     │ │
                            │  │ │ExtensionOrRenewal│ │
                            │  │ │  (Listener)      │ │
                            │  │ └──────┬───────────┘ │
                            │  │        │              │
                            │  └────────┼──────────────┘
                            │           ▼
                            │  ┌──────────────────────┐
                            │  │ NotificationSender    │
                            │  │  → Mail / Slack /     │
                            │  │    Pushover           │
                            │  └──────────────────────┘
```
