# Firefly III 汇率换算回退逻辑与报表影响分析报告

## 一、汇率换算回退顺序详解

`ExchangeRateConverter::getRate()` 方法（`app/Support/Http/Api/ExchangeRateConverter.php:239-287`）实现了完整的五级回退机制，确保在不同情况下尽可能获取到可用汇率。

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
           │                └─→ 存入永久缓存 + 返回
           ├─ 任一无效：进入兜底
           │
           └─→ 第五步：默认值兜底
                  └─ 返回 "1"（1:1 换算，⚠️ 不写入缓存）
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
- 缓存键格式：`cer-{fromId}-{toId}-{Y-m-d}`，包含货币对和日期
- 使用 Laravel Cache 门面，支持多种缓存驱动（文件、Redis 等）
- 性能最优，零数据库查询

---

#### 第二级：正向数据库查询

**代码位置**：`ExchangeRateConverter.php:251-258`

```php
$rate = $this->getFromDB($from->id, $to->id, $date->format('Y-m-d'));
if (null !== $rate) {
    Cache::forever($key, $rate);  // ✅ 写入永久缓存
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
    Cache::forever($key, $rate);  // ✅ 写入永久缓存
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
Cache::forever($key, $rate);  // ✅ 写入永久缓存
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

1. 正向查询：currency → EUR（内部调用 `getFromDB`，命中则写入 `CacheProperties` 缓存）
2. 反向查询：EUR → currency，取倒数（同样可能写入缓存）
3. 配置文件兜底：`config('cer.rates.{code}')`

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
return '1';  // ⚠️ 重要：此处直接 return，没有 Cache::forever 调用
```

**⚠️ 关键修正**：此分支**不会写入缓存**！

代码证据：在 `return '1'` 之前没有任何 `Cache::forever()` 或 `Cache::put()` 调用。

**行为总结**：
- 静默返回汇率 "1"
- 仅记录一条 warning 日志
- 不抛出异常，不中断执行
- 等价于 1:1 换算
- ❌ 不写入应用缓存（下次请求仍会走完整回退链路）

---

## 二、缓存失效机制分析

### 2.1 缓存分层架构

Firefly III 采用三层缓存体系：

| 缓存层级 | 存储位置 | 过期策略 | 失效时机 |
|---------|---------|---------|---------|
| L1 请求内缓存 | `ExchangeRateConverter::$prepared` | 请求结束即销毁 | 每次请求结束 |
| L2 应用缓存 | Laravel Cache (Cache::forever) | **永久存储** | 全局 `Cache::clear()` |
| L3 数据库 | `currency_exchange_rates` 表 | 永久存储 | 手动删除 / 新数据覆盖 |

### 2.2 CacheProperties 缓存键影响因素

**代码证据**：`CacheProperties.php:43-51`

```php
public function __construct()
{
    $this->properties = new Collection();
    if (auth()->check()) {
        $this->addProperty(auth()->user()->id);           // 因素1：用户ID
        $this->addProperty(Preferences::lastActivity());  // 因素2：最后活动时间
        $this->addProperty(Steam::anonymous());           // 因素3：匿名模式
    }
}
```

#### 因素 1：用户 ID（`auth()->user()->id`）

- 不同用户的缓存完全隔离
- 确保多租户环境下数据安全

#### 因素 2：最后活动时间（`Preferences::lastActivity()`）

**代码证据**：`Preferences.php:243-264`

```php
public function lastActivity(): string
{
    $instance     = PreferencesSingleton::getInstance();
    $pref         = $instance->getPreference('last_activity');
    if (null !== $pref) {
        return $pref;
    }
    $lastActivity = microtime();
    $preference   = $this->get('lastActivity', microtime());
    // ...
    $setting      = hash('sha256', (string) $lastActivity);
    $instance->setPreference('last_activity', $setting);
    return $setting;
}
```

- 返回值是 `microtime()` 的 SHA256 哈希
- 当 `Preferences::mark()` 被调用时，`lastActivity` 会更新为当前时间
- **影响**：用户活动时间变化 → 哈希变化 → 缓存键变化 → 缓存不命中

**`mark()` 触发场景**：
- 汇率数据变更时（`ProcessesExchangeRates.php:42`）
- 用户执行重要操作时
- 导致所有用户相关缓存全部失效

#### 因素 3：匿名模式（`Steam::anonymous()`）

- 匿名模式开启/关闭状态变化会改变缓存键
- 确保匿名模式下的数据缓存与正常模式隔离

#### 缓存键生成逻辑

**代码证据**：`CacheProperties.php:92-104`

