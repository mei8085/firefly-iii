# Firefly III 汇率换算回退逻辑与报表影响分析报告

## 一、汇率换算回退顺序详解

`ExchangeRateConverter::getRate()` 方法（`app/Support/Http/Api/ExchangeRateConverter.php:239-287`）实现了完整的四级回退机制，确保在不同情况下尽可能获取到可用汇率。

### 1.1 回退流程图

```
请求汇率 (from → to, date)
    │
    ├─→ 第一步：应用缓存查询 (Cache::get)
    │      ├─ 命中：直接返回
    │      └─ 未命中：继续
    │
    ├─→ 第二步：正向数据库查询 (from → to)
    │      ├─ 命中：存入永久缓存 + 返回
    │      └─ 未命中：继续
    │
    ├─→ 第三步：反向数据库查询 (to → from)，取倒数 1/rate
    │      ├─ 命中：存入永久缓存 + 返回
    │      └─ 未命中：继续
    │
    └─→ 第四步：EUR 桥接换算
           ├─ 获取 from → EUR 汇率 (getEuroRate)
           ├─ 获取 to → EUR 汇率 (getEuroRate)
           ├─ 两者都有效：计算 rate = (from→EUR) / (to→EUR)
           │                = (from→EUR) * (1/(to→EUR))
           ├─ 任一无效：进入兜底
           │
           └─→ 第五步：默认值兜底
                  └─ 返回 "1"（1:1 换算）
```

### 1.2 各阶段代码与逻辑分析

#### 第一级：应用缓存查询

**代码位置**：`ExchangeRateConverter.php:241-249`

```php
$key = $this->getCacheKey($from, $to, $date);  // cer-{fromId}-{toId}-{Y-m-d}
$res = Cache::get($key);
if (null !== $res) {
    return $res;
}
```

**特点**：
- 缓存键包含货币对和日期，确保不同日期的汇率独立缓存
- 使用 Laravel Cache 门面，支持多种缓存驱动（文件、Redis 等）
- 性能最优，零数据库查询

---

#### 第二级：正向数据库查询

**代码位置**：`ExchangeRateConverter.php:251-258`

```php
$rate = $this->getFromDB($from->id, $to->id, $date->format('Y-m-d'));
if (null !== $rate) {
    Cache::forever($key, $rate);  // ⚠️ 永久缓存
    return $rate;
}
```

**`getFromDB` 内部查询逻辑**（`ExchangeRateConverter.php:203-211`）：

```php
$result = $this->userGroup
    ->currencyExchangeRates()
    ->where('from_currency_id', $from)
    ->where('to_currency_id', $to)
    ->where('date', '<=', $date)  // 关键：小于等于查询
    ->orderBy('date', 'DESC')       // 取最新的一条
    ->first();
```

**重要特性**：
- 使用 `date <= $date` + `ORDER BY date DESC` 组合查询：获取**小于等于目标日期的最新汇率**
- 这意味着如果 5月20日查询 5月15日的汇率，如果那天没有，会使用 5月14日或更早的最新汇率
- 查询结果存入 `Cache::forever()` 永久缓存

---

#### 第三级：反向查询取倒数

**代码位置**：`ExchangeRateConverter.php:260-268`

```php
$rate = $this->getFromDB($to->id, $from->id, $date->format('Y-m-d'));
if (null !== $rate) {
    $rate = bcdiv('1', $rate);  // 取倒数
    Cache::forever($key, $rate);
    return $rate;
}
```

**数学原理**：
- 已知 B → A 的汇率为 R
- 则 A → B 的汇率 = 1 / R
- 例如：已知 EUR→USD = 1.08，则 USD→EUR = 1/1.08 ≈ 0.9259

**精度问题**：
- 由于浮点数精度限制，连续两次反向换算可能产生微小误差
- 例如：USD→EUR→USD 可能不等于原始值

---

#### 第四级：EUR 桥接换算

**代码位置**：`ExchangeRateConverter.php:270-286`

```php
$first  = $this->getEuroRate($from, $date);  // from → EUR
$second = $this->getEuroRate($to, $date);    // to → EUR

if (0 === bccomp('0', $first) || 0 === bccomp('0', $second)) {
    return '1';  // 任一汇率缺失，进入兜底
}

$second = bcdiv('1', $second);  // 1/(to→EUR) = EUR→to
$rate   = bcmul($first, $second);  // (from→EUR) * (EUR→to) = from→to
```

