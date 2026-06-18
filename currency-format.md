# 货币偏好、多币种切换、汇率折算与本地化格式交互分析

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [货币偏好与用户设置](#2-货币偏好与用户设置)
3. [金额本地化格式化](#3-金额本地化格式化)
4. [交易数据模型与多币种字段](#4-交易数据模型与多币种字段)
5. [汇率转换机制](#5-汇率转换机制)
6. [账户余额计算流程](#6-账户余额计算流程)
7. [报表数值计算](#7-报表数值计算)
8. [完整交互流程图](#8-完整交互流程图)

---

## 1. 整体架构概览

Firefly III 的货币系统围绕以下核心类展开协作：

| 组件 | 文件路径 | 职责 |
|------|----------|------|
| Amount 服务 | [Amount.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Amount.php) | 金额格式化、主货币获取、转换开关判断 |
| Steam 服务 | [Steam.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Steam.php) | 账户余额计算、汇率批量转换、Locale获取 |
| Preferences 服务 | [Preferences.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Preferences.php) | 用户偏好存取（含货币偏好） |
| ExchangeRateConverter | [ExchangeRateConverter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Http/Api/ExchangeRateConverter.php) | 汇率查询与金额转换 |
| TransactionObserver | [TransactionObserver.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Handlers/Observer/TransactionObserver.php) | 交易创建/更新时自动折算主货币金额 |
| AmountFormat (Twig) | [AmountFormat.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Twig/AmountFormat.php) | 视图层金额格式化过滤器/函数 |
| NetWorth 报表 | [NetWorth.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Helpers/Report/NetWorth.php) | 净资产报表计算 |

---

## 2. 货币偏好与用户设置

### 2.1 关键偏好项

定义于 [firefly.php#L187-L195](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/config/firefly.php#L187-L195)：

```php
'default_preferences' => [
    'currencyPreference' => 'EUR',   // 货币偏好代码
    'language'           => 'en_US', // 界面语言（同时影响locale）
    'locale'             => 'equal', // 本地化设置：'equal' 表示跟随语言
    'convertToPrimary'   => false,   // 是否折算为主货币
],
```

### 2.2 主货币（Primary Currency）获取逻辑

主入口方法：[Amount::getPrimaryCurrency()](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Amount.php#L281-L292)

```
用户已登录？
  ├─ 是 → 用户所属 UserGroup 是否存在？
  │        ├─ 是 → getPrimaryCurrencyByUserGroup()
  │        │         └─ 查询 user_group 关联的 currencies 中 group_default=true 的记录
  │        │              └─ 不存在？→ 回退到系统货币（EUR），并自动建立关联
  │        └─ 否 → 回退到系统货币
  └─ 否 → getSystemCurrency() → 固定返回 code='EUR' 的货币
```

**关键点**：
- 系统货币硬编码为 EUR（[Amount.php#L315-L318](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Amount.php#L315-L318)）
- 主货币是 **UserGroup 级别**的，而非单个用户级别
- 通过 `user_group_transaction_currency` 中间表的 `group_default` 字段标识

### 2.3 Locale 获取逻辑

入口：[Steam::getLocale()](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Steam.php#L639-L662)

```
1. 读取用户偏好 'locale'，默认值 'equal'
2. 若值为 'equal' → 使用用户的 'language' 偏好值（如 'zh_CN'）
3. Windows 系统下将下划线替换为连字符（zh_CN → zh-CN）
4. 通过 PreferencesSingleton 单例缓存，避免重复查询
```

### 2.4 "是否折算为主货币" 开关

入口：[Amount::convertToPrimary()](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Amount.php#L115-L143)

需要 **两个条件同时满足**：
1. 用户偏好 `convert_to_primary` = true
2. 系统配置 `enable_exchange_rates` = true（来自 FireflyConfig 或 `cer.enabled`）

结果按用户ID缓存在 `PreferencesSingleton` 中。

---

## 3. 金额本地化格式化

### 3.1 核心格式化方法

所有格式化最终汇聚到：[Amount::formatFlat()](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Amount.php#L169-L194)

```php
public function formatFlat(string $symbol, int $decimalPlaces, string $amount, ?bool $coloured = null): string
{
    $amount  = Steam::anonymous() ? '0' : $amount;
    $locale  = Steam::getLocale();
    $rounded = Steam::bcround($amount, $decimalPlaces);
    
    $fmt = new NumberFormatter($locale, NumberFormatter::CURRENCY);
    $fmt->setSymbol(NumberFormatter::CURRENCY_SYMBOL, $symbol);
    $fmt->setAttribute(NumberFormatter::MIN_FRACTION_DIGITS, $decimalPlaces);
    $fmt->setAttribute(NumberFormatter::MAX_FRACTION_DIGITS, $decimalPlaces);
    $result = (string) $fmt->format((float) $rounded);
    
    // 根据正负值包裹不同颜色的 span
    if ($coloured) {
        正数 → text-success money-positive
        负数 → text-danger money-negative
        零   → money-neutral
    }
    return $result;
}
```

### 3.2 格式化调用链

```
Twig 视图层
  ├─ {{ amount|formatAmount }}
  │    └─ AmountFormat::formatAmount()
  │         └─ Amount::getPrimaryCurrency()
  │              └─ Amount::formatAnything(currency, amount, coloured=true)
  │                   └─ Amount::formatFlat(symbol, decimal_places, amount, coloured)
  │
  ├─ {{ formatAmountByAccount(account, amount) }}
  │    └─ 获取账户货币 → 若无则回退主货币 → formatAnything()
  │
  ├─ {{ formatAmountByCurrency(currency, amount) }}
  │    └─ 直接使用传入货币 → formatAnything()
  │
  ├─ {{ formatAmountByCode(amount, 'USD') }}
  │    └─ 按代码查找货币 → 若失败回退主货币 → formatAnything()
  │
  └─ {{ formatAmountBySymbol(amount, '$', 2) }}
       └─ 构造临时 TransactionCurrency 对象 → formatAnything()
```

### 3.3 JavaScript 格式化配置

用于前端图表（accounting.js 库）：[Amount::getJsConfig()](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Amount.php#L268-L279)

通过 `localeconv()` + `NumberFormatter` 获取本地化信息：
- 小数点符号：`mon_decimal_point`
- 千位分隔符：`mon_thousands_sep`
- 正负号格式：根据 `n_sign_posn` / `p_sign_posn`（0-4五种位置模式）生成 accounting.js 格式字符串

**符号位置模式**（[getAmountJsConfig](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Amount.php#L51-L113)）：
| 值 | 含义 | 示例（$123.45负数） |
|----|------|---------------------|
| 0 | 括号包裹 | ($123.45) |
| 1 | 符号在最前 | -$123.45 |
| 2 | 符号在最后 | $123.45- |
| 3 | 符号紧贴货币符号前 | -$123.45 |
| 4 | 符号紧贴货币符号后 | $-123.45 |

---

## 4. 交易数据模型与多币种字段

### 4.1 Transaction 模型字段

定义于 [Transaction.php#L51-L63](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Models/Transaction.php#L51-L63)：

| 字段 | 类型 | 含义 |
|------|------|------|
| `amount` | string | 交易原始货币金额 |
| `transaction_currency_id` | int | 交易货币ID |
| `foreign_amount` | string | 外币金额（可选） |
| `foreign_currency_id` | int | 外币ID（可选） |
| `native_amount` | string | **折算后的主货币金额**（由观察者自动填充） |
| `native_foreign_amount` | string | 折算后的主货币外币金额（观察者自动填充） |

### 4.2 观察者自动折算机制

[TransactionObserver.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Handlers/Observer/TransactionObserver.php) 在交易 `created` 和 `updated` 时触发：

```
Transaction 创建/更新
  └─ updatePrimaryCurrencyAmount()
       ├─ 转换 amount → native_amount
       │    └─ ConvertsAmountToPrimaryAmount::convert()
       │         ├─ 检查 convertToPrimary 开关？
       │         │    └─ 关闭 → native_amount = null，直接返回
       │         ├─ originalCurrency 是否为空？→ 跳过
       │         ├─ 原货币 == 主货币？→ 跳过（无需转换）
       │         ├─ amount 为空或零？→ 两字段都设为 null
       │         └─ ExchangeRateConverter::convert(原货币, 主货币, 现在, 金额)
       │              └─ 结果存入 native_amount
       │
       └─ 转换 foreign_amount → native_foreign_amount（同上流程）
```

### 4.3 Account 模型相关字段

[Account.php#L57](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Models/Account.php#L57)：
- `virtual_balance`：账户虚拟余额（原货币）
- `native_virtual_balance`：折算后的主货币虚拟余额

账户货币通过 `account_meta` 表的 `currency_id` 元数据存储，由 [Steam::getAccountCurrency()](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Steam.php#L562-L577) 读取。

---

## 5. 汇率转换机制

### 5.1 汇率数据存储

[CurrencyExchangeRate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Models/CurrencyExchangeRate.php) 表结构：
- `user_group_id`：用户组隔离
- `from_currency_id` / `to_currency_id`：货币对
- `date`：汇率日期
- `rate`：汇率值（字符串精度）

### 5.2 汇率查询优先级

核心方法：[ExchangeRateConverter::getRate()](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Http/Api/ExchangeRateConverter.php#L239-L287)

```
请求转换 Currency A → Currency B，日期 D
  │
  ├─ 1. 查 Laravel Cache（键：cer-{from}-{to}-{date}）→ 命中直接返回
  │
  ├─ 2. 查数据库正向：user_group.currencyExchangeRates()
  │     where from_currency_id=A, to_currency_id=B, date<=D
  │     order by date DESC → first() → 命中则缓存并返回
  │
  ├─ 3. 查数据库反向：from=B, to=A → 命中则取倒数 1/rate → 缓存返回
  │
  └─ 4. EUR 中转 fallback（三角套汇）：
        ├─ A → EUR 汇率：getEuroRate(A, D)
        │     ├─ 查DB A→EUR 或 EUR→A（取倒数）
        │     └─ 查 config/cer.php 硬编码备份汇率
        ├─ B → EUR 汇率：getEuroRate(B, D)（同上）
        └─ 任一为 0？→ 返回 1（不转换，警告日志）
        └─ 都有值？→ rate = (A→EUR) * (1/(B→EUR)) → 缓存返回
```

### 5.3 备份汇率配置

[cer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/config/cer.php) 中硬编码了以 EUR 为基准的汇率，日期标注为 `2025-04-15`。

### 5.4 汇率下载

[DownloadExchangeRates.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Jobs/DownloadExchangeRates.php) 定时任务：
- 从 `https://ff3exchangerates.z6.web.core.windows.net/{year}/{isoWeek}/{CODE}.json` 下载
- 每个启用货币下载一份，包含对其他所有货币的周汇率
- 按用户组分别保存（每个用户有独立的汇率数据）

### 5.5 转换计算

```php
// [ExchangeRateConverter::convert()](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Http/Api/ExchangeRateConverter.php#L61-L71)
$result = Steam::bcround(bcmul($amount, $rate), $to->decimal_places);
```

使用 BC Math 任意精度运算，按目标货币的 `decimal_places` 四舍五入。

---

## 6. 账户余额计算流程

### 6.1 优化版批量余额计算

[Steam::accountsBalancesOptimized()](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Steam.php#L72-L158)

```
输入：账户集合、截止日期 D、主货币 P、是否折算 C
  │
  ├─ 1. SQL 聚合查询：按 account_id + currency_code 分组
  │     SUM(transactions.amount) where journal.date <= D
  │
  ├─ 2. 遍历每个账户：
  │     │
  │     ├─ 初始化：['pc_balance'=>'0', 'balance'=>'0']
  │     ├─ 获取账户自身货币 currency
  │     ├─ balance = 该账户按 currency.code 的求和值
  │     │
  │     ├─ 若 C=false（不折算）：
  │     │    └─ unset(pc_balance) → balance += virtual_balance
  │     │
  │     └─ 若 C=true（折算主货币）：
  │          ├─ 对各币种余额逐一调用 convertAllBalances()
  │          │    └─ 非主货币都经 ExchangeRateConverter 转为 P → 求和得到 pc_balance
  │          ├─ virtual_balance 也折算为 P → 累加到 pc_balance
  │          └─ 若账户货币 == 主货币：balance += virtual_balance
  │
  └─ 返回结构：
       [account_id => [
           'balance'      => '1234.56',    // 账户货币余额
           'pc_balance'   => '789.01',     // 折算后主货币余额（仅C=true时有）
           'EUR'          => '...',         // 各具体币种余额明细
           'USD'          => '...',
           ...
       ]]
```

### 6.2 从交易流水获取金额（列表展示用）

[Amount::getAmountFromJournal()](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Amount.php#L205-L220)

```
convertToPrimary 开启 且 主货币ID ≠ 交易货币ID？
  ├─ 是 → 取 pc_amount（预计算的主货币字段）
  │       但如果 foreign_currency_id == 主货币ID → 优先取 foreign_amount
  └─ 否 → 取 amount（原始金额）
```

---

## 7. 报表数值计算

### 7.1 净资产报表（NetWorth）

[NetWorth::byAccounts()](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Helpers/Report/NetWorth.php#L58-L113)

```
输入：账户集合 + 日期
  │
  ├─ 1. Steam::accountsBalancesOptimized() 获取批量余额
  │
  ├─ 2. 遍历账户：
  │     ├─ 获取账户货币 currency
  │     ├─ usePrimary = convertToPrimary && 主货币 != currency
  │     ├─ amountToUse = usePrimary ? pc_balance : balance
  │     ├─ 减去 virtual_balance（因净资产不含虚拟余额）
  │     │   └─ usePrimary 时减 native_virtual_balance
  │     └─ 按目标货币 code 分组累加
  │
  └─ 返回：
       [EUR => ['balance'=>'...', 'currency_id'=>.., 'symbol'=>'€', ...],
        USD => [...], ...]
```

### 7.2 日报表 / 区间余额追踪

[Steam::finalAccountBalanceInRange()](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Steam.php#L423-L530)

```
获取起始日余额 → 逐日累加当日变化：
  1. 取 startDate 前一日的账户余额作为起点
  2. SQL 查询区间内按日期+货币分组的 SUM(amount)
  3. 遍历每日：
     ├─ 对应币种 code 余额 += 当日和
     ├─ convertToPrimary=false：
     │    └─ balance += 当日和（仅当币种匹配）
     └─ convertToPrimary=true：
          ├─ 当日和折算为 P → 累加到 pc_balance
          └─ 若币种 == 账户货币：同时加到 balance
  4. 每日期末余额存入结果数组
```

---

## 8. 完整交互流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户设置层                                     │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────────┐  │
│  │ 语言(lang)   │   │ Locale偏好   │   │ convert_to_primary 开关  │  │
│  │ zh_CN        │──▶│ 'equal'      │   │ + enable_exchange_rates  │  │
│  └──────────────┘   └──────┬───────┘   └────────────┬─────────────┘  │
│                            │                         │                │
│                            ▼                         ▼                │
│                    Steam::getLocale()        Amount::convertToPrimary()│
│                            │                         │                │
└────────────────────────────┼─────────────────────────┼────────────────┘
                             │                         │
┌────────────────────────────┼─────────────────────────┼────────────────┐
│                        数据持久层                                     │
│  Transaction 创建/更新   │                         │                │
│         │                 │                         │                │
│         ▼                 │                         ▼                │
│  TransactionObserver      │              主货币 = EUR/用户组默认      │
│         │                 │                         │                │
│         ▼                 │                         ▼                │
│  ConvertsAmountToPrimary  │              TransactionCurrency 模型    │
│         │                 │                         │                │
│         ▼                 │                         │                │
│  native_amount =          │                         │                │
│  convert(原→主, amount)   │                         │                │
│                           │                         │                │
└───────────────────────────┼─────────────────────────┼────────────────┘
                            │                         │
┌───────────────────────────┼─────────────────────────┼────────────────┐
│                       展示计算层                                      │
│                            │                         │                │
│  ┌──────────────┐          ▼                         │                │
│  │  余额计算     │──▶ accountsBalancesOptimized()    │                │
│  │ Steam.php    │          │                         │                │
│  └──────────────┘          │ convertToPrimary?       │                │
│                            │     ├── Yes → 各币种→主货币 求和         │
│                            │     └── No  → 仅账户货币余额              │
│                            │                         │                │
│  ┌──────────────┐          ▼                         ▼                │
│  │  报表计算     │──▶ NetWorth / PopupReport    Amount::formatFlat()  │
│  │ Report/*.php │          │                         │                │
│  └──────────────┘          │                         │                │
│                            │                         ├─ Locale: zh_CN │
│                            │                         ├─ Symbol: ¥/€/$ │
│                            │                         └─ Decimal: 2位  │
│                            │                         │                │
│                            ▼                         ▼                │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │                     Twig 视图渲染                               │   │
│  │  {{ 1234.56|formatAmount }}  →  "¥1,234.56" (绿色/红色span)    │   │
│  └───────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### 总结：三个关键决策点

1. **`convertToPrimary` 开关**：决定余额字段是取原生 `amount/balance` 还是折算后的 `native_amount/pc_balance`
2. **`currencyPreference` + `locale`**：决定格式化时的货币符号、小数点/千分位、正负号位置
3. **账户货币 vs 主货币**：决定单账户展示时显示哪种货币，以及汇总时是否需要汇率折算