```php
private function hash(): void
{
    $content = '';
    foreach ($this->properties as $property) {
        try {
            $content = sprintf('%s%s', $content, json_encode($property, JSON_THROW_ON_ERROR));
        } catch (JsonException) {
            $content = sprintf('%s%s', $content, hash('sha256', (string) Carbon::now()->getTimestamp()));
        }
    }
    $this->hash = substr(hash('sha256', $content), 0, 16);
}
```

**结论**：只要 `user_id`、`lastActivity`、`anonymous` 任一因素变化，缓存键就会变化，导致缓存不命中。

### 2.3 L2 缓存永久存储问题

**代码证据**：所有汇率缓存写入均使用 `Cache::forever()`

```php
// ExchangeRateConverter.php:254 - 正向查询命中
Cache::forever($key, $rate);

// ExchangeRateConverter.php:264 - 反向查询命中
Cache::forever($key, $rate);

// ExchangeRateConverter.php:284 - EUR桥接成功
Cache::forever($key, $rate);

// CacheProperties.php:87-90 - getFromDB内部缓存
public function store($data): void
{
    Cache::forever($this->hash, $data);
}
```

**⚠️ 注意**：默认值兜底分支（`return '1'`）不会写入缓存。

### 2.4 缓存失效触发点

全局搜索 `Cache::clear()` 仅在三个位置被调用：

| 文件 | 触发场景 |
|------|---------|
| `ProcessesExchangeRates.php:43` | 汇率数据创建/更新/删除时 |
| `InstallController.php:157` | 系统安装时 |
| `DebugController.php:112` | 调试操作时 |

**核心失效逻辑**：`ProcessesExchangeRates` 监听器（`app/Listeners/Model/CurrencyExchangeRate/ProcessesExchangeRates.php:40-52`）：

```php
public function handle(CreatedCurrencyExchangeRate|DestroyedCurrencyExchangeRate|UpdatedCurrencyExchangeRate $event): void
{
    Preferences::mark();  // 更新 lastActivity，影响 CacheProperties 键
    Cache::clear();       // ⚠️ 清除全部缓存，不仅仅是汇率缓存
    // ... 重新计算相关货币金额
}
```

### 2.5 缓存一致性风险

**问题 1：** `Cache::clear()` 是**全局清除**，会清除所有缓存数据，不仅仅是汇率缓存。

**问题 2：** 只有汇率数据变更才会触发缓存清除，以下情况缓存不会自动失效：

- 汇率下载任务执行，但没有新增/更新/删除操作（例如数据已存在）
- 系统时间流逝，旧汇率数据未触发模型事件
- 手动修改数据库中的汇率数据，绕过模型事件

**问题 3：** 缓存键包含日期，但缓存永久存储，意味着：

- 2025-05-20 查询过 2025-05-15 的汇率会被永久缓存
- 即使后续补充了 2025-05-15 的汇率数据，已缓存的值不会自动更新

---

## 三、native_amount 与 pc_amount 汇总链路分析

### 3.1 字段关系与命名

**数据库层**：`transactions` 表中的字段是 `native_amount`

**代码层**：查询时通过别名映射为 `pc_amount`

**代码证据**：`GroupCollector.php:140`

```php
// 字段映射
'source.native_amount as pc_amount',
'source.native_foreign_amount as pc_foreign_amount',
```

### 3.2 完整数据链路

```
交易创建/更新
    ↓
TransactionObserver::created/updated
    ↓
ConvertsAmountToPrimaryAmount::convert()
    ↓
ExchangeRateConverter::convert() → 计算汇率
    ↓
写入 transactions.native_amount 字段
    ↓
报表查询时
    ↓
GroupCollector 执行 SQL 查询
    ├─ SELECT source.native_amount as pc_amount
    ├─ SELECT source.native_foreign_amount as pc_foreign_amount
    ↓
TransactionSummarizer::groupByCurrencyId()
    ├─ 读取 $journal['pc_amount']
    └─ 按货币汇总求和
    ↓
报表展示
```

### 3.3 交易保存时的金额计算

**代码证据**：`TransactionObserver.php:49-70`

```php
private function updatePrimaryCurrencyAmount(Transaction $transaction): void
{
    // 转换交易金额到主货币
    $params = new ConversionParameters();
    $params->amountField = 'amount';
    $params->primaryAmountField = 'native_amount';
    $params->date = $transaction->transactionJournal->date;
    ConvertsAmountToPrimaryAmount::convert($params);
    
    // 同时转换外币金额到主货币
    $params = new ConversionParameters();
    $params->amountField = 'foreign_amount';
    $params->primaryAmountField = 'native_foreign_amount';
    ConvertsAmountToPrimaryAmount::convert($params);
}
```

### 3.4 报表汇总时的字段选择

**代码证据**：`TransactionSummarizer.php:67-90`

