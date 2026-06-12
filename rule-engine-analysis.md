# Firefly III 规则引擎源码分析报告

## 一、概述

Firefly III 的规则引擎是一套基于搜索的交易自动化处理系统，用户可以通过配置**条件（触发器）**和**动作**来实现交易的自动分类、打标签、设置预算等操作。规则引擎采用**搜索驱动**的设计理念，将条件匹配转化为搜索查询，通过搜索结果来驱动动作执行。

核心入口：[SearchRuleEngine.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php)

---

## 二、核心数据模型

### 2.1 模型层次结构

规则引擎采用四层模型结构，从外到内依次为：

```
RuleGroup（规则组）
    └── Rule（规则）
         ├── RuleTrigger（触发器/条件）
         └── RuleAction（动作）
```

### 2.2 各模型详解

#### RuleGroup（规则组）

定义文件：[RuleGroup.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Models/RuleGroup.php)

| 属性 | 类型 | 说明 |
|------|------|------|
| `id` | int | 主键 |
| `title` | string | 规则组名称 |
| `description` | string | 描述 |
| `order` | int | 执行顺序 |
| `active` | bool | 是否激活 |
| `stop_processing` | bool | 组内规则触发后是否停止处理后续规则 |
| `user_id` | int | 所属用户 |

规则组是规则的容器，用于将相关规则组织在一起，并提供组级别的执行控制。

#### Rule（规则）

定义文件：[Rule.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Models/Rule.php)

| 属性 | 类型 | 说明 |
|------|------|------|
| `id` | int | 主键 |
| `rule_group_id` | int | 所属规则组 |
| `title` | string | 规则名称 |
| `description` | string | 描述 |
| `order` | int | 组内执行顺序 |
| `active` | bool | 是否激活 |
| `strict` | bool | 是否严格模式（AND / OR） |
| `stop_processing` | bool | 触发后是否停止处理后续规则 |
| `user_id` | int | 所属用户 |

`strict` 是规则的核心属性，决定了条件匹配的逻辑：
- `strict = true`：**严格模式（AND 逻辑）**，所有触发器必须同时满足
- `strict = false`：**非严格模式（OR 逻辑）**，任一触发器满足即可

#### RuleTrigger（触发器/条件）

定义文件：[RuleTrigger.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Models/RuleTrigger.php)

| 属性 | 类型 | 说明 |
|------|------|------|
| `id` | int | 主键 |
| `rule_id` | int | 所属规则 |
| `trigger_type` | string | 触发器类型（如 `description_is`, `category_is` 等） |
| `trigger_value` | string | 触发器值 |
| `order` | int | 执行顺序 |
| `active` | bool | 是否激活 |
| `stop_processing` | bool | 匹配后是否停止处理后续触发器 |

支持的触发器类型在 [search.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/config/search.php) 中配置，共 100+ 种，涵盖：
- 账户相关：`source_account_is`, `destination_account_contains` 等
- 描述相关：`description_is`, `description_contains` 等
- 分类/预算/标签：`category_is`, `budget_contains`, `tag_is` 等
- 日期相关：`date_on`, `date_before`, `date_after` 等
- 金额相关：`amount_is`, `amount_less`, `amount_more` 等
- 布尔判断：`has_attachments`, `has_any_category`, `source_is_cash` 等

#### RuleAction（动作）

定义文件：[RuleAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Models/RuleAction.php)

| 属性 | 类型 | 说明 |
|------|------|------|
| `id` | int | 主键 |
| `rule_id` | int | 所属规则 |
| `action_type` | string | 动作类型（如 `set_category`, `add_tag` 等） |
| `action_value` | string | 动作值 |
| `order` | int | 执行顺序 |
| `active` | bool | 是否激活 |
| `stop_processing` | bool | 执行后是否停止处理后续动作 |