**换算公式**：

```
from → to = (from → EUR) × (EUR → to)
          = (from → EUR) ÷ (to → EUR)
```

**示例**：
- 目标：USD → JPY
- USD → EUR = 0.9259
- JPY → EUR = 0.006155
- USD → JPY = 0.9259 ÷ 0.006155 ≈ 150.43

**`getEuroRate` 的三级回退**（`ExchangeRateConverter.php:142-172`）：

1. 正向查询：currency → EUR
2. 反向查询：EUR → currency，取倒数
3. 配置文件兜底：`config('cer.rates.{code}`

```php
$backup = config(sprintf('cer.rates.%s', $currency->code));
if (null !== $backup) {
    return bcdiv('1', (string)$backup);
}
```

配置文件中的备份汇率日期为 `2025-04-15`（`config/cer.php:32`），是静态值，不会随时间更新。

---

#### 第五级：默认值兜底

**代码位置**：`ExchangeRateConverter.php:275-278`

```php
Log::warning(sprintf('There is not enough information to convert %s to %s on date %s', 
    $from->code, $to->code, $date->format('Y-m-d')));
return '1';
```

**关键行为**：
- 静默返回汇率 "1"
- 仅记录一条 warning 日志
- 不抛出异常，不中断执行
- 等价于 1:1 换算

---

## 二、缓存失效机制分析

### 2.1 缓存分层架构

Firefly III 采用三层缓存体系：

| 缓存层级 | 存储位置 | 过期策略 | 失效时机 |
|---------|---------|---------|---------|
| L1 请求内缓存 | `ExchangeRateConverter::$prepared` | 请求结束即销毁 | 每次请求结束 |
| L2 应用缓存 | Laravel Cache | **永久存储** | 全局 `Cache::clear()` |
| L3 数据库 | `currency_exchange_rates` 表 | 永久存储 | 手动删除 / 新数据覆盖 |

### 2.2 L2 缓存永久存储问题

**代码证据**：所有缓存写入均使用 `Cache::forever()`

```php
// ExchangeRateConverter.php:254
Cache::forever($key, $rate);

// ExchangeRateConverter.php:264
Cache::forever($key, $rate);

// ExchangeRateConverter.php:284
Cache::forever($key, $rate);

// CacheProperties.php:87-90
public function store($data): void
{
    Cache::forever($this->hash, $data);
}
```

**没有设置过期时间**：汇率缓存永不过期**。

### 2.3 缓存失效触发点

全局搜索 `Cache::clear()` 仅在三个位置被调用：

| 文件 | 触发场景 |
|------|---------|
| `ProcessesExchangeRates.php:43` | 汇率数据创建/更新/删除时 |
| `InstallController.php:157` | 系统安装时 |
| `DebugController.php:112` | 调试操作时 |

**核心失效逻辑**：`ProcessesExchangeRates` 监听器（`app/Listeners/Models/CurrencyExchangeRate/ProcessesExchangeRates.php:40-52`）：

```php
public function handle(CreatedCurrencyExchangeRate|DestroyedCurrencyExchangeRate|UpdatedCurrencyExchangeRate $event): void
{
    Preferences::mark();
    Cache::clear();  // ⚠️ 清除全部缓存
    // ... 重新计算相关货币金额
}
```

### 2.4 缓存一致性风险

**问题 1：** `Cache::clear()` 是**全局清除**，会清除所有缓存数据，不仅仅是汇率缓存。

**问题 2：** 只有汇率数据变更才会触发缓存清除，以下情况缓存不会自动失效：

- 汇率下载任务执行，但没有新增/更新/删除操作（例如数据已存在）
- 系统时间流逝，旧汇率数据未触发模型事件
- 手动修改数据库中的汇率数据，绕过模型事件

**问题 3：** 缓存键包含日期，但缓存永久存储，意味着：

- 2025-05-20 查询过 2025-05-15 的汇率会被永久缓存
- 即使后续补充了 2025-05-15 的汇率数据，只要缓存中已有的缓存值不会自动更新

---

## 三、历史交易日期与取汇率时间点一致性分析

### 3.1 交易保存时的汇率时间点

**关键发现**：交易金额换算使用**当前日期**的汇率，而非交易日期的汇率！

