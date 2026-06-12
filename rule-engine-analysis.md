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

## 十、参考文件清单

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