```php
if ($this->convertToPrimary) {
    $usePrimary = $this->default->id !== (int) $journal['currency_id'];
    $useForeign = $this->default->id === (int) $journal['foreign_currency_id'];
    
    if ($usePrimary) {
        $field = 'pc_amount';  // ✅ 使用从 native_amount 映射来的字段
        // 切换货币信息为主货币
    }
    if ($useForeign) {
        $field = 'foreign_amount';  // 外币恰好是主货币时，直接用外币金额
    }
}
```

### 3.5 GroupCollector 中的求和逻辑

**代码证据**：`GroupCollector.php:960-989`

```php
$pcAmount = (string) ('' === $transaction['pc_amount'] ? '0' : $transaction['pc_amount']);
$pcForeignAmount = (string) ('' === $transaction['pc_foreign_amount'] ? '0' : $transaction['pc_foreign_amount']);

// 主货币金额汇总
$groups[$groudId]['sums'][$currencyId]['pc_amount'] 
    = bcadd((string) $groups[$groudId]['sums'][$currencyId]['pc_amount'], $pcAmount);

// 外币金额汇总（如果存在）
if (null !== $transaction['foreign_amount'] && null !== $transaction['foreign_currency_id']) {
    $groups[$groudId]['sums'][$currencyId]['pc_amount'] 
        = bcadd($groups[$groudId]['sums'][$currencyId]['amount'], $pcForeignAmount);
}
```

### 3.6 关键结论

1. **命名关系**：数据库字段 `native_amount` = 查询结果字段 `pc_amount`
2. **持久化**：`native_amount` 是持久化字段，保存在数据库中
3. **汇总方式**：报表汇总时直接读取 `pc_amount`（即 `native_amount`）进行求和
4. **汇率使用时机**：汇率换算发生在**交易保存时**，而非报表查询时

---

## 四、历史交易日期与取汇率时间点一致性分析

### 4.1 交易保存时的汇率时间点

**⚠️ 关键发现**：交易金额换算使用**当前日期**的汇率，而非交易日期的汇率！

**代码证据**：`ConvertsAmountToPrimaryAmount.php:89`

```php
$newAmount = $converter->convert(
    $params->originalCurrency,
    $primaryCurrency,
    now(),  // ⚠️ 使用当前时间！而不是 $params->date
    $amount
);
```

虽然 `ConversionParameters` 类有 `$date` 属性且默认设为 `now()`，但在实际调用时传入的是 `now()` 而非交易日期。

### 4.2 影响分析

**场景 1：编辑历史交易**

| 时间点 | 事件 | 汇率使用 |
|--------|------|----------|
| 2025-01-15 | 交易发生，USD/EUR = 0.88 | - |
| 2025-05-20 | 用户编辑并保存该交易 | 使用 2025-05-20 的汇率（例如 0.92） |

**结果**：1月15日发生的交易，使用 5月20日保存时的汇率换算。

**场景 2：汇率补充后重算**

| 时间点 | 事件 |
|--------|------|
| 2025-05-20 | 补充了 2025-01-15 的历史汇率 |
| 2025-05-21 | 触发 `PrimaryAmountRecalculationService` 重算 | 使用 `today()`（2025-05-21）的汇率，仍不是 01-15 的汇率 |

**代码证据**：`PrimaryAmountRecalculationService.php:333-338`

```php
$account->pivot->native_current_amount = $converter->convert(
    $piggyBank->transactionCurrency,
    $currency,
    today(),  // ⚠️ 依然使用今天
    (string)$account->pivot->current_amount
);
```

### 4.3 报表查询时的汇率时间点

**报表中的余额计算**：`Steam.php:138-139`

```php
$converter = new ExchangeRateConverter();
$pcVirtualBalance = $converter->convert($currency, $primary, $date, $virtualBalance);
```

这里使用了 **查询日期** `$date` 的汇率。

### 4.4 不一致性总结

| 操作场景 | 汇率日期 | 代码位置 |
|---------|---------|---------|
| 交易保存/编辑 | `now()` 当前日期 | `ConvertsAmountToPrimaryAmount.php:89` |
| 储蓄账户余额重算 | `today()` 今天 | `PrimaryAmountRecalculationService.php:336` |
| 报表余额计算（虚拟余额） | 查询日期 `$date` | `Steam.php:139` |
| 交易金额换算（`getRate` 方法） | 传入的 `$date` | `ExchangeRateConverter.php:239` |

**问题**：
- 保存时用 `now()`，查询时用交易日期，导致两者不一致
- 历史交易的 `native_amount` 反映的是**保存时**的汇率，不是**交易发生时**的汇率

---

## 五、汇率缺失对报表的偏差影响