**代码证据**：`ConvertsAmountToPrimaryAmount.php:89`

```php
$newAmount = $converter->convert(
    $params->originalCurrency,
    $primaryCurrency,
    now(),  // ⚠️ 使用当前时间！
    $amount
);
```

### 3.2 影响分析

**场景 1：编辑历史交易

| 时间点 | 事件 | 汇率使用 |
|--------|------|----------|
| 2025-01-15 | 交易发生，USD/EUR = 0.88 | - |
| 2025-05-20 | 用户编辑并保存该交易 | 使用 2025-05-20 的汇率（例如 0.92） |

**结果**：1月15日发生的交易，使用 5月20日保存时用的汇率换算。

**场景 2：汇率补充后重算

| 时间点 | 事件 |
|--------|------|
| 2025-05-20 | 补充了 2025-01-15 的历史汇率 |
| 2025-05-21 | 触发 `PrimaryAmountRecalculationService 重算 | 使用 `now()`（2025-05-21）的汇率，仍不是 01-15 的汇率 |

**代码证据**：`PrimaryAmountRecalculationService.php:333-338`

```php
$account->pivot->native_current_amount = $converter->convert(
    $piggyBank->transactionCurrency,
    $currency,
    today(),  // ⚠️ 依然使用今天
    (string)$account->pivot->current_amount
);
```

### 3.3 报表查询时的汇率时间点

**报表中的余额计算**：`Steam.php:138-139`

```php
$converter = new ExchangeRateConverter();
$pcVirtualBalance = $converter->convert($currency, $primary, $date, $virtualBalance);
```

这里使用了 **查询日期** `$date` 的汇率。

### 3.4 不一致性总结

| 操作场景 | 汇率日期 | 代码位置 |
|---------|---------|---------|
| 交易保存/编辑 | `now()` 当前日期 | `ConvertsAmountToPrimaryAmount.php:89` |
| 储蓄账户余额 | `now()` 当前日期 | `PrimaryAmountRecalculationService.php:336` |
| 报表余额计算 | 查询日期 `$date` | `Steam.php:139` |
| 交易金额换算（`getRate`） | 传入的 `$date` | `ExchangeRateConverter.php:239` |

**问题**：
- 保存时用 `now()`，查询时用交易日期，导致两者不一致
- 历史交易的 `native_amount` 反映的是**保存时**的汇率，不是**交易发生时**的汇率

---

## 四、汇率缺失对报表的偏差影响

### 4.1 汇率缺失触发条件

当以下所有条件满足时，汇率缺失：

1. L1/L2 缓存未命中
2. 正向数据库无 `from → to` 汇率
3. 反向数据库无 `to → from` 汇率
4. `from → EUR` 汇率缺失（返回 '0'）
5. `to → EUR` 汇率缺失（返回 '0'）

此时返回汇率 `'1'`。

### 4.2 各类报表的偏差分析

#### 4.2.1 账户余额报表

**场景**：
- 用户主货币：EUR
- 账户 A：USD 账户，余额 10,000 USD
- 汇率缺失，返回 1

**计算过程**：
```
native_amount = 10,000 USD × 1.0 = 10,000 EUR
```

**实际情况**：假设真实汇率 0.92，真实余额应为 9,200 EUR

**偏差**：+8.7%（高估）

#### 4.2.2 收支汇总报表

**场景**：
- 月度报表，包含多笔 USD 交易
- 收入：5,000 USD（真实汇率 0.92，应为 4,600 EUR）
- 支出：3,000 USD（真实汇率 0.92，应为 2,760 EUR）
- 汇率缺失，全部使用 1

**报表显示**：
- 收入：5,000 EUR
- 支出：3,000 EUR
- 净收入：2,000 EUR

**实际应为**：
- 收入：4,600 EUR
- 支出：2,760 EUR
- 净收入：1,840 EUR

**偏差**：
- 收入偏差：+8.7%
- 净收入偏差：+8.7%

#### 4.2.3 预算执行报表

**场景**：
- 月度预算：1,000 EUR
- 实际消费：1,200 USD（真实汇率 0.92，实际 1,104 EUR）
- 汇率缺失

**报表显示**：消费 1,200 EUR，预算使用率 120%

**实际应为**：消费 1,104 EUR，预算使用率 110.4%

**偏差**：预算使用率高估约 9.6 个百分点

#### 4.2.4 跨货币转账报表

**场景**：
- 从 USD 账户转出 1,000 USD 到 EUR 账户
- 真实汇率：1 USD = 0.92 EUR
- 汇率缺失

**报表显示**：
- USD 账户减少 1,000 USD（换算为 1,000 EUR）
- EUR 账户增加 920 EUR
- 报表显示「汇兑损失 80 EUR

**实际**：转账本身不应产生汇兑损失（直接兑换时已完成）

**偏差**：凭空出现 80 EUR 损失

### 4.3 数据污染扩散

#### 4.3.1 缓存污染

```
汇率缺失 → 返回 1 → 计算 native_amount → 存入数据库 → 被 L2 永久缓存
                                                          ↑
                                              即使后续补充汇率，缓存值不变
