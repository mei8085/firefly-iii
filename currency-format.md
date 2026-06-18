# 货币偏好、多币种切换、汇率折算与本地化格式交互分析

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [货币偏好与用户设置](#2-货币偏好与用户设置)
3. [金额本地化格式化](#3-金额本地化格式化)
4. [交易数据模型与多币种字段](#4-交易数据模型与多币种字段)
5. [汇率转换机制](#5-汇率转换机制)
6. [账户余额计算流程](#6-账户余额计算流程)
7. [报表数值计算](#7-报表数值计算)
8. [convertToPrimary 开关完整传递链路](#8-converttoprimary-开关完整传递链路)
9. [区间余额未折算场景深度分析](#9-区间余额未折算场景深度分析)
10. [完整交互流程图](#10-完整交互流程图)

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

### 2.5 偏好的保存与触发重算

**两种保存路径，行为不同**：

#### 路径 A：Web 设置页面（/preferences）

[PreferencesController::postIndex()](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Http/Controllers/PreferencesController.php#L284-L295)

```
用户提交 convertToPrimary=1（Web 表单 POST）
  │
  ├─ Preferences::set('convert_to_primary', true)
  ├─ PreferencesSingleton::getInstance()->resetPreferences()  ← 清空单例缓存
  └─ event(new UserGroupChangedPrimaryCurrency($user->userGroup))
       │
       └─ [RecalculatesPrimaryCurrencyAmounts](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Listeners/System/RecalculatesPrimaryCurrencyAmounts.php#L34-L45) 监听器
            └─ Amount::convertToPrimary() === true ?
                 ├─ 是 → PrimaryAmountRecalculationService::recalculate()  // 批量重算所有交易 native_amount
                 └─ 否 → 仅输出日志 "Will NOT convert..."
```

> 注意：只有当开关从 false 切换到 true 时才会触发 `UserGroupChangedPrimaryCurrency` 事件（见 L286 的 if 判断）。

#### 路径 B：API 接口（/api/v1/preferences/{name}）

[Api/V1/Controllers/User/PreferencesController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Api/V1/Controllers/User/PreferencesController.php#L105-L148)

```
PUT /api/v1/preferences/{name} 或 POST /api/v1/preferences
  │
  └─ Preferences::set($name, $data)    ← 仅写偏好值
       │
       ├─ ❌ 不触发 UserGroupChangedPrimaryCurrency 事件
       ├─ ❌ 不调用 PreferencesSingleton::resetPreferences()
       └─ ❌ 不重算 native_amount
```

**重要区别**：从仪表盘页面内通过前端 `setVariable()` 切换开关，走的是 **API 路径**，因此不会触发 `native_amount` 批量重算，也不会重置后端单例缓存。但 `TransactionObserver` 在每次交易创建/更新时仍会按当前开关状态折算 `native_amount`。

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
  1. 取 startDate 前一日的账户余额作为起点（调用 accountsBalancesOptimized）
  2. SQL 查询区间内按日期+货币分组的 SUM(amount)
  3. 遍历每条日+币种记录：
     ├─ 对应币种 code 余额 += 当日和
     ├─ convertToPrimary=false：
     │    └─ balance += 当日和（所有币种都直接加到 balance）
     └─ convertToPrimary=true：
          ├─ 当日和折算为 P → 累加到 pc_balance
          └─ 若币种 == 账户货币：同时加到 balance
  4. 每日期末余额存入结果数组
```

---

## 8. convertToPrimary 开关完整传递链路

### 8.1 链路全景：偏好存储 → 后端注入 → API → 前端

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        第一阶段：偏好持久化（两条路径）                              │
│                                                                                   │
│  ┌───────────────────────────┐        ┌─────────────────────────────┐            │
│  │  路径 A：设置页面 Web 表单 │        │  路径 B：仪表盘内开关（API）  │            │
│  │  POST /preferences        │        │  PUT /api/v1/preferences/…  │            │
│  │                           │        │                             │            │
│  │  PreferencesController    │        │  Api\V1\...\PreferencesCtrl │            │
│  │    ::postIndex()          │        │    ::update() / ::store()   │            │
│  │    ✓ 重置单例缓存         │        │    ✗ 不重置单例缓存          │            │
│  │    ✓ 触发重算事件         │        │    ✗ 不触发重算事件          │            │
│  └─────────────┬─────────────┘        └──────────────┬──────────────┘            │
│                │                                     │                            │
│                └──────────────┬──────────────────────┘                            │
│                               │                                                   │
│                               ▼                                                   │
│                     Preferences::set()  ← 共同点：都写数据库偏好                    │
│                                                                                   │
└───────────────────────────────┬───────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        第二阶段：后端 Controller 注入                              │
│                                                                                   │
│  每个 HTTP 请求经过基类 Controller middleware                                      │
│  [Controller.php#L137-L173](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Http/Controllers/Controller.php#L137-L173) │
│         │                                                                         │
│         ├─ $this->primaryCurrency  = Amount::getPrimaryCurrency();                │
│         ├─ $this->convertToPrimary = Amount::convertToPrimary();                  │
│         │     ├─ 用户偏好 convert_to_primary = true ?                              │
│         │     └─ AND 系统配置 cer.enabled = true ?  ← 双条件判断                   │
│         │                                                                         │
│         ├─ View::share('convertToPrimary', $this->convertToPrimary)               │
│         │     └─ 所有 Blade/Twig 视图可直接使用变量                                  │
│         │                                                                         │
│         └─ 所有继承 Controller 的子类自动获得 $this->convertToPrimary               │
│              (包括 Json/*、Chart/*、Api/V1/* 等所有控制器)                          │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
              ┌─────────────────────┴─────────────────────┐
              ▼                                           ▼
┌──────────────────────────────┐          ┌─────────────────────────────────────┐
│    第三阶段 A：API JSON 数据   │          │    第三阶段 B：V2 前端 Blade 视图     │
│                              │          │                                      │
│  Api\V1\Controllers\         │          │  [v2/index.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/resources/views/v2/index.blade.php) │
│    Summary\BasicController   │          │    @include('partials.dashboard.*')  │
│    Chart\AccountController   │          │                                      │
│                              │          │  internalsModal 中复选框：             │
│  - 后端侧直接使用             │          │    x-model="convertToPrimary"        │
│    $this->convertToPrimary   │          │    @change="savePrimarySettings"     │
│    控制返回字段结构            │          │                                      │
│                              │          │  初始值由前端 JS 异步从 API 获取         │
└──────────────────────────────┘          └──────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        第四阶段：前端 Alpine.js 组件                                │
│                                                                                   │
│  [dashboard.js](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/resources/assets/v2/src/pages/dashboard/dashboard.js)（主入口） │
│    init():                                                                        │
│      Promise.all([                                                                │
│        getVariable('convert_to_primary', false),   ← 用户偏好                     │
│        getConfiguration('cer.enabled', false)       ← 系统配置                    │
│      ]).then((values) => {                                                         │
│        this.convertToPrimary = values[1] && values[3];  ← 前端也做双条件判断       │
│      });                                                                           │
│                                                                                   │
│      getVariable(name, default):                                                   │
│         ├─ window.store.get(name)  → 优先读本地 store                             │
│         └─ 否则 GET /api/v1/preferences/{name}  → 走 API 读服务端                 │
│              └─ Api\V1\Controllers\User\PreferencesController                     │
│                                                                                   │
│    savePrimarySettings(event):                                                     │
│      setVariable('convert_to_primary', target.checked)                             │
│         ├─ window[name] = value        → 全局变量                                 │
│         ├─ window.store.set(name, value) → 本地 store                             │
│         └─ PUT /api/v1/preferences/{name} → 写回服务器                            │
│              └─ 失败则 POST 创建新偏好（API 路径，不触发重算）                       │
│                                                                                   │
│    事件广播：$dispatch('convert-to-primary', target.checked)                       │
│         │                                                                         │
│         ├── [accounts.js](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/resources/assets/v2/src/pages/dashboard/accounts.js) eventListeners │
│         │    ├─ this.convertToPrimary = event.detail                               │
│         │    ├─ this.accountList = []  ← 清空列表缓存                              │
│         │    ├─ chartData = null      ← 清空图表缓存                               │
│         │    └─ loadChart() + loadAccounts()  ← 重新拉取数据                       │
│         │                                                                         │
│         └── [boxes.js](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/resources/assets/v2/src/pages/dashboard/boxes.js) eventListeners │
│              ├─ this.convertToPrimary = event.detail                               │
│              ├─ this.boxData = null   ← 清空盒子缓存                               │
│              └─ loadBoxes()            ← 重新拉取 Summary API                     │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 各 API 端点如何消费开关

| API 端点 | 所在文件 | 消费方式 | 影响的字段 |
|----------|----------|----------|------------|
| `GET /api/v1/summary/basic` | [BasicController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Api/V1/Controllers/Summary/BasicController.php#L116-L314) | `Amount::convertToPrimary()` 判断是否将收入/支出折算为单一主货币 | `balance-in-{code}`、`spent-in-{code}`、`earned-in-{code}`、`bills-paid/unpaid-in-{code}` 是按多币种分组还是按单一主货币合并 |
| `GET /api/v1/chart/account/overview` | [AccountController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Api/V1/Controllers/Chart/AccountController.php#L77-L171) | `$this->convertToPrimary` 传入 `Steam::finalAccountBalanceInRange()` | 返回数据包含 `entries`（原始币种）；开关开启时额外包含 `pc_entries`（折算后主货币），关闭时 `pc_entries` 字段不存在 |
| `GET /api/v1/accounts/{id}` | [AccountTransformer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Transformers/AccountTransformer.php#L58-L158) + [AccountEnrichment.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/JsonApi/Enrichments/AccountEnrichment.php#L241-L257) | 构造时读 `Amount::convertToPrimary()`，决定是否填充 `pc_*` 字段 | `pc_current_balance`、`pc_opening_balance`、`pc_virtual_balance`、`pc_debt_amount`、`pc_balance_difference`；开关关闭时这些字段为 `null` |
| `GET /json/box/balance` | [BoxController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Http/Controllers/Json/BoxController.php#L62-L137) | `$this->convertToPrimary` 决定 `Amount::getAmountFromJournal()` 取 `amount` 还是 `pc_amount` | 各币种汇总的 `sums`、`incomes`、`expenses` 按主货币合并还是按原币种分桶 |
| `GET /json/box/net-worth` | [BoxController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Http/Controllers/Json/BoxController.php#L142-L189) | 内部调用 `NetWorth::byAccounts()` 受开关影响 | 净资产按多币种还是按主货币展示 |

### 8.3 前端组件的消费方式

| 组件 | 文件 | 消费方式 | 展示差异 |
|------|------|----------|----------|
| **accounts.js（账户图表）** | [accounts.js](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/resources/assets/v2/src/pages/dashboard/accounts.js#L102-L119) | `this.convertToPrimary ? Object.values(current.pc_entries) : Object.values(current.entries)`；Y 轴 key 分别用 `primary_currency_code` 或 `currency_code`，按货币 code 去重后生成独立 Y 轴 | 开启：所有账户共享主货币单 Y 轴；关闭：按货币种类生成多条 Y 轴，同币种账户共享同一条 Y 轴 |
| **accounts.js（账户列表）** | [accounts.js](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/resources/assets/v2/src/pages/dashboard/accounts.js#L257-L258) | `formatMoney(parent.attributes.current_balance, currency_code)` 或 `formatMoney(parent.attributes.pc_current_balance, primary_currency_code)` | 卡片展示账户货币余额 vs 主货币折算余额 |
| **boxes.js（汇总盒子）** | [boxes.js](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/resources/assets/v2/src/pages/dashboard/boxes.js#L90-L139) | 直接消费 Summary API 返回的 `balance-in-{code}` key，根据 currency_code 用 `formatMoney()` | 开启时只有一个主货币盒子；关闭时每种货币独立盒子 |
| **format-money.js** | [format-money.js](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/resources/assets/v2/src/util/format-money.js#L23-L37) | 使用浏览器 `Intl.NumberFormat(locale, {style:'currency', currency: code})` | 完全由 locale + ISO 4217 货币代码决定符号、小数点、千分位格式 |

---

## 9. 区间余额未折算场景深度分析

### 9.1 场景定义

"区间余额未折算" = `convertToPrimary = false` 时调用 `Steam::finalAccountBalanceInRange()`。

核心代码位于 [Steam.php#L508-L511](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/Steam.php#L508-L511)：

```php
if (!$convertToPrimary) {
    $currentBalance['balance'] = bcadd((string) $currentBalance['balance'], $sumOfDay);
}
```

### 9.2 数据流：未折算 vs 折算对比

| 步骤 | 未折算 (convertToPrimary=false) | 折算 (convertToPrimary=true) |
|------|----------------------------------|-------------------------------|
| **起始余额** | `accountsBalancesOptimized(..., false)`：`balance` = 账户货币的余额（仅账户货币求和 + virtual_balance）；同时返回各币种明细字段（如 `EUR`、`USD`） | `accountsBalancesOptimized(..., true)`：`balance` = 账户货币余额；`pc_balance` = 所有币种折入主货币的总和（含 virtual_balance 折算） |
| **每日累加逻辑** | 按 `date + currency_id` 分组遍历：<br>1) 对应币种 code 字段 += 当日和<br>2) `balance` += 当日所有币种的和（直接数值相加） | 按 `date + currency_id` 分组遍历：<br>1) 对应币种 code 字段 += 当日和<br>2) `pc_balance` += 折算后主货币金额<br>3) 若币种 == 账户货币：`balance` += 当日和 |
| **返回字段** | `balance` + 各币种 code 明细字段（无 `pc_balance`） | `balance` + `pc_balance` + 各币种 code 明细字段 |
| **Chart API 消费** | [AccountController.php#L163-L166](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Api/V1/Controllers/Chart/AccountController.php#L163-L166)：`entries[$label] = $previous`；`pc_entries` 字段不存在 | `entries[$label] = $previous; pc_entries[$label] = $pcPrevious` |
| **前端显示** | `accounts.js` L114-119：用 `entries` + `currency_code`，按币种去重生成多条 Y 轴 | `accounts.js` L103-113：用 `pc_entries` + `primary_currency_code`，所有账户同一条 Y 轴 |

### 9.3 未折算场景的典型问题与数据含义

#### 问题 1：多币种账户 `balance` 字段逐步变成无意义的混合数值

以美元账户为例，假设期初 USD 1000，期间发生一笔 EUR 收入 500：

```
起始余额（来自 accountsBalancesOptimized）：
  balance = USD 1000     ← 只有账户货币，准确
  USD = 1000
  EUR = 0

第1天 EUR 收入 500：
  USD = 1000（不变）
  EUR = 500
  balance = 1000 + 500 = 1500  ← USD 和 EUR 数值直接相加，单位混合！
```

**关键点**：
- 起始 `balance` 是**准确的账户货币余额**（仅账户货币求和）
- 但随着多币种交易发生，`balance` 逐步变成不同币种的数值直接字符串相加，经济含义失真
- 各分币种字段（如 `USD`、`EUR`）始终准确

#### 问题 2：前端图表 Y 轴 — 按币种去重，同币种共享

`accounts.js` 在 `convertToPrimary=false` 时的 Y 轴生成逻辑（L132-150）：

```javascript
// 第 1 步：每个 dataset 设置 yAxisID
yAxis = 'y' + current.currency_code;   // 如 'yUSD'、'yEUR'
dataset.yAxisID = yAxis;

// 第 2 步：按币种去重生成 scales
for (let currency in currencies) {
    let code = 'y' + currencies[currency];
    if (!options.options.scales.hasOwnProperty(code)) {
        options.options.scales[code] = {
            // 每个币种一条独立 Y 轴
            ticks: { callback: value => formatMoney(value, currencies[currency]) }
        };
    }
}
```

这意味着：
- 两个同币种账户 → 共享同一条 Y 轴 → 数值可比较 ✅
- 两个不同币种账户 → 各有独立 Y 轴 → 数值单位不同但视觉上可能重叠 ⚠️
- 每条 Y 轴的刻度都用对应货币的 `formatMoney()` 格式化，标签是正确的

#### 问题 3：AccountTransformer 的 `pc_*` 字段为 null

关闭开关时 [AccountEnrichment.php#L241-L257](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Support/JsonApi/Enrichments/AccountEnrichment.php#L241-L257) 的两个 `if` 分支都不执行：

```php
if ($this->convertToPrimary && $currency->id !== $this->primaryCurrency->id) {
    $pcCurrentBalance = $converter->convert(...);  // 不执行
}
if ($this->convertToPrimary && $currency->id === $this->primaryCurrency->id) {
    $pcCurrentBalance = $currentBalance;            // 不执行
}
// 结果：pcCurrentBalance = null（初始值）
```

前端 `accounts.js` L258 的处理：

```javascript
pc_current_balance: null === parent.attributes.pc_current_balance 
    ? null 
    : formatMoney(parent.attributes.pc_current_balance, parent.attributes.primary_currency_code),
```

→ 用户侧看到账户列表卡片上主货币余额位置为空白。

### 9.4 Summary API 在未折算场景的行为

[BasicController::getBalanceInformation()](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Api/V1/Controllers/Summary/BasicController.php#L210-L224)：

```php
if (!$convertToPrimary) {
    // 按原始 currency_id 分桶：收入、支出各算各的
    foreach ([$expenses, $incomes] as $array) {
        foreach ($array as $entry) {
            $sums[$currencyId]['sum'] = bcadd(..., $entry['sum']);
        }
    }
}
```

返回结果示例（未折算）：

```json
{
  "balance-in-EUR": {"monetary_value": "1200.00", "currency_code": "EUR", ...},
  "balance-in-USD": {"monetary_value":  "800.00", "currency_code": "USD", ...}
}
```

→ `boxes.js` 会为每种货币渲染一个独立的余额盒子。

### 9.5 切换开关时的缓存失效机制

| 层级 | 缓存位置 | 失效触发 |
|------|----------|----------|
| 后端 SQL 结果 | `CacheProperties`（通常是 Laravel Cache） | key 包含 `$convertToPrimary` → 切换后 key 不同，自然不命中 |
| 前端 store（Pinia/Vuex 类似） | `window.store` | `setVariable()` 直接覆盖新值；`observe('convert_to_primary')` 监听器触发 reload |
| 前端图表数据 | `chartData` 局部变量 + `window.store` 带 cacheKey | 切换开关时前端 JS 显式设为 `null`，强制重新请求 |
| 前端账户列表 | `accountList` 局部变量 | 切换开关时前端 JS 显式设为 `[]`，强制重新请求 |

---

## 10. 完整交互流程图

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

### 总结：六个关键决策点

1. **偏好持久化层**：有两条保存路径 — 设置页面 Web 表单（触发 `UserGroupChangedPrimaryCurrency` 事件 → 批量重算 `native_amount`）vs 仪表盘内 API 保存（仅写偏好，不触发重算）。只有前者会触发全量重算。
2. **后端注入层**：基类 [Controller.php](file:///d:/fz/0601-2/solo-dogfeeding/code/31-firefly-iii/app/Http/Controllers/Controller.php#L148-L152) 的 middleware 每个请求重新读 `Amount::convertToPrimary()`，双条件判断（用户偏好 + 系统配置），结果同时赋给 `$this->convertToPrimary` 和 `View::share`
3. **API 字段结构层**：开关决定返回数据是否包含 `pc_*` 字段族（`pc_entries`、`pc_current_balance`、`pc_balance`），关闭时这些字段为 null 或不存在
4. **余额数值层**：未折算时起始 `balance` 是账户货币的准确余额，但逐日累加过程中所有币种变动都直接数值相加，导致多币种场景下 `balance` 逐步变成单位混合的无意义数值
5. **前端展示层**：开关决定图表用单 Y 轴（主货币）还是多 Y 轴（按币种去重，同币种共享）；账户卡片上主货币余额位置在开关关闭时显示为空白；前端自身也做双条件判断（偏好 + `cer.enabled`）
6. **缓存失效层**：后端 Cache key 含 `$convertToPrimary` 自动隔离，前端靠 Alpine.js `$dispatch('convert-to-primary')` 事件广播 + 显式清空局部变量触发重新加载