### 5.1 汇率缺失触发条件

当以下所有条件满足时，汇率缺失并返回 `'1'`：

1. L1/L2 缓存未命中
2. 正向数据库无 `from → to` 汇率
3. 反向数据库无 `to → from` 汇率
4. `from → EUR` 汇率缺失（返回 `'0'`）
5. `to → EUR` 汇率缺失（返回 `'0'`）

### 5.2 各类报表的偏差分析

#### 5.2.1 账户余额报表

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

#### 5.2.2 收支汇总报表

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

#### 5.2.3 预算执行报表

**场景**：
- 月度预算：1,000 EUR
- 实际消费：1,200 USD（真实汇率 0.92，实际 1,104 EUR）
- 汇率缺失

**报表显示**：消费 1,200 EUR，预算使用率 120%

**实际应为**：消费 1,104 EUR，预算使用率 110.4%

**偏差**：预算使用率高估约 9.6 个百分点

#### 5.2.4 跨货币转账报表

**场景**：
- 从 USD 账户转出 1,000 USD 到 EUR 账户
- 真实汇率：1 USD = 0.92 EUR
- 汇率缺失

**报表显示**：
- USD 账户减少 1,000 USD（换算为 1,000 EUR）
- EUR 账户增加 920 EUR
- 报表显示「汇兑损失 80 EUR」

**实际**：转账本身不应产生汇兑损失（直接兑换时已完成）

**偏差**：凭空出现 80 EUR 损失

### 5.3 数据污染扩散

#### 5.3.1 数据库污染（而非缓存污染）

⚠️ **修正**：由于 `return '1'` 分支不写入缓存，所以不会直接造成缓存污染，但会造成**数据库污染**：

```
汇率缺失 → 返回 1 → 计算 native_amount → 写入数据库 transactions 表
                                                          ↓
                                        后续报表汇总时直接读取该值
```

**补充说明**：如果在 `return '1'` 之前的 `getEuroRate` 调用中命中了数据库（例如 EUR 的汇率存在），那么 `getFromDB` 会将 EUR 汇率写入 `CacheProperties` 缓存，但 `return '1'` 本身不会写入缓存。

#### 5.3.2 汇总级联影响

| 层级 | 影响对象 | 偏差来源 |
|-----|---------|---------|
| 1 | 交易 `native_amount` 字段 | 汇率缺失时保存到数据库 |
| 2 | 账户余额汇总 | 多笔交易偏差累积 |
| 3 | 分类/预算汇总 | 账户余额偏差传导 |
| 4 | 净资产统计 | 全系统汇总偏差 |

### 5.4 偏差检测方法

**日志检测**：
```bash
grep "There is not enough information to convert" storage/logs/*.log
```

**数据校验 SQL（修正语法）**：
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
    t.transaction_currency_id != tj.transaction_currency_id
    AND t.native_amount IS NOT NULL
    AND ABS(t.native_amount / t.amount) NOT BETWEEN 0.5 AND 2.0;
```

**修正说明**：
- 原 SQL 中 `t.transaction_currency_id != t.foreign_currency_id` 条件错误，应比较交易货币与日记账货币
- 增加 `t.native_amount IS NOT NULL` 避免除零错误
- 修正括号语法错误

---

## 六、修复与优化建议

### 6.1 紧急修复

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

// 修复后（24小时过期）
Cache::put($key, $rate, now()->addHours(24));
```

#### 修复 3：汇率缺失时抛出异常或标记

```php
// 当前
return '1';

// 建议
Log::error(sprintf('Exchange rate missing: %s->%s @ %s', $from->code, $to->code, $date->format('Y-m-d')));
// 或抛出 MissingExchangeRateException
```

### 6.2 架构优化

| 优化方向 | 具体措施 |
|---------|---------|
| 汇率监控 | 增加汇率健康检查，报表生成前验证 |
| 数据完整性 | 汇率缺失告警，通知用户数据可能不准确 |
| 缓存精度 | 按日期缓存，支持历史汇率查询 |
| 重算机制 | 汇率更新后自动触发历史交易金额重算 |

### 6.3 运维建议

1. **定期执行**：`php artisan cache:clear` 清除缓存
2. **监控日志**：告警 "There is not enough information"
3. **验证数据**：每月核对汇率完整性
4. **重算机制**：补充汇率后执行 `PrimaryAmountRecalculationService`

---

## 七、核心代码引用速查

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
| GroupCollector字段映射 | `GroupCollector.php` | 136-154 |
| CacheProperties构造函数 | `CacheProperties.php` | 43-51 |
| Preferences::lastActivity | `Preferences.php` | 243-264 |
| TransactionSummarizer汇总 | `TransactionSummarizer.php` | 67-90 |