```

#### 4.3.2 汇总级联影响

| 层级 | 影响对象 | 偏差来源 |
|-----|---------|---------|
| 1 | 交易 `native_amount` | 汇率缺失时保存 |
| 2 | 账户余额汇总 | 多笔交易偏差累积 |
| 3 | 分类/预算汇总 | 账户余额偏差传导 |
| 4 | 净资产统计 | 全系统汇总偏差 |

### 4.4 偏差检测方法

**日志检测**：
```
grep "There is not enough information to convert" storage/logs/*.log
```

**数据校验 SQL**：
```sql
-- 检查 native_amount 与 amount 比例异常的交易
SELECT 
    t.id,
    t.amount,
    t.native_amount,
    tc.code AS currency,
    tj.date
FROM transactions t
JOIN transaction_currencies tc ON tc.id = t.transaction_currency_id
JOIN transaction_journals tj ON tj.id = t.transaction_journal_id
WHERE 
    t.transaction_currency_id != t.foreign_currency_id
    AND ABS(
        (t.native_amount / t.amount) 
        NOT BETWEEN 0.5 AND 2.0;
```

---

## 五、修复与优化建议

### 5.1 紧急修复

#### 修复 1：使用交易日期进行换算

**文件**：`app/Handlers/ExchangeRate/ConvertsAmountToPrimaryAmount.php:89`

```php
// 当前（有问题）
$newAmount = $converter->convert($from, $to, now(), $amount);

// 修复后
$newAmount = $converter->convert($from, $to, $params->date, $amount);
```

#### 修复 2：增加缓存过期时间

**文件**：`app/Support/Http/Api/ExchangeRateConverter.php`

```php
// 当前（永久）
Cache::forever($key, $rate);

// 修复后（24小时）
Cache::put($key, $rate, now()->addHours(24));
```

#### 修复 3：汇率缺失时抛出异常或标记

```php
// 当前
return '1';

// 建议
Log::error(sprintf('Exchange rate missing: %s->%s @ %s', $from->code, $to->code, $date);
// 或抛出 MissingExchangeRateException
```

### 5.2 架构优化

| 优化方向 | 具体措施 |
|---------|---------|
| 汇率监控 | 增加汇率健康检查，报表生成前验证 |
| 数据完整性 | 汇率缺失告警，通知用户数据可能不准确 |
| 缓存精度 | 按日期缓存，支持历史汇率查询 |
| 重算机制 | 汇率更新后自动触发历史交易金额 |

### 5.3 运维建议

1. **定期执行**：`php artisan cache:clear` 清除缓存
2. **监控日志**：告警 "There is not enough information"
3. **验证数据**：每月核对汇率完整性
4. **重算机制**：补充汇率后执行 `PrimaryAmountRecalculationService`

---

## 六、核心代码引用速查

| 功能 | 文件位置 | 行号 |
|-----|---------|------|
| 汇率回退主逻辑 | `ExchangeRateConverter.php` | 239-287 |
| 数据库查询 | `ExchangeRateConverter.php` | 174-234 |
| EUR桥接换算 | `ExchangeRateConverter.php` | 142-172 |
| 交易金额换算 | `ConvertsAmountToPrimaryAmount.php` | 33-103 |
| 缓存失效触发 | `ProcessesExchangeRates.php` | 40-52 |
| 报表余额计算 | `Steam.php` | 127-141 |
| 交易观察者 | `TransactionObserver.php` | 38-70 |
| 配置备份汇率 | `config/cer.php` | 34-79 |
