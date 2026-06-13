# Firefly III 调度器时间语义与两层 date 分离机制源码分析

---

## 一、Carbon 3.x 中 `new Carbon(null|空串|now)` 的 isNow fast-path

### 1.1 Carbon 3.11.4 构造器源码

项目使用 Carbon `3.11.4`（见 [composer.lock#L3677-L3678](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/composer.lock#L3677-L3678)），其构造器定义在 `Carbon\Traits\Creator` Trait 中。

**核心构造逻辑**（摘自 Carbon 3.x `Creator.php`）：

```php
public function __construct(
    DateTimeInterface|WeekDay|Month|string|int|float|null $time = null,
    DateTimeZone|string|int|null $timezone = null,
) {
    $this->initLocalFactory();

    // ① 类型预处理：Month → 字符串, WeekDay → 字符串, DateTimeInterface → 格式化字符串
    if ($time instanceof Month)        { $time = $time->name.' 1'; }
    elseif ($time instanceof WeekDay)  { $time = $time->name; }
    elseif ($time instanceof DateTimeInterface) {
        $time = $this->constructTimezoneFromDateTime($time, $timezone)
                     ->format('Y-m-d H:i:s.u');
    }

    // ② 时间戳前缀处理
    if (is_string($time) && str_starts_with($time, '@')) {
        $time = static::createFromTimestampUTC(substr($time, 1))
                       ->format('Y-m-d\TH:i:s.uP');
    } elseif (is_numeric($time) && (!is_string($time) || !preg_match('/^\d{1,14}$/', $time))) {
        $time = static::createFromTimestampUTC($time)
                       ->format('Y-m-d\TH:i:s.uP');
    }

    // ③ isNow fast-path 判定
    $isNow = in_array($time, [null, '', 'now'], true);

    // ④ 时区安全创建
    $timezone = static::safeCreateDateTimeZone($timezone) ?? null;

    // ⑤ TestNow / Clock 注入分支（仅 isNow 或相对关键词时进入）
    if (
        ($this->clock || (
            method_exists(static::class, 'hasTestNow') &&
            method_exists(static::class, 'getTestNow') &&
            static::hasTestNow()
        )) &&
        ($isNow || static::hasRelativeKeywords($time))
    ) {
        $this->mockConstructorParameters($time, $timezone);
    }

    // ⑥ 最终 DateTime 构造
    try {
        parent::__construct($time ?? 'now', $timezone);
    } catch (Exception $exception) {
        throw new InvalidFormatException($exception->getMessage(), 0, $exception);
    }

    $this->constructedObjectId = spl_object_hash($this);
    self::setLastErrors(parent::getLastErrors());
}
```

### 1.2 isNow fast-path 的统一转换机制

**关键行**：

```php
$isNow = in_array($time, [null, '', 'now'], true);
```

这意味着三种输入统一被视为「now」语义：

| 输入 | `$time` 值 | `$isNow` | 最终传给 `parent::__construct` 的值 |
|------|-----------|----------|--------------------------------------|
| `new Carbon(null)` | `null` | `true` | `'now'`（`$time ?? 'now'`） |
| `new Carbon('')` | `''` | `true` | `''`（DateTime 将 `''` 等同于 `'now'`） |
| `new Carbon('now')` | `'now'` | `true` | `'now'` |

**核心结论**：`null`、`''`、`'now'` 三者在 `$isNow` 判定上完全等价。它们统一通过 fast-path 进入步骤 ⑤（TestNow/Clock 注入），最终在步骤 ⑥ 中 `parent::__construct($time ?? 'now', $timezone)` 调用 `DateTime::__construct('now', $tz)`，生成当前时刻实例。

### 1.3 `new Carbon()` vs `Carbon::now()` vs `Carbon::today()` 的区别

```php
// Carbon 3.x Creator.php 中的静态工厂方法：

public static function now(DateTimeZone|string|int|null $timezone = null): static
{
    return new static(null, $timezone);   // → 构造器 $time=null → isNow=true → 当前时刻
}

public static function today(DateTimeZone|string|int|null $timezone = null): static
{
    return static::rawParse('today', $timezone);  // → 构造器 $time='today' → isNow=false → 零点
}
```

| 方法 | 构造器 `$time` 参数 | `$isNow` | 结果时间 | 时分秒 |
|------|-------------------|----------|---------|--------|
| `new Carbon()` | `null` | `true` | 当前时刻 | **保留实际时分秒** |
| `Carbon::now()` | `null` | `true` | 当前时刻 | **保留实际时分秒** |
| `Carbon::today()` | `'today'` | `false` | 今天 | **00:00:00** |

**关键差异**：`new Carbon()` / `Carbon::now()` 保留时分秒；`Carbon::today()` 重置为零点。

---

## 二、`Cron::handle` 无 `--date` 参数时 `last_rt_job` 的实际写入分支

### 2.1 完整执行路径追踪

#### 步骤 1：`Cron::handle()` 解析 `--date` 参数

代码位置：[Cron.php#L67-L73](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Console/Commands/Tools/Cron.php#L67-L73)

```php
$date = null;                     // ← 初始化为 null

try {
    $date = new Carbon($this->option('date'));  // ← --date 无值时 $this->option('date') = null
} catch (InvalidArgumentException $e) {
    $this->friendlyError(sprintf('"%s" is not a valid date', $this->option('date')));
}
```

**当 CLI 不传 `--date` 时**：
- `$this->option('date')` 返回 `null`
- `new Carbon(null)` 进入 isNow fast-path → 生成**当前时刻**（含时分秒）
- `$date` 被赋值为一个**当前时刻的 Carbon 实例**

**关键发现**：与通常假设的「无 `--date` 时 `$date = null`」不同，实际 `$date` 是一个**非 null 的 Carbon 实例**，只是它的值是 `now()`。

#### 步骤 2：`recurringCronJob()` 判断是否调用 `setDate`

代码位置：[Cron.php#L211-L231](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Console/Commands/Tools/Cron.php#L211-L231)

```php
private function recurringCronJob(bool $force, ?Carbon $date): void
{
    $recurring = new RecurringCronjob();        // ← 构造器: $this->date = today() = 零点
    $recurring->setForce($force);

    // set date in cron job:
    if ($date instanceof Carbon) {              // ← $date 是 Carbon 实例，始终为 true！
        $recurring->setDate($date);             // ← 覆盖构造器中的 today()
    }

    $recurring->fire();
}
```

**当 CLI 不传 `--date` 时**：
- `$date` 是 `new Carbon(null)` = 当前时刻（非 null）
- `$date instanceof Carbon` 为 **true**
- **始终调用 `setDate($date)`**，覆盖构造器中的 `today()`

#### 步骤 3：`AbstractCronjob::setDate()` 不做 `startOfDay`

代码位置：[AbstractCronjob.php#L52-L56](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Support/Cronjobs/AbstractCronjob.php#L52-L56)

```php
final public function setDate(Carbon $date): void
{
    $newDate    = clone $date;     // ← 直接 clone，不调用 startOfDay()
    $this->date = $newDate;
}
```

**结果**：`$this->date` = 当前时刻（含时分秒），**不是零点**。

#### 步骤 4：`RecurringCronjob::fireRecurring()` 写入 `last_rt_job`

代码位置：[RecurringCronjob.php#L94](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Support/Cronjobs/RecurringCronjob.php#L94)

```php
FireflyConfig::set('last_rt_job', (int) $this->date->format('U'));
```

**`$this->date->format('U')`** 输出 Unix 时间戳，**包含时分秒**。

### 2.2 实际写入分支总结

| 场景 | `$date` 变量 | 是否调用 `setDate` | `$this->date` 值 | `last_rt_job` 时间戳 |
|------|-------------|-------------------|-----------------|---------------------|
| CLI 无 `--date` | `new Carbon(null)` = **当前时刻** | ✅ 始终调用 | 当前时刻（含时分秒） | **实际运行时刻** |
| CLI `--date=2024-06-15` | `new Carbon('2024-06-15')` = 指定日期零点 | ✅ 始终调用 | 指定日期零点 | 指定日期零点 |
| CLI `--date` 传非法值 | `null`（catch 后未赋值） | ❌ 不调用 | `today()` 零点 | 当天零点 |
| API 无 date 参数 | `today()` 零点 | ✅ 始终调用 | 当天零点 | 当天零点 |
| API 有 date 参数 | 解析后的 Carbon | ✅ 始终调用 | 解析后的日期 | 解析后的时间 |

**最重要的发现**：

**CLI 无 `--date` 时，`last_rt_job` 写入的是实际运行时刻（含时分秒），而非零点！**

这是因为 `new Carbon(null)` 通过 isNow fast-path 生成了**当前时刻**，且 `$date instanceof Carbon` 始终为 true，导致 `setDate` 始终被调用，覆盖了构造器中的 `today()`。

### 2.3 与 12 小时冷却比较的语义影响

```php
// RecurringCronjob.php#L48-L57
$lastTime = (int) FireflyConfig::get('last_rt_job', 0)->data;
$diff = now(config('app.timezone'))->getTimestamp() - $lastTime;
if ($lastTime > 0 && $diff <= 43_200) { ... }
```

| 写入场景 | `last_rt_job` 含义 | `$diff` 计算 | 语义 |
|---------|-------------------|-------------|------|
| CLI 无 `--date`（**实际路径**） | 实际运行时刻（如 14:30:00） | `now() - 实际运行时刻` | **精确时刻差**，语义一致 |
| API 无 date | 当天零点 | `now() - 当天零点` | 时刻差，但含义模糊 |
| CLI 非法 `--date` | 当天零点 | `now() - 当天零点` | 时刻差，但含义模糊 |

**纠正之前的分析**：在正常的 CLI 调用（无 `--date`）下，`last_rt_job` 存储的是**实际运行时刻**，12 小时冷却的 `$diff` 是**两次实际运行时刻之间的精确时间差**，语义是一致的。

之前的分析错误地认为 CLI 无 `--date` 时 `$date = null` 且不调用 `setDate`，实际上 `new Carbon(null)` 通过 isNow fast-path 生成了当前时刻的 Carbon 实例，`setDate` 始终被调用。

---

## 三、cronjob 层 `date` 与 Job 层 `date` 的两层分离闭环

### 3.1 两层 `date` 的数据流

```
                    Cronjob 层                              Job 层
                (RecurringCronjob)                   (CreateRecurringTransactions)

  $this->date ─────────────────────────→ $date (构造器参数)
  (含时分秒或零点)                           │
                                            ↓
  写入 last_rt_job ◄─── $this->date     $this->date (二次 startOfDay)
  (cronjob 层的 date)                      (Job 层的 date，强制零点)
                                            │
                                            ↓
                                      handleRepetitions()
                                      handleOccurrence()
                                      $date->ne($this->date) 比较
```

### 3.2 RecurringCronjob 层：使用 cronjob 的 `$this->date`

[RecurringCronjob.php#L82-L95](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Support/Cronjobs/RecurringCronjob.php#L82-L95)

```php
private function fireRecurring(): void
{
    Log::info(sprintf('Will now fire recurring cron job task for date "%s".', 
        $this->date->format('Y-m-d H:i:s')));   // ← 可能含时分秒

    $job = new CreateRecurringTransactions($this->date);  // ← 传递 cronjob 层的 date
    $job->setForce($this->force);
    $job->handle();

    // ...

    FireflyConfig::set('last_rt_job', (int) $this->date->format('U'));  // ← 使用 cronjob 层的 date
    Log::info(sprintf('Marked the last time this job has run as "%s" (%d)', 
        $this->date->format('Y-m-d H:i:s'), (int) $this->date->format('U')));
}
```

**关键**：`last_rt_job` 的写入使用的是 `$this->date`（cronjob 层），**不受 Job 层 `startOfDay` 的影响**。

### 3.3 CreateRecurringTransactions 层：二次 `startOfDay` 作用于 Job 的 `$this->date`

[CreateRecurringTransactions.php#L71-L93](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Jobs/CreateRecurringTransactions.php#L71-L93)

```php
public function __construct(?Carbon $date)
{
    // 分支 A：无论 $date 是否为 null，先创建一个今天的零点实例
    $newDate     = new Carbon();      // ← isNow fast-path → 当前时刻
    $newDate->startOfDay();           // ← 强制零点
    $this->date  = $newDate;          // ← Job 层 date = 今天零点

    // 分支 B：如果 $date 不为 null，覆盖为零点化的 $date
    if ($date instanceof Carbon) {
        $newDate     = clone $date;   // ← clone cronjob 层的 date（可能含时分秒）
        $newDate->startOfDay();       // ← 强制零点 ← 关键操作
        $this->date  = $newDate;      // ← Job 层 date = cronjob date 的零点版本
    }

    Log::debug(sprintf('Created new CreateRecurringTransactions("%s")', 
        $this->date->format('Y-m-d')));  // ← 只输出日期，验证是零点
}
```

**二次 `startOfDay` 的作用**：

| cronjob 层 `$this->date` | Job 构造器 `$date` 参数 | `clone $date` | `startOfDay()` | Job 层 `$this->date` |
|--------------------------|------------------------|---------------|---------------|---------------------|
| 2024-06-13 14:30:00 | 2024-06-13 14:30:00 | 2024-06-13 14:30:00 | 2024-06-13 00:00:00 | **2024-06-13 00:00:00** |
| 2024-06-13 00:00:00 | 2024-06-13 00:00:00 | 2024-06-13 00:00:00 | 2024-06-13 00:00:00 | **2024-06-13 00:00:00** |
| 2024-06-15 00:00:00 | 2024-06-15 00:00:00 | 2024-06-15 00:00:00 | 2024-06-15 00:00:00 | **2024-06-15 00:00:00** |

**无论 cronjob 层的 `date` 是否含时分秒，Job 层的 `date` 始终是零点。**

### 3.4 闭环机制：两层 date 的职责分离

```
                         cronjob 层 $this->date                  Job 层 $this->date
                         ─────────────────────                   ────────────────────
    来源                  AbstractCronjob 构造器                  CreateRecurringTransactions 构造器
                         + setDate() 覆盖                        + startOfDay() 修正

    时间精度              可能含时分秒                            始终零点 00:00:00

    用途                  ① 写入 last_rt_job                     ① handleOccurrence 日期比较
                         ② 冷却判断的参照基准                     ② handleRepetitions 日期范围计算
                         ③ 传递给 Job 构造器                      ③ getTransactionData 交易日期
                                                                ④ recurrence_date Meta 字段

    影响范围              仅在 RecurringCronjob 内部               仅在 CreateRecurringTransactions 内部
```

**闭环逻辑**：

1. **cronjob 层的 `$this->date`** 负责「何时执行」的判断：
   - `fire()` 中与 `last_rt_job` 比较，决定是否触发
   - `fireRecurring()` 中写入 `last_rt_job`，记录执行时刻
   - 传递给 Job 构造器，作为日期的「初始来源」

2. **Job 层的 `$this->date`** 负责「执行哪天」的判断：
   - `handleOccurrence` 中 `$date->ne($this->date)` 只匹配当天零点
   - 所有业务逻辑（日期范围计算、交易创建）都基于零点
   - `startOfDay()` 确保了即使 cronjob 层传来含时分秒的 Carbon，Job 内部也只关心「日期」而非「时刻」

3. **两层之间是单向数据流**：cronjob → Job（通过构造器参数），Job 不会反向影响 cronjob 的 `$this->date`。

### 3.5 完整执行时序图

```
CLI: php artisan firefly-iii:cron（无 --date）
│
├─ Cron::handle()
│   ├─ $date = new Carbon(null)           ← isNow fast-path → 当前时刻 14:30:00
│   └─ $this->recurringCronJob(false, $date)
│
├─ recurringCronJob(false, 2024-06-13 14:30:00)
│   ├─ $recurring = new RecurringCronjob()
│   │   └─ __construct(): $this->date = today() = 2024-06-13 00:00:00
│   ├─ $recurring->setDate(2024-06-13 14:30:00)
│   │   └─ $this->date = clone 14:30:00 = 2024-06-13 14:30:00  ← 覆盖零点
│   └─ $recurring->fire()
│
├─ RecurringCronjob::fire()
│   ├─ $lastTime = last_rt_job (上次的时间戳)
│   ├─ $diff = now().timestamp - $lastTime
│   ├─ if ($diff <= 43200) → 冷却中，return
│   └─ if ($diff > 43200) → 继续
│
├─ RecurringCronjob::fireRecurring()
│   ├─ $job = new CreateRecurringTransactions(2024-06-13 14:30:00)
│   │   ├─ 分支 A: $this->date = new Carbon()→startOfDay() = 2024-06-13 00:00:00
│   │   └─ 分支 B: $this->date = clone 14:30:00 → startOfDay() = 2024-06-13 00:00:00  ← 关键！
│   ├─ $job->handle()
│   │   └─ ... 所有业务逻辑使用 Job 层 $this->date = 2024-06-13 00:00:00 ...
│   │
│   └─ FireflyConfig::set('last_rt_job', $this->date->format('U'))
│       └─ $this->date = 2024-06-13 14:30:00  ← cronjob 层，含时分秒！
│       └─ 写入 last_rt_job = 1718272200 (14:30:00 的时间戳)
```

**闭环验证**：

- 下次 CLI 调用时，`now().timestamp - last_rt_job` = `now() - 14:30:00`
- 如果下次调用在 12 小时内（如 22:00），`$diff = 7.5h < 12h`，冷却生效 ✓
- 如果下次调用在 12 小时后（如次日 08:00），`$diff = 17.5h > 12h`，可执行 ✓
- Job 内部业务逻辑始终基于零点，不受时分秒影响 ✓

### 3.6 设计意图与实际效果

| 设计点 | 意图 | 实际效果 |
|--------|------|---------|
| cronjob 层 `date` 含时分秒 | 精确记录执行时刻 | 12h 冷却判断精确到秒级 |
| Job 层 `startOfDay` | 业务逻辑只关心日期 | 交易创建、日期比较都基于零点，逻辑正确 |
| 两层分离 | 调度关注「何时执行」，业务关注「执行哪天」 | 各司其职，互不干扰 |

**潜在问题**：如果 cronjob 层的 `$this->date` 跨天（如 23:59:59 执行，`startOfDay` 后变成当天零点），而 `last_rt_job` 记录的是 23:59:59，那么次日 00:01 的 `$diff` 仅 2 秒 × 60 = 约 2 分钟，远小于 12 小时，不会触发。这实际上是**正确行为**——因为 Job 层已经处理了当天（23:59:59 的 startOfDay = 当天零点）的交易，次日不需要再处理。

---

## 四、与 API 入口的对比

### 4.1 API 入口的 `date` 处理

[CronRequest.php#L49-L64](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Api/V1/Requests/System/CronRequest.php#L49-L64)

```php
public function getAll(): array
{
    $data = ['force' => false, 'date' => today(config('app.timezone'))];  // ← 默认 today() = 零点
    if ($this->has('force')) { $data['force'] = $this->boolean('force'); }
    if ($this->has('date'))  { $data['date'] = $this->getCarbonDate('date'); }
    if (!$data['date'] instanceof Carbon) { $data['date'] = today(config('app.timezone')); }
    return $data;
}
```

[CronRunner.php#L103-L122](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Support/Http/Controllers/CronRunner.php#L103-L122)

```php
protected function runRecurring(bool $force, Carbon $date): array
{
    $recurring = app(RecurringCronjob::class);
    $recurring->setForce($force);
    $recurring->setDate($date);   // ← 始终调用，$date = today() = 零点
    $recurring->fire();
    // ...
}
```

### 4.2 CLI 与 API 的关键差异

| 维度 | CLI 入口 | API 入口 |
|------|---------|---------|
| 无 date 参数时 | `new Carbon(null)` → 当前时刻（含时分秒） | `today()` → 零点 |
| `setDate` 调用 | 条件：`$date instanceof Carbon`（**始终为 true**） | **始终调用** |
| `last_rt_job` 默认写入值 | **实际运行时刻**（含时分秒） | **当天零点** |
| 冷却语义 | 两次精确时刻之间的差值 | 当前时刻与零点之间的差值 |

**本质区别**：CLI 使用 `new Carbon(null)` 生成当前时刻，API 使用 `today()` 生成零点。导致两者在相同条件下写入的 `last_rt_job` 时间戳不同，冷却判断的语义也不同。

---

## 五、关键文件索引

| 文件 | 分析要点 |
|------|---------|
| [Carbon Creator.php](https://github.com/CarbonPHP/carbon/blob/e890471a3494740f7d9326d72ce6a8c559ffee60/src/Carbon/Traits/Creator.php) | `__construct` 中 `$isNow = in_array($time, [null, '', 'now'], true)` 的 fast-path |
| [Cron.php#L67-L73](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Console/Commands/Tools/Cron.php#L67-L73) | `new Carbon(null)` 生成当前时刻，非 null |
| [Cron.php#L211-L231](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Console/Commands/Tools/Cron.php#L211-L231) | `$date instanceof Carbon` 始终为 true，`setDate` 始终被调用 |
| [AbstractCronjob.php#L45-L56](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Support/Cronjobs/AbstractCronjob.php#L45-L56) | 构造器 `today()` 零点，`setDate` 不做 `startOfDay` |
| [RecurringCronjob.php#L80-L97](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Support/Cronjobs/RecurringCronjob.php#L80-L97) | 使用 cronjob 层 `$this->date` 写入 `last_rt_job` |
| [CreateRecurringTransactions.php#L71-L93](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Jobs/CreateRecurringTransactions.php#L71-L93) | 二次 `startOfDay` 仅作用于 Job 层 `$this->date` |
| [CronRequest.php#L49-L64](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Api/V1/Requests/System/CronRequest.php#L49-L64) | API 默认 `today()` = 零点，与 CLI 的 `new Carbon(null)` 不同 |
| [CronRunner.php#L103-L122](file:///d:/fz/0601-1/solo-dogfeeding/code/52-firefly-iii/app/Support/Http/Controllers/CronRunner.php#L103-L122) | API 始终调用 `setDate` |