支持的动作类型在 [firefly.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/config/firefly.php#L416-L449) 中配置，包括：
- 分类操作：`set_category`, `clear_category`
- 预算操作：`set_budget`, `clear_budget`
- 标签操作：`add_tag`, `remove_tag`, `remove_all_tags`
- 描述操作：`set_description`
- 账户操作：`set_source_account`, `set_destination_account`, `switch_accounts`
- 备注操作：`set_notes`, `clear_notes`
- 账单关联：`link_to_bill`
- 类型转换：`convert_withdrawal`, `convert_deposit`, `convert_transfer`
- 其他：`update_piggy`, `delete_transaction`, `set_amount`

---

## 三、条件匹配机制

### 3.1 匹配模式

规则引擎支持两种条件匹配模式，由 `Rule.strict` 属性控制：

#### 严格模式（Strict = true，AND 逻辑）

实现方法：[findStrictRule()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L303-L366)

**工作原理**：
1. 收集所有激活的触发器，构建搜索条件数组
2. 所有条件通过搜索引擎**一次性组合查询**（AND 关系）
3. 返回同时满足所有条件的交易集合

```php
// 伪代码示意
$searchArray = [];
foreach ($triggers as $trigger) {
    $searchArray[$trigger->trigger_type][] = $trigger->trigger_value;
}
// 一次性提交给搜索引擎，AND 逻辑
$searchEngine->parseQuery($allConditions);
return $searchEngine->searchTransactions();
```

#### 非严格模式（Strict = false，OR 逻辑）

实现方法：[findNonStrictRule()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L204-L298)

**工作原理**：
1. 按顺序逐个执行触发器
2. 每个触发器单独执行一次搜索
3. 将所有搜索结果合并，去重后返回

```php
// 伪代码示意
$total = new Collection();
foreach ($triggers as $trigger) {
    $result = $searchEngine->search($trigger);
    $total = $total->merge($result);
    // 支持 stop_processing 提前终止
    if ($trigger->stop_processing && $result->count() > 0) {
        break;
    }
}
return $total->unique();
```

### 3.2 搜索驱动的条件匹配

条件匹配的核心实现基于 [OperatorQuerySearch](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Support/Search/OperatorQuerySearch.php) 类，它将各种触发器类型转换为数据库查询条件。

搜索流程：
1. 解析查询字符串（如 `description_is:"超市" amount_less:"100"`）
2. 将每个操作符映射到对应的查询构建器方法
3. 通过 `GroupCollectorInterface` 执行数据库查询
4. 返回分页结果

---

## 四、动作执行机制

### 4.1 动作工厂（ActionFactory）

定义文件：[ActionFactory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Factory/ActionFactory.php)

动作采用**工厂模式**创建，根据 `action_type` 从配置中查找对应的实现类：

```php
public static function getAction(RuleAction $action): ActionInterface
{
    $class = self::getActionClass($action->action_type);
    return new $class($action);
}
```

动作类型与实现类的映射关系在 [Domain.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Support/Domain.php) 中通过配置 `firefly.rule-actions` 获取。

### 4.2 动作接口（ActionInterface）

定义文件：[ActionInterface.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/ActionInterface.php)

所有动作实现统一接口：

```php
interface ActionInterface
{
    public function actOnArray(array $journal): bool;
}
```

- 输入：交易数组（包含交易的完整信息）
- 返回：`bool` 表示是否实际执行了修改

### 4.3 动作执行示例

以 [SetCategory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetCategory.php) 为例：

```php
public function actOnArray(array $journal): bool
{
    // 1. 获取用户
    $user = User::find($journal['user_id']);
    
    // 2. 查找或创建分类
    $category = $factory->findOrCreate(null, $search);
    
    // 3. 检查是否已有相同分类（幂等性）
    if ((int)$oldCategory?->id === $category->id) {
        return false; // 未修改
    }
    
    // 4. 执行修改
    DB::table('category_transaction_journal')->where(...)->delete();
    DB::table('category_transaction_journal')->insert([...]);
    
    return true; // 已修改
}
```

**关键特性**：
- **幂等性**：如果目标状态与当前状态一致，返回 `false` 表示未修改
- **审计日志**：修改成功后触发 `TransactionGroupRequestsAuditLogEntry` 事件
- **表达式支持**：动作值支持表达式引擎（`ActionExpression`），可动态计算

---

## 五、规则链路的串联与执行流程

### 5.1 整体执行架构

规则引擎的执行入口是 [fire()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L95-L131) 方法，支持两种执行模式：

1. **独立规则模式**：直接设置规则集合（`setRules()`）
2. **规则组模式**：设置规则组集合（`setRuleGroups()`）

### 5.2 规则组执行流程

实现方法：[fireGroup()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L371-L395)

```
开始执行规则组
    │
    ▼
按 order 加载组内激活的规则
    │
    ▼
┌───────────────────────────┐
│   执行第 N 条规则          │
│   （调用 fireRule()）      │
└─────────────┬─────────────┘
              │
              ▼
        规则是否触发？
         /          \
       是           否
        |            |
        ▼            ▼
stop_processing?   继续下一条
    /    \
  是     否
  |      |
  ▼      ▼
停止    继续
```

核心代码逻辑：
```php
foreach ($rules as $rule) {
    $result = $this->fireRule($rule);
    // 规则触发且设置了 stop_processing，则停止组内后续规则
    if ($result && true === $rule->stop_processing) {
        return;
    }
}
```

### 5.3 单条规则执行流程

实现方法：[fireRule()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L426-L442)

```
开始执行规则
    │
    ▼
规则是否激活？── 否 ──> 返回 false
    │ 是
    ▼
严格模式？
    /    \
  是      否
  |       |
  ▼       ▼
fireStrictRule  fireNonStrictRule
    \       /
     \     /
      ▼   ▼
    处理结果（processResults）
        │
        ▼
    返回是否触发
```

### 5.4 动作执行流程

实现方法：[processTransactionJournal()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L574-L589)

```
对交易执行动作序列
    │
    ▼
按 order 加载所有激活的动作
    │
    ▼
┌───────────────────────────┐
│   执行第 N 个动作           │
│   （调用 processRuleAction()）│
└─────────────┬─────────────┘
              │
              ▼
        动作是否执行成功？
         /          \
       是           否
        |            |
        ▼            ▼
stop_processing?   继续下一个
    /    \
  是     否
  |      |
  ▼      ▼
停止    继续
```

核心代码逻辑：
```php
foreach ($actions as $ruleAction) {
    if (false === $ruleAction->active) {
        continue;
    }
    $break = $this->processRuleAction($ruleAction, $transaction);
    if ($break) {
        break; // 动作执行成功且 stop_processing=true 时中断
    }
}
```

### 5.5 完整执行链路图

```
RuleGroup（按 order 排序）
    │
    ├── Rule #1（order=1）
    │     ├── RuleTrigger #1（order=1）← 条件匹配
    │     ├── RuleTrigger #2（order=2）
    │     └── ...
    │           │
    │           ▼
    │     条件匹配结果
    │           │
    │           ▼
    │     RuleAction #1（order=1）← 动作执行
    │     RuleAction #2（order=2）
    │     ...
    │
    ├── Rule #2（order=2）
    │     └── ...
    │
    └── ...
```

---

## 六、匹配冲突的选优策略

### 6.1 冲突解决的核心机制

Firefly III 规则引擎的"匹配冲突选优"并非传统规则引擎中的**优先级算分**机制，而是采用**基于顺序的短路执行**策略。核心思想是：**先配置的规则先执行，执行成功且标记为 stop_processing 的规则会终止后续处理**。

### 6.2 四层优先级控制

规则引擎在四个层级上都提供了 `stop_processing` 控制，形成完整的冲突解决链条：

| 层级 | 控制属性 | 触发条件 | 效果 |
|------|----------|----------|------|
| 触发器层 | `RuleTrigger.stop_processing` | 非严格模式下，触发器匹配到结果 | 停止后续触发器的匹配 |
| 动作层 | `RuleAction.stop_processing` | 动作执行成功（返回 true） | 停止后续动作的执行 |
| 规则层 | `Rule.stop_processing` | 规则触发（匹配到交易） | 停止组内后续规则的执行（仅在组内有效） |
| 规则组层 | `RuleGroup.stop_processing` | - | 保留字段，当前代码中未直接使用 |

### 6.3 触发器层冲突解决

仅在**非严格模式**（OR 逻辑）下生效。

代码位置：[findNonStrictRule() L274-L278](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L274-L278)

```php
// 如果触发器设置了 stop_processing 且匹配到结果，则停止后续触发器
if (true === $ruleTrigger->stop_processing && $result->count() > 0) {
    Log::debug('The trigger in this rule trigger says to stop processing, so stop processing other triggers.');
    break;
}
```

**选优策略**：按 `order` 顺序匹配，第一个匹配成功且设置了 `stop_processing` 的触发器会"获胜"，后续触发器不再执行。

### 6.4 动作层冲突解决

代码位置：[processRuleAction() L546-L556](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L546-L556)

```php
// 只有动作真正执行了修改（result=true）且设置了 stop_processing，才停止后续动作
if (true === $ruleAction->stop_processing && $result) {
    Log::debug(sprintf('Rule action "%s" reports changes AND asks to break, so break!', $ruleAction->action_type));
    return true;
}
```

**选优策略**：
- 按 `order` 顺序执行动作
- 第一个**实际产生修改**且设置了 `stop_processing` 的动作会终止动作链
- 如果动作未产生实际修改（幂等返回 false），即使设置了 `stop_processing` 也不会终止

### 6.5 规则层冲突解决

代码位置：[fireGroup() L388-L392](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L388-L392)

```php
// 在规则组内，如果规则触发且设置了 stop_processing，则停止组内后续规则
if ($result && true === $rule->stop_processing) {
    Log::debug(sprintf('The rule was triggered and rule->stop_processing = true, so group #%d will stop processing further rules.', $group->id));
    return;
}
```

**重要注意**：在**非规则组模式**下（直接通过 `setRules()` 设置规则），`stop_processing` 会被**忽略**：

代码位置：[fire() L107-L115](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L107-L115)

```php
if ($result && true === $rule->stop_processing) {
    Log::debug('Rule #%d has triggered and executed, but calls to stop processing. Since not in the context of a group, do not stop.');
    // 注意：这里只是打日志，并不实际停止！
}
```

### 6.6 选优策略总结

Firefly III 的冲突选优可以总结为：

1. **顺序优先**：所有元素（触发器、动作、规则）都通过 `order` 字段排序，排在前面的先执行
2. **短路终止**：通过 `stop_processing` 标志实现短路逻辑，先匹配/执行成功的可以终止后续处理
3. **实际生效原则**：动作层的 `stop_processing` 只有在动作**实际产生了修改**时才生效
4. **上下文相关**：规则层的 `stop_processing` 仅在**规则组上下文**中生效，独立规则模式下不生效

这种设计的优点是：
- 直观易懂，用户可以通过调整顺序精确控制执行流程
- 灵活可控，每层都可以独立设置终止条件
- 幂等友好，未产生实际修改的动作不会意外中断流程

---

## 七、规则触发的入口

### 7.1 事件驱动的自动触发

规则引擎通过事件监听机制，在交易创建/更新时自动执行。

核心 Trait：[SupportsGroupProcessingTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Listeners/Model/TransactionGroup/SupportsGroupProcessingTrait.php)

```php
protected function processRules(Collection $set, string $type): void
{
    // 1. 获取该类型的所有规则组
    $groups = $ruleGroupRepository->getRuleGroupsWithRules($type);
    
    // 2. 创建规则引擎
    $newRuleEngine = app(RuleEngineInterface::class);
    $newRuleEngine->setUser($user);
    $newRuleEngine->setRuleGroups($groups);
    
    // 3. 对每条交易执行规则（通过 journal_id 限定范围）
    foreach ($array as $journalId) {
        $newRuleEngine->addOperator(['type' => 'journal_id', 'value' => $journalId]);
        $newRuleEngine->fire();
    }
}
```

相关监听器：
- [ProcessesNewTransactionGroup.php]() - 新交易创建时触发
- [ProcessesUpdatedTransactionGroup.php]() - 交易更新时触发

### 7.2 手动触发

用户也可以通过界面或 API 手动触发规则：
- 控制台命令：[ApplyRules.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Console/Commands/Tools/ApplyRules.php)
- Web 控制器：[ExecutionController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Http/Controllers/RuleGroup/ExecutionController.php)
- API 控制器：[TriggerController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Api/V1/Controllers/Models/RuleGroup/TriggerController.php)

---

## 八、服务注册与依赖注入

### 8.1 规则引擎绑定

在 [FireflyServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Providers/FireflyServiceProvider.php#L171-L178) 中注册：

```php
$this->app->bind(static function (Application $app): RuleEngineInterface {
    $engine = app(SearchRuleEngine::class);
    if ($app->auth->check()) {
        $engine->setUser(auth()->user());
    }
    return $engine;
});
```

### 8.2 规则仓库绑定

在 [RuleServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Providers/RuleServiceProvider.php) 中注册规则和规则组的仓库类。

---

## 九、关键设计特点总结

### 9.1 搜索驱动架构

- 条件匹配复用搜索系统，避免重复实现查询逻辑
- 支持 100+ 种搜索操作符，功能丰富
- 可扩展性强，新增搜索操作符即可用于规则条件

### 9.2 多层次控制流

- 四层结构（组→规则→触发器→动作），每层都有顺序和终止控制
- `stop_processing` 机制提供灵活的短路逻辑
- 严格/非严格模式覆盖 AND/OR 两种常见需求

### 9.3 幂等性设计

- 动作在执行前检查当前状态，未修改返回 false
- 避免重复执行导致的问题
- 与 `stop_processing` 配合，确保只有实际修改才会中断流程

### 9.4 可扩展性

- 动作通过配置映射，新增动作只需实现接口并添加配置
- 触发器通过搜索操作符扩展
- 表达式引擎支持复杂的动态值计算

---

## 十、交易上下文挂入查询条件的机制

### 10.1 问题背景

当同一组条件需要反复应用到不同交易上时（例如用户刚创建了10条交易，需要逐条跑规则），规则引擎如何确保每条规则只针对**当前目标交易**进行匹配，而不是误匹配到其他交易？

答案是：通过 **`addOperator` 机制注入 `journal_id` 限定条件**，将全局搜索缩小为单条交易范围的匹配。

### 10.2 完整链路追踪

#### 阶段一：事件监听器发起调用

入口：[SupportsGroupProcessingTrait::processRules()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Listeners/Model/TransactionGroup/SupportsGroupProcessingTrait.php#L29-L66)

```php
protected function processRules(Collection $set, string $type): void
{
    // ... 收集 journal_ids
    
    // 对每条交易，循环调用规则引擎
    foreach ($array as $journalId) {
        Log::debug(sprintf('Fire rule engine for journal #%d', $journalId));
        
        // 关键1：先清除旧的 journal_id，防止上下文污染
        $newRuleEngine->removeOperator('journal_id');
        
        // 关键2：注入当前交易的 ID 作为限定条件
        $newRuleEngine->addOperator(['type' => 'journal_id', 'value' => $journalId]);
        
        // 关键3：执行规则（所有条件都会叠加这个 journal_id 过滤）
        $newRuleEngine->fire();
    }
}
```

**循环策略**：
- 多条交易不是一次性批量传入，而是**逐条循环调用引擎**
- 每次循环前先 `removeOperator('journal_id')` 清除上一条的上下文
- 再 `addOperator()` 注入当前交易的 ID，确保引擎每次只"看见"一条交易

#### 阶段二：SearchRuleEngine 内部持有 operators

定义：[SearchRuleEngine.php L53](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L53)

```php
// 私有成员变量，存储所有外部注入的"附加查询条件"
private array $operators = [];
```

注入方法：[addOperator()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L66-L70)

```php
public function addOperator(array $operator): void
{
    // 格式：['type' => 'journal_id', 'value' => '123']
    $this->operators[] = $operator;
}
```

清除方法：[removeOperator()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L142-L154)

```php
public function removeOperator(string $type): void
{
    // 按 type 过滤掉，防止不同交易之间的 ID 串台
    $new = [];
    foreach ($this->operators as $operator) {
        if ($type === $operator['type']) {
            continue;  // 跳过指定 type 的操作符
        }
        $new[] = $operator;
    }
    $this->operators = $new;
}
```

#### 阶段三：构建查询时叠加注入的操作符

##### 非严格模式下的挂入

代码位置：[findNonStrictRule() L251-L255](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L251-L255)

```php
// 1. 先构建规则自身定义的触发器条件
$searchArray = [];
$searchArray[$ruleTrigger->trigger_type] = sprintf('"%s"', $ruleTrigger->trigger_value);

// 2. ★ 关键：再把外部注入的 operators（如 journal_id）也追加进去
foreach ($this->operators as $operator) {
    Log::debug(sprintf('SearchRuleEngine:: add local added operator: %s:"%s"', $operator['type'], $operator['value']));
    $searchArray[$operator['type']] = sprintf('"%s"', $operator['value']);
}

// 3. 组合后的 searchArray 交给搜索引擎
//    例如最终变成：description_is:"超市" AND journal_id:"123"
```

##### 严格模式下的挂入

代码位置：[findStrictRule() L338-L341](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L338-L341)

```php
// 1. 先收集所有触发器（AND 组合）
foreach ($triggers as $ruleTrigger) {
    $searchArray[$ruleTrigger->trigger_type][] = sprintf('"%s"', $ruleTrigger->trigger_value);
}

// 2. ★ 关键：同样追加注入的 operators（注意是数组形式，因为严格模式下允许多值）
foreach ($this->operators as $operator) {
    $searchArray[$operator['type']][] = sprintf('"%s"', $operator['value']);
}
```

**差异说明**：
- 非严格模式：`$searchArray[$type] = $value`（每个触发器独立一次搜索）
- 严格模式：`$searchArray[$type][] = $value`（所有触发器攒成数组，统一搜索）
- 但无论哪种，**注入的 operators 都会被合并到最终的查询条件中**

#### 阶段四：OperatorQuerySearch 解析 journal_id 操作符

代码位置：[OperatorQuerySearch::updateCollector() L1497-L1501](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Support/Search/OperatorQuerySearch.php#L1497-L1501)

```php
case 'journal_id':
    $parts = explode(',', $value);
    // 调用 GroupCollector 的 setJournalIds 方法，最终落地为 SQL whereIn
    $this->collector->setJournalIds($parts);
    break;
```

#### 阶段五：GroupCollector 落地为 SQL 条件

代码位置：[GroupCollector::setJournalIds()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Helpers/Collector/GroupCollector.php#L540-L551)

```php
public function setJournalIds(array $journalIds): GroupCollectorInterface
{
    if (0 !== count($journalIds)) {
        $integerIDs = array_map(intval(...), $journalIds);
        
        // ★ 最终落地为 SQL 的 WHERE IN 条件
        $this->query->whereIn('transaction_journals.id', $integerIDs);
    }
    return $this;
}
```

### 10.3 上下文挂入的完整数据流图

```
用户创建/更新交易
    │
    ▼
触发 CreatedSingleTransactionGroup / UpdatedSingleTransactionGroup 事件
    │
    ▼
ProcessesNewTransactionGroup::handle()
    │ 调用 $this->processRules($journals, 'store-journal')
    ▼
SupportsGroupProcessingTrait::processRules()
    │
    ├── 加载所有规则组（同一组条件集合）
    │
    └── 对每条交易循环：
         ├─ removeOperator('journal_id')  ← 清空前一条交易的 ID
         ├─ addOperator(['journal_id' => $id])  ← 注入当前交易 ID
         └─ fire()
              │
              ▼
         SearchRuleEngine 构建 searchArray
              │ + operators（包含 journal_id）
              ▼
         OperatorQuerySearch::updateCollector()
              │ case 'journal_id': setJournalIds()
              ▼
         GroupCollector::whereIn('transaction_journals.id', [$id])
              │
              ▼
         SQL 查询：SELECT ... WHERE ... AND transaction_journals.id IN (123)
              │
              ▼
         只返回当前这一条交易（如果满足规则自身条件）→ 执行动作
```

### 10.4 关键设计要点

| 设计点 | 说明 |
|--------|------|
| **逐条循环而非批量** | 每条交易独立调用 `fire()`，保证规则层 `stop_processing` 等语义正确 |
| **先清除后注入** | 每次循环先 `removeOperator` 再 `addOperator`，防止 ID 残留 |
| **同一套规则组复用** | 规则引擎实例只创建一次，`$groups` 不变，仅通过 `operators` 切换目标交易 |
| **落地为 WHERE IN** | 最终由 GroupCollector 通过 `whereIn('id', [$id])` 精确限定范围 |
| **对规则透明** | 规则定义者完全感知不到这个机制，所有条件的写法与全局搜索一致 |

---

## 十一、动作生效后避免重复触发的机制

### 11.1 问题背景

规则引擎修改交易后（例如把交易 A 分类为"餐饮"），**规则引擎自己需要再次广播 `UpdatedSingleTransactionGroup` 事件**来触发后续的清理工作（重算余额、清统计缓存、触发 Webhook 等）。但如果不加防护，这条广播又会让监听器把规则引擎重新跑一遍，就会：

1. 规则引擎执行完动作 → **引擎自己在末尾显式**广播 `UpdatedSingleTransactionGroup`
2. `ProcessesUpdatedTransactionGroup` 监听器收到事件 → 再次调用 `processRules()`
3. 交易再次被规则引擎扫描，又匹配到相同规则 → 又广播一次事件
4. **无限循环 / 重复执行**

Firefly III 通过 **`TransactionGroupEventFlags` 标志位**在事件传播链路中切断重复触发。

#### 两个需要先讲准的关键事实

##### 事实一：`UpdatedSingleTransactionGroup` 是业务层手动广播的，不是 Eloquent Model 事件自动触发的

在整个代码库里，该事件共被显式 `event(new UpdatedSingleTransactionGroup(...))` 广播了 **9 处**，主要来源：

| 广播位置 | 场景 | applyRules 默认值 |
|----------|------|-------------------|
| [SearchRuleEngine::fireStrictRule()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L466) | 规则引擎严格模式执行完 | **false**（引擎显式设置） |
| [SearchRuleEngine::fireNonStrictRule()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L415) | 规则引擎非严格模式执行完 | **false**（引擎显式设置） |
| [TransactionGroupRepository::store()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php#L365) | 用户新建交易（发出 `CreatedSingleTransactionGroup`） | `$data['apply_rules'] ?? true` |
| [AccountServiceTrait](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Services/Internal/Support/AccountServiceTrait.php) | 账户服务内部变更 | **false** |
| [MassController](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Http/Controllers/Transaction/MassController.php) | 批量编辑 | true（默认构造） |
| [ConvertController](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Http/Controllers/Transaction/ConvertController.php) | 转换交易类型 | true（默认构造） |
| [BulkController](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Http/Controllers/Transaction/BulkController.php) | 批量操作 | 取决于请求参数 |
| [CorrectsGroupAccounts](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Console/Commands/Correction/CorrectsGroupAccounts.php) | 修复命令 | **false** |
| [UpdateController (API)](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Api/V1/Controllers/Models/Transaction/UpdateController.php) | API 更新交易 | 取决于请求参数 |

`UpdatedSingleTransactionGroup` **不跟 Eloquent ORM 的 `saving` / `saved` / `updating` / `updated` 事件挂钩**——它完全是业务层自己控制什么时候发。`TransactionGroup` 模型上唯一注册的 Observer 是 [DeletedTransactionGroupObserver](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Handlers/Observer/DeletedTransactionGroupObserver.php)，只监听 `deleting`，与更新事件无关。

##### 事实二：动作本身不会直接触发 `UpdatedSingleTransactionGroup`

动作修改数据库分两种写法，但**无论哪种都不会直接产生 `UpdatedSingleTransactionGroup` 事件**：

**写法 A（绝大多数动作）：`DB::table()->update/insert/delete`**

例如：
- [SetCategory L92-L96](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetCategory.php#L92-L96)：`DB::table('category_transaction_journal')->...`
- [SetDescription L64](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetDescription.php#L64)：`DB::table('transaction_journals')->where('id', ...)->update(...)`
- [SetBudget L98-L102](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetBudget.php#L98-L102)
- [SetDestinationAccount L127](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetDestinationAccount.php#L127)
- [ConvertToWithdrawal L166-L173](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/ConvertToWithdrawal.php#L166-L173)

`DB::table()` 走的是 Laravel Query Builder，**绕开 Eloquent ORM**，所以：
- ❌ 不会触发任何 Model 事件（`saving`、`saved`、`updating`、`updated`）
- ❌ 自然不会广播 `UpdatedSingleTransactionGroup`

**写法 B（少数动作）：Eloquent Model 的 `->save()`**

目前代码中只有 3 个动作使用了 `->save()`：

- [SetNotes L56](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetNotes.php#L56)：`$dbNote->save()`（操作 Note 模型）
- [SwitchAccounts L97-L98](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SwitchAccounts.php#L97-L98)：`$sourceTransaction->save()` 和 `$destTransaction->save()`（操作 Transaction 模型）
- [SetAmount L106](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetAmount.php#L106)：`$transaction->save()` + `$object->transactionGroup->touch()`

这些 `->save()` **会触发对应 Model 的 Observer**，但 Observer 里只做本位币金额换算等数据维护工作，**完全不会广播 `UpdatedSingleTransactionGroup`**。也就是说，即使动作用了 Eloquent 写法，也不会间接引发规则引擎的二次执行。

所以结论是：
- ✅ `UpdatedSingleTransactionGroup` 只来自业务层显式的 `event(...)` 调用
- ✅ 动作执行完后这条事件是 **SearchRuleEngine 自己在 fireStrictRule / fireNonStrictRule 的末尾主动发出来的**
- ✅ 发的时候 `flags->applyRules` 已经被设为 `false`

**两个需要先讲准的关键事实**：

#### 事实一：`UpdatedSingleTransactionGroup` 是业务层手动广播的，不是 Eloquent Model 事件自动触发的

在整个代码库里，该事件共被显式 `event(new UpdatedSingleTransactionGroup(...))` 广播了 **9 处**，主要来源：

| 广播位置 | 场景 | applyRules 默认值 |
|----------|------|-------------------|
| [SearchRuleEngine::fireStrictRule() L466](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L466) | 规则引擎严格模式执行完 | **false**（引擎显式设置） |
| [SearchRuleEngine::fireNonStrictRule() L415](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L415) | 规则引擎非严格模式执行完 | **false**（引擎显式设置） |
| [TransactionGroupRepository::store() L368](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php#L368) | 用户新建交易（这是 `CreatedSingleTransactionGroup`） | `$data['apply_rules'] ?? true` |
| [AccountServiceTrait L675](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Services/Internal/Support/AccountServiceTrait.php#L675) | 账户服务内部变更 | **false** |
| [MassController L282](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Http/Controllers/Transaction/MassController.php#L282) | 批量编辑 | true（默认构造） |
| [ConvertController L170](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Http/Controllers/Transaction/ConvertController.php#L170) | 转换交易类型 | true（默认构造） |
| [BulkController L124](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Http/Controllers/Transaction/BulkController.php#L124) | 批量操作 | 取决于请求参数 |
| [CorrectsGroupAccounts L71](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Console/Commands/Correction/CorrectsGroupAccounts.php#L71) | 修复命令 | **false** |
| [UpdateController (API) L97](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Api/V1/Controllers/Models/Transaction/UpdateController.php#L97) | API 更新交易 | 取决于请求参数 `applyRules` |

`UpdatedSingleTransactionGroup` **不跟 Eloquent ORM 的 `saving` / `saved` / `updating` / `updated` 事件挂钩**——它完全是业务层自己控制什么时候发。

#### 事实二：动作本身不会直接触发 `UpdatedSingleTransactionGroup`

动作修改数据库分两种写法，但**无论哪种都不会直接产生 `UpdatedSingleTransactionGroup` 事件**：

**写法 A（绝大多数动作）：`DB::table()->update/insert/delete`**

例如：
- [SetCategory L92-L96](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetCategory.php#L92-L96)：`DB::table('category_transaction_journal')->...`
- [SetDescription L64](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetDescription.php#L64)：`DB::table('transaction_journals')->where('id', ...)->update(...)`
- [SetBudget L98-L102](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetBudget.php#L98-L102)
- [SetDestinationAccount L127](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetDestinationAccount.php#L127)
- [ConvertToWithdrawal L166-L173](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/ConvertToWithdrawal.php#L166-L173)

`DB::table()` 走的是 Laravel Query Builder，**绕开 Eloquent ORM**，所以：
- ❌ 不会触发任何 Model 事件（`saving`、`saved`、`updating`、`updated`）
- ❌ 自然不会广播 `UpdatedSingleTransactionGroup`

**写法 B（少数动作）：Eloquent Model 的 `->save()`**

目前代码中只有 3 个动作使用了 `->save()`：

- [SetNotes L56](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetNotes.php#L56)：`$dbNote->save()`（操作 Note 模型）
- [SwitchAccounts L97-L98](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SwitchAccounts.php#L97-L98)：`$sourceTransaction->save()` 和 `$destTransaction->save()`（操作 Transaction 模型）
- [PrependNotes L62](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/PrependNotes.php)、[AppendNotes L65](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/AppendNotes.php)：操作 Note 模型

这些 `->save()` **会触发对应 Model 的 Observer**（例如 Transaction 模型通过 `#[ObservedBy([TransactionObserver::class])]` 注册了观察者）。但 [TransactionObserver::updated()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Handlers/Observer/TransactionObserver.php#L44-L47) 只做一件事——把金额换算成本位币金额（`updatePrimaryCurrencyAmount`），**完全不会广播 `UpdatedSingleTransactionGroup`**。

所以结论是：
- ✅ `UpdatedSingleTransactionGroup` 只来自业务层显式的 `event(...)` 调用
- ✅ 动作执行完后这条事件是 **SearchRuleEngine 自己在 fireStrictRule / fireNonStrictRule 的末尾主动发出来的**
- ✅ 发的时候 `flags->applyRules` 已经被设为 `false`

### 11.2 核心角色：TransactionGroupEventFlags

定义文件：[TransactionGroupEventFlags.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Events/Model/TransactionGroup/TransactionGroupEventFlags.php)

```php
class TransactionGroupEventFlags
{
    // ★ 关键：默认是 true，表示"应当执行规则"
    public bool $applyRules        = true;
    
    public bool $fireWebhooks      = true;
    public bool $batchSubmission   = false;
    public bool $recalculateCredit = true;
    public bool $unifyOnly         = false;
}
```

这是一个简单的 DTO（数据传输对象），作为**参数**随事件对象一起传递给监听器。监听器在处理前会先检查 `flags->applyRules` 的值。

### 11.3 完整防循环链路

#### 第一层：规则引擎在 fire 时主动置 false

在 [fireStrictRule()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L449-L477) 和 [fireNonStrictRule()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L402-L419) 中：

```php
private function fireStrictRule(Rule $rule): bool
{
    // ★ 步骤1：创建 flags，并主动把 applyRules 置为 false！
    $flags               = new TransactionGroupEventFlags();
    $flags->applyRules   = false;   // ← 关键：告诉后续监听器不要再跑规则
    $flags->fireWebhooks = false;   // 同时也不触发 Webhook（副作用收敛）
    
    $objects             = new TransactionGroupEventObjects();
    $collection          = $this->findStrictRule($rule);

    // ★ 步骤2：执行动作，直接用 SQL 修改数据库（不走 Model save）
    $this->processResults($rule, $collection);

    // ★ 步骤3：触发更新事件，但传入的是 applyRules=false 的 flags！
    $objects->collectFromCollection($collection);
    event(new UpdatedSingleTransactionGroup($flags, $objects));
    //                          ↑↑↑↑↑
    //               带着 applyRules=false 广播事件
    
    return $collection->count() > 0;
}
```

**非严格模式完全一致**：
```php
private function fireNonStrictRule(Rule $rule): bool
{
    $flags               = new TransactionGroupEventFlags();
    $flags->applyRules   = false;   // 同样置 false
    $flags->fireWebhooks = false;
    // ...
    event(new UpdatedSingleTransactionGroup($flags, $objects));
}
```

#### 第二层：监听器接收到事件后检查 flags

在 [ProcessesUpdatedTransactionGroup::handle()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesUpdatedTransactionGroup.php#L40-L75)：

```php
public function handle(UpdatedSingleTransactionGroup $event): void
{
    // ★ 步骤A：检查 flags 开关，若为 false 则跳过规则处理
    if (!$event->flags->applyRules) {
        Log::debug(sprintf('Will NOT process rules for %d journal(s)', 
            $event->objects->transactionJournals->count()));
        // ↑↑↑ 打日志后跳过 processRules()
    }
    
    // 其他清理工作（如重算余额、删统计缓存）仍然会执行
    
    // ★ 步骤B：只有 applyRules=true 时才执行 processRules
    if ($event->flags->applyRules) {
        $this->processRules($event->objects->transactionJournals, 'update-journal');
    }
    
    // 下面的工作照常进行（不受 applyRules 影响）：
    if ($event->flags->recalculateCredit) { /* 重算信用 */ }
    if ($event->flags->fireWebhooks)     { /* 触发 Webhook */ }
    $this->removePeriodStatistics($event->objects);
    $this->recalculateRunningBalance($event->objects);
}
```

同理，[ProcessesNewTransactionGroup::handle()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesNewTransactionGroup.php#L53-L65) 也是完全同样的判断逻辑。

### 11.4 防循环时序图

```
用户手动更新交易
    │
    ├─ 构造事件: flags.applyRules = true（默认）
    └─ 广播 UpdatedSingleTransactionGroup(flags=true)
           │
           ▼
    ProcessesUpdatedTransactionGroup 收到事件
           │ flags.applyRules = true ✓
           ▼
    调用 processRules() 启动规则引擎
           │
           ▼
    SearchRuleEngine::fireStrictRule()
           │
           ├─ ① 创建新 flags: applyRules = false ✗
           ├─ ② processResults() → 动作用 SQL 修改 DB（SET category=X）
           │        ↑ 直接 DB 操作，不走 Model 事件
           │
           └─ ③ 广播 UpdatedSingleTransactionGroup(flags=false ✗)
                    │
                    ▼
              ProcessesUpdatedTransactionGroup 再次收到事件
                    │ flags.applyRules = false ✗
                    ▼
              ★ 不会再调用 processRules()！循环被切断！
              
              仍然会执行：
              ├─ recalculateCredit（重算信用）
              ├─ removePeriodStatistics（清统计缓存）
              └─ recalculateRunningBalance（重算余额）
```

### 11.5 辅助防线：动作层的幂等性设计

除了 flags 机制外，每个动作实现也都有**幂等检查**，构成第二道防线：

以 [SetCategory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetCategory.php#L86-L90) 为例：

```php
// 检查旧分类是否与新分类相同
if ((int)$oldCategory?->id === $category->id) {
    // 已经是目标分类，返回 false 表示"未修改"
    return false;
}
```

所有动作都有类似逻辑：
- [SetDescription.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetDescription.php)：比较 before/after 字符串是否相同
- [SetDestinationAccount.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetDestinationAccount.php)：比较旧账户ID与新账户ID
- [AddTag.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/AddTag.php)：检查标签是否已存在

**双重保险**：
1. **第一道（flags）**：从事件源头上不让规则引擎被二次调用
2. **第二道（幂等）**：即使被调用了，动作层也不会重复修改

### 11.6 不同动作触发方式的对比

| 触发场景 | applyRules 初始值 | 是否二次触发事件 | 是否重复执行规则 |
|----------|-------------------|------------------|------------------|
| 用户创建交易 | true（默认） | 是（动作广播事件） | **否**（动作广播时设为 false） |
| 用户更新交易 | true（默认） | 是（动作广播事件） | **否**（同上） |
| 规则引擎动作修改 | **false**（引擎显式设置） | 是（动作广播事件） | **否**（监听器判断为 false） |
| 控制台命令手动触发 | 取决于命令参数 | 视命令实现而定 | 通常命令自己控制循环 |

### 11.7 关键设计要点

1. **事件参数透传而非全局状态**：`applyRules` 不存 Session 或静态变量，而是作为事件对象的字段传递，天然线程安全
2. **主动置 false 而非默认跳过**：规则引擎在自己触发事件前**主动**把标志设为 false，语义明确
3. **只阻断规则，不阻断其他流程**：清缓存、重算余额等清理工作仍然执行，避免了跳过规则导致的数据不一致
4. **两道防线纵深防御**：flags 机制（第一道）+ 动作幂等（第二道），确保在各种边界情况下都不会出现死循环

---

## 十二、匹配顺序（Order）的完整应用

### 12.1 Order 是整个链路的核心排序键

从规则组加载到动作执行的每一层，`order` 字段都决定了"谁先谁后"。这也是冲突选优（第六章）的物理基础。

### 12.2 四层 Order 在 DB 查询时的应用

所有层的排序都在 **Repository 层加载数据时** 通过 SQL `ORDER BY` 完成，确保内存中集合顺序就是执行顺序。

#### 第 1 层：规则组的 order

代码位置：[RuleGroupRepository::getRuleGroupsWithRules()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Repositories/RuleGroup/RuleGroupRepository.php#L220-L236)

```php
$groups = $this->user
    ->ruleGroups()
    ->orderBy('order', 'ASC')       // ← 规则组按 order 正序
    ->where('active', true)
    ->with([...])                   // 同时预加载子级数据
    ->get();
```

类似的加载点还有：
- [getActiveGroups()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Repositories/RuleGroup/RuleGroupRepository.php#L126-L134) → `orderBy('order', 'ASC')`
- [get()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Repositories/RuleGroup/RuleGroupRepository.php#L121-L124) → `orderBy('order', 'ASC')`

#### 第 2 层：规则的 order

在同一个 `with()` 闭包中对子集排序：

```php
->with([
    'rules' => static function (HasMany $query): void {
        $query->orderBy('order', 'ASC');       // ← 组内规则按 order 正序
        $query->where('rules.active', true);
    },
```

如果规则组没有使用 `with()` 预加载，SearchRuleEngine 内部也会重新加载时排序：

[fireGroup() L378-L381](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L378-L381)：
```php
$rules = $group
    ->rules()
    ->orderBy('rules.order', 'ASC')      // ← 再次保证 order 正序
    ->where('rules.active', true)
    ->get(['rules.*']);
```

#### 第 3 层：触发器的 order

同样在 Repository 的 `with()` 中定义：

```php
'rules.ruleTriggers' => static function (HasMany $query): void {
    $query->orderBy('order', 'ASC');        // ← 触发器按 order 正序
},
```

SearchRuleEngine 内部加载时也排序：

[findNonStrictRule() L213](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L213)：
```php
$triggers = $rule->ruleTriggers()->orderBy('order', 'ASC')->get();
```

[findStrictRule() L309](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L309)：
```php
$triggers = $rule->ruleTriggers()->orderBy('order', 'ASC')->get();
```

#### 第 4 层：动作的 order

在 Repository 的 `with()` 中定义：

```php
'rules.ruleActions' => static function (HasMany $query): void {
    $query->orderBy('order', 'ASC');        // ← 动作按 order 正序
},
```

SearchRuleEngine 内部执行时同样排序：

[processTransactionJournal() L577](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L577)：
```php
$actions = $rule->ruleActions()->orderBy('order', 'ASC')->get();
```

### 12.3 Order 在循环执行时的实际效果

以一条包含 3 个规则组的完整链路为例：

```
SQL 查询层（ORDER BY order ASC 已排好）
    │
    ▼
内存中拿到的 Collection
    │
    ▼
RuleGroup(order=1) "默认规则组"
    │
    ├─ Rule(order=1) "超市消费"
    │    ├─ RuleTrigger(order=1): description_contains "永辉"
    │    ├─ RuleTrigger(order=2): source_account_is "招行卡"
    │    └─ RuleAction(order=1): set_category "餐饮"
    │         RuleAction(order=2): add_tag "日常消费"
    │         RuleAction(order=3, stop_processing=true): set_budget "伙食费" ← 最后一个并中断
    │
    ├─ Rule(order=2, stop_processing=true) "大额转账" ← 如果触发，组内不再往下
    │    └─ ...
    │
    └─ Rule(order=3) "工资收入"（可能因 Rule#2 触发而永远执行不到）
         └─ ...
    │
RuleGroup(order=2) "高级规则组"
    └─ ...
    │
RuleGroup(order=3) "清理规则组"
    └─ ...
```

**执行顺序严格等于 order 升序**：
1. 第 1 层：按 rule_group.order 从小到大遍历组
2. 第 2 层：按 rule.order 从小到大遍历组内规则（触发且 stop_processing=true 则跳出）
3. 第 3 层：按 trigger.order 从小到大尝试匹配（非严格模式下支持提前中断）
4. 第 4 层：按 action.order 从小到大执行动作（修改成功且 stop_processing=true 则跳出）

### 12.4 Order 的维护

RuleGroupRepository 提供了重排方法：[correctRuleGroupOrder()](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Repositories/RuleGroup/RuleGroupRepository.php#L44-L69)

```php
public function correctRuleGroupOrder(): void
{
    $set = $this->user
        ->ruleGroups()
        ->orderBy('order', 'ASC')    // 先按当前 order 取出
        ->orderBy('active', 'DESC')  // 激活的排前面
        ->orderBy('title', 'ASC')    // 再按标题
        ->get(['rule_groups.id']);
    
    $index = 1;
    foreach ($set as $item) {
        RuleGroup::where('id', $item->id)->update(['order' => $index++]);
        // 重新写入连续的 order 值
    }
}
```

### 12.5 Order 与 stop_processing 的配合

`order` 决定**顺序**，`stop_processing` 决定**是否中断**，两者组合实现完整的控制流：

| 场景 | order 作用 | stop_processing 作用 |
|------|-----------|----------------------|
| 优先级高的规则先匹配 | 把常用规则设为 order=1 | 匹配后设为 true，避免浪费计算 |
| "兜底"规则最后执行 | 设为最大 order 值 | 设为 false，前面的先跑完 |
| 条件匹配时找到就停 | 先写命中率高的 trigger（非严格） | trigger 设为 true，停止尝试其他条件 |
| 动作链中修改完即止 | 最核心的 action 放最前 | action 设为 true，避免后续覆盖 |

---

## 十三、参考文件清单（扩展）

| 文件路径 | 说明 |
|----------|------|
| [app/TransactionRules/Engine/SearchRuleEngine.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php) | 规则引擎核心实现（含 addOperator/removeOperator/fireStrictRule/fireNonStrictRule） |
| [app/TransactionRules/Engine/RuleEngineInterface.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/RuleEngineInterface.php) | 规则引擎接口 |
| [app/Listeners/Model/TransactionGroup/SupportsGroupProcessingTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Listeners/Model/TransactionGroup/SupportsGroupProcessingTrait.php) | processRules() 循环注入 journal_id 的入口 |
| [app/Listeners/Model/TransactionGroup/ProcessesNewTransactionGroup.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesNewTransactionGroup.php) | 新交易事件监听器（检查 flags） |
| [app/Listeners/Model/TransactionGroup/ProcessesUpdatedTransactionGroup.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesUpdatedTransactionGroup.php) | 更新交易事件监听器（检查 flags） |
| [app/Events/Model/TransactionGroup/TransactionGroupEventFlags.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Events/Model/TransactionGroup/TransactionGroupEventFlags.php) | 事件标志位（applyRules/fireWebhooks 等） |
| [app/Repositories/RuleGroup/RuleGroupRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Repositories/RuleGroup/RuleGroupRepository.php) | 规则组仓库（含四层 ORDER BY） |
| [app/Models/Rule.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Models/Rule.php) | 规则模型 |
| [app/Models/RuleGroup.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Models/RuleGroup.php) | 规则组模型 |
| [app/Models/RuleTrigger.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Models/RuleTrigger.php) | 触发器模型 |
| [app/Models/RuleAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Models/RuleAction.php) | 动作模型 |
| [app/TransactionRules/Factory/ActionFactory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Factory/ActionFactory.php) | 动作工厂 |
| [app/TransactionRules/Actions/ActionInterface.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/ActionInterface.php) | 动作接口 |
| [app/TransactionRules/Actions/SetCategory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/SetCategory.php) | 动作示例（含幂等检查） |
| [app/Support/Search/OperatorQuerySearch.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Support/Search/OperatorQuerySearch.php) | 搜索实现（journal_id case） |
| [app/Helpers/Collector/GroupCollector.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Helpers/Collector/GroupCollector.php) | 查询收集器（setJournalIds → whereIn） |
| [config/search.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/config/search.php) | 搜索操作符配置 |
| [config/firefly.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/config/firefly.php) | 动作类型配置 |

---

## 十四、参考文件清单

| 文件路径 | 说明 |
|----------|------|
| [app/TransactionRules/Engine/SearchRuleEngine.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php) | 规则引擎核心实现 |
| [app/TransactionRules/Engine/RuleEngineInterface.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Engine/RuleEngineInterface.php) | 规则引擎接口 |
| [app/Models/Rule.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Models/Rule.php) | 规则模型 |
| [app/Models/RuleGroup.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Models/RuleGroup.php) | 规则组模型 |
| [app/Models/RuleTrigger.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Models/RuleTrigger.php) | 触发器模型 |
| [app/Models/RuleAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Models/RuleAction.php) | 动作模型 |
| [app/TransactionRules/Factory/ActionFactory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Factory/ActionFactory.php) | 动作工厂 |
| [app/TransactionRules/Actions/ActionInterface.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/TransactionRules/Actions/ActionInterface.php) | 动作接口 |
| [app/Support/Search/OperatorQuerySearch.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/app/Support/Search/OperatorQuerySearch.php) | 搜索实现 |
| [config/search.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/config/search.php) | 搜索操作符配置 |
| [config/firefly.php](file:///d:/fz/0601-1/solo-dogfeeding/code/25-firefly-iii/config/firefly.php) | 动作类型配置 |
