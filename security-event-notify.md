# Firefly III 安全事件：触发、发现、持久化与通知分发协作机制

本文档基于代码事实梳理安全相关事件从**触发** → **事件发现与绑定** → **记录持久化** → **通知分发**的完整协作链路。

---

## 一、整体架构全景

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           触发源 (Triggers)                                   │
│  ┌────────────────┐  ┌───────────────┐  ┌──────────────────┐  ┌────────────┐ │
│  │ LoginController│  │ MfaController │  │ OAuthController  │  │ Passport   │ │
│  │ TwoFactorCtrl  │  │               │  │                   │  │ (内部事件) │ │
│  └───────┬────────┘  └──────┬────────┘  └────────┬─────────┘  └──────┬─────┘ │
│          │ event()          │ event()            │ createToken()      │       │
└──────────┼──────────────────┼────────────────────┼─────────────────────┼───────┘
           │                  │                    │                     │
           ▼                  ▼                    ▼                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      Event 事件类 (数据载体)                                  │
│  ┌───────────────────────┐ ┌────────────────────────┐ ┌─────────────────────┐│
│  │FireflyIII\Events\...  │ │Illuminate\Auth\Events\ │ │Laravel\Passport\    ││
│  │(自定义安全事件 17种)   │ │Login (框架内置)        │ │Events\AccessTok..   ││
│  └───────────┬───────────┘ └────────────┬───────────┘ └──────────┬──────────┘│
└──────────────┼───────────────────────────┼─────────────────────────┼───────────┘
               │                           │                         │
               ▼                           ▼                         ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│    事件-监听器自动发现 (入口: bootstrap/app.php withEvents)                   │
│    扫描 app/Listeners → 解析 handle() 参数类型 → 建立 Event→Listener 映射    │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │
               ┌────────────────┼─────────────────────┐
               ▼                ▼                     ▼
  ┌──────────────────┐ ┌─────────────────────┐ ┌───────────────────┐
  │ 审计日志文件      │ │ 数据库审计表         │ │ 通知分发           │
  │ (多模块直写)      │ │ (Listener 间写)      │ │ (Listener 触发)   │
  │ Log::channel     │ │ ALERepository       │ │ NotificationSender│
  │ ('audit')        │ │ ::store()           │ │ ::send()          │
  └──────────────────┘ └─────────────────────┘ └───────────────────┘
```

---

## 二、事件发现入口与应用启动配置的对应关系

### 2.1 启动配置链

```
public/index.php
    │
    ▼ require
bootstrap/app.php                           ← 应用实例创建入口
    │
    ├── Application::configure(basePath: ...)
    │       ->withRouting(...)
    │       ->withMiddleware(...)
    │       ->withEvents(discover: [        ← ★ 事件发现的真实配置入口
    │               __DIR__ . '/../app/Listeners',
    │           ])
    │       ->withExceptions(...)
    │       ->create();
    │
    └── bootstrap/providers.php             ← 服务提供者注册
            │
            ├── AppServiceProvider::class
            ├── AuthServiceProvider::class
            ├── // EventServiceProvider::class   ← ★ 已被注释掉！
            ├── RouteServiceProvider::class
            └── ... (其他 ServiceProvider)
```

### 2.2 核心发现：withEvents(discover:) 是真正入口

在 `bootstrap/app.php` 第166-168行：

```php
->withEvents(discover: [
    __DIR__ . '/../app/Listeners',
])
```

这行配置告诉 Laravel 框架：**扫描 `app/Listeners` 目录下的所有类，自动发现事件-监听器映射**。

**同时注意**：`EventServiceProvider` 在 `bootstrap/providers.php` 第50行已被注释：

```php
// EventServiceProvider::class,
```

这意味着 `EventServiceProvider::$listen` 数组中的映射**完全不生效**（该数组本身也是全注释状态），事件绑定**完全依赖** `withEvents(discover:)` 的自动发现机制。

### 2.3 发现规则的执行流程

```
Laravel 框架启动
    │
    ▼ Application::configure()->withEvents(discover: ['...app/Listeners'])
    │
    ▼ bootDiscoverEvents()
    │
    ▼ discoverEvents()  (由 Illuminate\Foundation\Support\Providers\EventServiceProvider 基类提供)
    │
    ├── 1. 遍历 discover 指定的目录
    │      → app/Listeners/ 下所有 .php 文件
    │
    ├── 2. 对每个类使用反射
    │      → 找到 public function handle(XXX $event) 方法
    │      → 提取参数类型 XXX 的完整类名
    │
    └── 3. 建立映射
           XXX事件类 → 对应监听器类
           写入 bootstrap/cache/events.php (生产环境缓存)
```

### 2.4 绑定依据：handle() 参数类型声明

每个监听器通过 `handle()` 方法的**参数类型**声明自己监听哪个事件。下表列出三类事件来源的实例：

| 事件来源 | 事件类 | 监听器类 | 监听器文件 |
|---------|--------|---------|-----------|
| **Firefly 自定义** | `UserFailedLoginAttempt` | `NotifiesUserAboutFailedLogin` | `app/Listeners/Security/User/NotifiesUserAboutFailedLogin.php` |
| **Firefly 自定义** | `UserSuccessfullyLoggedIn` | `StoresNewIpAddress` | `app/Listeners/Security/User/StoresNewIpAddress.php` |
| **Firefly 自定义** | `UserHasEnabledMFA` | `NotifiesUserAboutEnabledMFA` | `app/Listeners/Security/User/NotifiesUserAboutEnabledMFA.php` |
| **Laravel 内置** | `Illuminate\Auth\Events\Login` | `RespondsToNewLogin` | `app/Listeners/Security/User/RespondsToNewLogin.php` |
| **Passport 内置** | `Laravel\Passport\Events\AccessTokenCreated` | `NotifiesUserAboutNewAccessToken` | `app/Listeners/Security/User/NotifiesUserAboutNewAccessToken.php` |

**复核方法**：打开任意监听器文件，查看 `public function handle(XXX $event)` 中 `XXX` 的 `use` 语句完整类名即被监听的事件。

---

## 三、审计日志记录细节

### 3.1 令牌创建路径无审计日志写入

`OAuthController::storePersonalAccessToken`（创建个人访问令牌的方法）中**没有** `Log::channel('audit')` 调用，令牌创建路径不直接写入审计日志。

```php
// app/Http/Controllers/Profile/OAuthController.php#L187-L194
public function storePersonalAccessToken(Request $request): JsonResponse
{
    $this->validation->make($request->only(['name']), [
        'name' => ['required', 'max:255'],
    ])->validate();

    return response()->json($request->user()->createToken($request->name));
}
```

### 3.2 审计日志调用者的完整分类（共100个文件）

全局搜索结果显示，`Log::channel('audit')` 调用分布在以下**七类**代码位置：

| 类别 | 目录路径 | 文件数 | 典型文件 | 典型用途 |
|------|---------|--------|---------|---------|
| **Web表单请求** | `app/Http/Requests/` | 41 | `*FormRequest.php`、`*StoreRequest.php`、`*UpdateRequest.php` | `withValidator()` 钩子中记录 Web 表单验证错误详情 |
| **API 请求** | `app/Api/V1/Requests/` | 17 | `Models/*/StoreRequest.php`、`UpdateRequest.php` | 记录 API V1 端点的表单验证错误 |
| **HTTP 控制器** | `app/Http/Controllers/` | ~29 | Auth 下 LoginController/TwoFactorController<br>Profile 下 MfaController<br>Admin、Account、Bill、Budget、Category 等目录下各类 CRUD 控制器<br>Webhooks、TransactionCurrency、Tag、Recurring、PiggyBank、Home 等控制器 | 安全操作日志（登录/登出/MFA）、<br>CRUD 操作记录、权限违规告警、Demo 用户操作拦截、访问页面记录 |
| **验证规则** | `app/Rules/` | 6 | `IsValidAmount.php`（3个变种）<br>`Admin/IsValidSlackUrl.php` 等（3个） | 自定义验证规则执行过程中记录金额/URL 等校验分支 |
| **仓储层** | `app/Repositories/` | 9 | Tag、Bill、Budget、Category、PiggyBank、Recurring、RuleGroup 等 Repository | `destroyAll()` 批量删除操作记录 |
| **Console 命令** | `app/Console/Commands/` | 1 | `System/ForcesMigrations.php` | 系统命令级别的迁移操作记录 |
| **工厂类** | `app/Factory/` | 1 | `AccountFactory.php` | 数据工厂生成记录 |

**合计**：100 个文件（数据来源：`app/` 目录下 `Log::channel('audit')` 全局搜索结果）

**调用时机说明**：安全相关操作的审计日志调用通常发生在 `event()` 触发**之前**（或同一方法内的邻近位置），与事件监听器无依赖关系。

以下是各类别的详细文件列表（按目录分组）：

- **app/Http/Requests/**（41个）：`TriggerRecurrenceRequest.php`, `UserFormRequest.php`, `UserRegistrationRequest.php`, `TestRuleFormRequest.php`, `TokenFormRequest.php`, `RuleGroupFormRequest.php`, `SelectTransactionsRequest.php`, `TagFormRequest.php`, `ReportFormRequest.php`, `RuleFormRequest.php`, `ProfileFormRequest.php`, `ReconciliationStoreRequest.php`, `RecurrenceFormRequest.php`, `PiggyBankUpdateRequest.php`, `PiggyBankStoreRequest.php`, `ObjectGroupFormRequest.php`, `CategoryFormRequest.php`, `ConfigurationRequest.php`, `CurrencyFormRequest.php`, `DeleteAccountFormRequest.php`, `EmailFormRequest.php`, `ExistingTokenFormRequest.php`, `InviteUserFormRequest.php`, `JournalLinkRequest.php`, `LinkTypeFormRequest.php`, `MassDeleteJournalRequest.php`, `MassEditJournalRequest.php`, `NewUserFormRequest.php`, `BillUpdateRequest.php`, `BudgetFormStoreRequest.php`, `BudgetFormUpdateRequest.php`, `BudgetIncomeRequest.php`, `BulkEditJournalRequest.php`, `BillStoreRequest.php`, `AttachmentFormRequest.php`, `AccountFormRequest.php` 等
- **app/Api/V1/Requests/**（17个）：`System/UserUpdateRequest.php`，以及 `Models/` 目录下 Account、AvailableBudget、Bill、Budget、BudgetLimit、PiggyBank、Recurrence、Rule、Transaction、TransactionLink 等实体的 `StoreRequest.php` / `UpdateRequest.php`
- **app/Http/Controllers/**（~29个）：Auth 下的 LoginController、TwoFactorController；Profile 下的 MfaController；Admin 下的 LinkController、NotificationController、ConfigurationController、HomeController；Account、Bill、Budget、Category、PiggyBank、Recurring、Tag 等目录下的 CreateController、EditController；以及 TransactionCurrency 下的三个控制器、Webhooks 下的四个控制器、HomeController
- **app/Rules/**（6个）：`IsValidZeroOrMoreAmount.php`、`IsValidPositiveAmount.php`、`IsValidAmount.php`、`Admin/IsValidSlackUrl.php`、`Admin/IsValidSlackOrDiscordUrl.php`、`Admin/IsValidDiscordUrl.php`
- **app/Repositories/**（9个）：`Tag/TagRepository.php`、`RuleGroup/RuleGroupRepository.php`、`Recurring/RecurringRepository.php`、`PiggyBank/PiggyBankRepository.php`、`Category/CategoryRepository.php`、`Budget/BudgetRepository.php`、`Budget/BudgetLimitRepository.php`、`Bill/BillRepository.php`、`Budget/AvailableBudgetRepository.php`
- **app/Console/Commands/**（1个）：`System/ForcesMigrations.php`
- **app/Factory/**（1个）：`AccountFactory.php`

### 3.3 AuditProcessor 上下文注入时机

审计日志通道在 `config/logging.php` 中配置，所有 `audit_*` 通道都通过 `tap` 挂载 `AuditLogger` 类：

```php
// config/logging.php 示例
'audit_daily' => [
    'driver' => 'daily',
    'path'   => storage_path('logs/ff3-audit.log'),
    'tap'    => [AuditLogger::class],   // ← 注入自定义处理器
    'days'   => 90,
],
```

**注入时机链路**：

```
Log::channel('audit')->info('消息')
    │
    ▼ Laravel 解析 'audit' 通道 (stack → audit_daily + audit_stdout)
    │
    ▼ audit_daily 通道初始化时，执行 tap: [AuditLogger::class]
    │
    ▼ AuditLogger::__invoke($logger)
    │   → 创建 AuditProcessor 实例
    │   → 为每个 Handler 设置自定义 LineFormatter
    │   → 为每个 Handler 推入 AuditProcessor
    │
    ▼ 日志写入时，AuditProcessor::__invoke($record) 被调用
    │
    ├── 已登录用户 → "AUDIT: {消息} ({IP} ({邮箱} -> {方法}:{URL})"
    └── 未登录用户 → "AUDIT: {消息} ({IP} -> {方法}:{URL})"
```

`AuditProcessor` 在每条日志写入前通过 `auth()->check()` 判断用户登录状态，通过 `request()->ip()`、`request()->method()`、`request()->url()` 获取请求信息，反映的是日志写入那一刻的请求状态。

### 3.4 审计日志通道选择

审计日志通道由环境变量 `AUDIT_LOG_CHANNEL` 控制：

```php
// config/logging.php 第38行
$auditLogChannel = (string) env('AUDIT_LOG_CHANNEL');

// 第47-49行：如果配置了有效通道名则使用，否则使用默认的 ['audit_daily', 'audit_stdout']
if (in_array($auditLogChannel, $validAuditChannels, true)) {
    $auditChannels = [$auditLogChannel];
}
```

| 可选通道 | 输出目标 | 保留时间 |
|---------|---------|---------|
| `audit_daily` (默认) | `storage/logs/ff3-audit.log` | 90天 |
| `audit_stdout` | 标准输出 | - |
| `audit_syslog` | 系统 syslog | - |
| `audit_papertrail` | Papertrail 远程日志 | - |
| `audit_errorlog` | PHP error_log | - |

### 3.5 数据库审计表（仅交易规则变更）

`audit_log_entries` 数据库表仅用于交易规则变更追踪（由 `StoresAuditLogEntry` 监听器写入），用户安全事件不写入此表。

---

## 四、生成访问令牌后的安全通知路径与异步处理细节

### 4.1 完整链路

```
步骤1  OAuthController::storePersonalAccessToken()
       app/Http/Controllers/Profile/OAuthController.php#L187-L194
       │
       │  $request->user()->createToken($request->name)
       │
步骤2  Laravel Passport 内部
       │  → 写入 oauth_access_tokens 表
       │  → event(new Laravel\Passport\Events\AccessTokenCreated($userId, $tokenId, $clientId))
       │
步骤3  NotifiesUserAboutNewAccessToken（未实现 ShouldQueue）
       app/Listeners/Security/User/NotifiesUserAboutNewAccessToken.php#L32-L43
       │
       │  $user = $repository->find((int) $event->userId);
       │  NotificationSender::send($user, new NewAccessToken());
       │
步骤4  NotificationSender::send()
       app/Notifications/NotificationSender.php#L36-L69
       │  → 读取用户语言偏好
       │  → NotificationFacade::locale($lang)->send($user, $notification)
       │
步骤5  NewAccessToken 通知类
       app/Notifications/User/NewAccessToken.php#L41-L102
       │  → via(): ReturnsAvailableChannels::returnChannels('user', $user)
       │  → toMail(): markdown('emails.token-created', [IP, UA, 时间])
       │  → toSlack(): SlackMessage
       │  → toPushover(): PushoverMessage
```

### 4.2 令牌通知的异步处理差异

`NotifiesUserAboutNewAccessToken` 未实现 `ShouldQueue` 接口。以下是所有19个安全监听器的实现状态：

**User 安全监听器（14个）**：

| 监听器类 | 实现 ShouldQueue? |
|---------|------------------|
| `NotifiesUserAboutFailedLogin` | ✅ |
| `NotifiesUserAboutEnabledMFA` | ✅ |
| `NotifiesUserAboutDisabledMFA` | ✅ |
| `NotifiesUserAboutUsedBackupCode` | ✅ |
| `NotifiesUserAboutNewBackupCodes` | ✅ |
| `NotifiesUserAboutFewCodesLeft` | ✅ |
| `NotifiesUserAboutNoCodesLeft` | ✅ |
| `NotifiesUserAboutRepeatedMFAFailures` | ✅ |
| `NotifiesUserAboutNewIpAddress` | ✅ |
| `StoresNewIpAddress` | ✅ |
| `RespondsToNewLogin` | ✅ |
| `SendsUserNewPassword` | ✅ |
| `HandlesChangeOfUserEmailAddress` | ✅ |
| **`NotifiesUserAboutNewAccessToken`** | **❌** |

**System 安全监听器（5个）**：

| 监听器类 | 实现 ShouldQueue? |
|---------|------------------|
| `NotifiesOwnerAboutUnknownUser` | ✅ |
| `HandlesNewUserRegistration` | ✅ |
| `NotifiesOwnerAboutNewVersion` | ✅ |
| `ChecksForNewVersion` | ✅ |
| `NotifiesAboutNewInvitation` | ✅ |

**总计**：19个安全监听器，18个实现了 `ShouldQueue`，`NotifiesUserAboutNewAccessToken` 未实现。

### 4.3 队列配置对异步的实际影响

即使监听器实现了 `ShouldQueue`，实际是否异步取决于队列驱动配置：

```php
// config/queue.php 第37行
'default' => env('QUEUE_CONNECTION', 'sync'),
```

| `QUEUE_CONNECTION` 值 | 实现 ShouldQueue 的监听器 | 未实现 ShouldQueue 的监听器 |
|----------------------|------------------------|--------------------------|
| `sync`（默认） | 同步执行（派发后立即在同进程执行） | 同步执行 |
| `database` | 异步执行（写入 jobs 表，由 queue worker 消费） | 同步执行 |
| `redis` | 异步执行（写入 Redis 队列） | 同步执行 |

默认配置下所有监听器均为同步执行。`NotifiesUserAboutNewAccessToken` 无论队列配置如何都同步执行——在当前配置下，这与其他监听器的行为没有区别。

---

## 五、通知分发机制

### 5.1 分发全链路

```
Listener::handle()
    │
    ▼ NotificationSender::send($user, new XxxNotification())
    │
    ├── 1. 读取用户语言偏好 (Preferences::getForUser($user, 'language'))
    │      如果是 OwnerNotifiable 则使用 config('firefly.default_language')
    │
    ├── 2. NotificationFacade::locale($lang)->send($user, $notification)
    │
    └── 3. 异常捕获
           ├── ClientException → Log::error()
           └── Exception (含 'Bcc' 或 'RFC 2822') → Log::warning()
```

### 5.2 渠道动态决策

#### Owner 类型（管理员通知）

从 `FireflyConfig` 读取系统级配置：

| 渠道 | 启用条件 | 配置来源 |
|------|---------|---------|
| `mail` | 始终启用 | `config('firefly.site_owner')` 邮箱 |
| `slack` | `config('notifications.channels.slack.enabled') === true` + 有效 webhook URL | `FireflyConfig` 加密存储 |
| `pushover` | `config('notifications.channels.pushover.enabled') === true` + `pushover_app_token` 非空 + `pushover_user_token` 非空 | `FireflyConfig` 加密存储 |

#### User 类型（用户通知）

从 `Preferences` 读取用户级配置：

| 渠道 | 启用条件 | 配置来源 |
|------|---------|---------|
| `mail` | 始终启用（Demo站点除外） | 用户邮箱 |
| `slack` | `config('notifications.channels.slack.enabled') === true` + 有效 webhook URL | `Preferences` 加密存储 |
| `pushover` | `config('notifications.channels.slack.enabled') === true` + `pushover_app_token` 非空 + `pushover_user_token` 非空 | `Preferences` 加密存储 |

**注意**：用户 Pushover 渠道的开关条件检查的是 `notifications.channels.slack.enabled` 配置（而非 `notifications.channels.pushover.enabled`）。相关代码见 `app/Notifications/ReturnsAvailableChannels.php#L116`：

```php
// returnUserChannels() 方法第116行
if (true === config('notifications.channels.slack.enabled', false)) {
    $pushoverAppToken  = (string) Preferences::getEncryptedForUser($user, 'pushover_app_token', '')->data;
    $pushoverUserToken = (string) Preferences::getEncryptedForUser($user, 'pushover_user_token', '')->data;
    if ('' !== $pushoverAppToken && '' !== $pushoverUserToken) {
        $channels[] = PushoverChannel::class;
    }
}
```

### 5.3 通知可配置性

`config/notifications.php` 定义了所有通知类型的开关和可配置性：

```php
// config/notifications.php notifications.user 部分
'new_access_token'     => ['enabled' => true, 'configurable' => true],
'user_login'           => ['enabled' => true, 'configurable' => true],
'login_failure'        => ['enabled' => true, 'configurable' => true],
'enabled_mfa'          => ['enabled' => true, 'configurable' => false],
'disabled_mfa'         => ['enabled' => true, 'configurable' => false],
'new_password'         => ['enabled' => true, 'configurable' => false],
'few_left_mfa'         => ['enabled' => true, 'configurable' => false],
'no_left_mfa'          => ['enabled' => true, 'configurable' => false],
'many_failed_mfa'      => ['enabled' => true, 'configurable' => false],
'new_backup_codes'     => ['enabled' => true, 'configurable' => false],
```

- `configurable: true` — 用户可在偏好设置中关闭该类通知
- `configurable: false` — 该类通知在配置中标记为不可由用户关闭

### 5.4 OwnerNotifiable 虚拟对象

系统级通知（如未知用户登录、新用户注册）不发送给真实 User 对象，而是发送给 `OwnerNotifiable` 虚拟对象：

```
app/Notifications/Notifiables/OwnerNotifiable.php
    │
    ├── routeNotificationFor('mail')     → config('firefly.site_owner')
    ├── routeNotificationForSlack()      → FireflyConfig::getEncrypted('slack_webhook_url')
    └── routeNotificationForPushover()   → PushoverReceiver(系统级 Token)
```

---

## 六、典型场景完整走查

### 场景A：用户登录失败（已知用户）

```
步骤  组件                                   操作
────  ─────────────────────────────────────  ──────────────────────────────────
 1    LoginController::login()               attemptLogin() 返回 false
       app/Http/Controllers/Auth/LoginController.php#L125-L134
       │
 2    查找用户                                $repository->findByEmail($username)
       │
 3    ├─ 未知用户 → event(new UnknownUserTriedLogin($username))
       │     └→ NotifiesOwnerAboutUnknownUser [ShouldQueue]
       │          → NotificationSender::send(new OwnerNotifiable, ...)
       │
       └─ 已知用户 → event(new UserFailedLoginAttempt($user))
                    │
 4                  │  审计日志（控制器直写）
                    │  Log::channel('audit')->warning('Login failed...')
                    │  app/Http/Controllers/Auth/LoginController.php#L141
                    │
 5                  └→ NotifiesUserAboutFailedLogin [ShouldQueue]
                         → NotificationSender::send($user, new UserFailedLoginAttempt())
                         → ReturnsAvailableChannels::returnChannels('user', $user)
                         → [mail, slack, pushover] (动态)
                         → toMail(): emails.security.failed-login
                             (含 IP、Hostname、UserAgent、时间)
```

### 场景B：新IP登录成功（事件链 + 偏好存储）

```
步骤  组件                                   操作
────  ─────────────────────────────────────  ──────────────────────────────────
 1    LoginController::login()
       │  event(new UserSuccessfullyLoggedIn($user))     ← 第121行
       │
 2    审计日志（控制器直写）
       │  Log::channel('audit')->info('User ... logged in')  ← 第114行
       │
 3    StoresNewIpAddress [ShouldQueue]
       │  ├── 读取 login_ip_history 偏好 (Preferences JSON数组)
       │  ├── IP已存在？刷新时间；新IP？追加 {ip, time, notified:false}
       │  ├── 清理 >6个月 的旧条目
       │  └── Preferences::setForUser() 保存偏好
       │       │
 4          └── 新IP && 用户开启通知？
                  event(new UserLoggedInFromNewIpAddress($user))
                       │
 5                  NotifiesUserAboutNewIpAddress [ShouldQueue]
                       ├── 读取 login_ip_history
                       ├── 遍历 notified=false 的条目
                       │     └── NotificationSender::send($user, new UserLogin())
                       ├── 全部标记 notified=true
                       └── Preferences::setForUser() 保存偏好
```

### 场景C：生成访问令牌（Passport 内部事件）

```
步骤  组件                                   操作
────  ─────────────────────────────────────  ──────────────────────────────────
 1    OAuthController::storePersonalAccessToken()
       app/Http/Controllers/Profile/OAuthController.php#L187-L194
       │  $request->user()->createToken($request->name)
       │
 2    Laravel Passport 内部
       │  → INSERT INTO oauth_access_tokens
       │  → event(new AccessTokenCreated($userId, $tokenId, $clientId))
       │
 3    NotifiesUserAboutNewAccessToken（未实现 ShouldQueue）
       app/Listeners/Security/User/NotifiesUserAboutNewAccessToken.php#L32-L43
       │  → $user = $repository->find((int) $event->userId)
       │  → NotificationSender::send($user, new NewAccessToken())
       │
 4    NewAccessToken 通知类
       app/Notifications/User/NewAccessToken.php#L41-L102
       │  → via(): ReturnsAvailableChannels::returnChannels('user', $user)
       │  → toMail(): markdown('emails.token-created', [IP, host, UA, time])
       │  → toSlack(): SlackMessage
       │  → toPushover(): PushoverMessage
       │
       ※ 通知发送完成后 OAuthController 才返回 JSON 响应
```

### 场景D：MFA 验证失败多次

```
步骤  组件                                   操作
────  ─────────────────────────────────────  ──────────────────────────────────
 1    TwoFactorController::submitMFA()
       app/Http/Controllers/Auth/TwoFactorController.php#L65-L126
       │  $authenticator->isAuthenticated() 返回 false
       │
 2    审计日志（控制器直写）
       │  Log::channel('audit')->info('MFA failure count is set to %d')
       │  ← addToMFAFailureCounter() 第132行
       │
 3    失败计数达 3 或 10？
       │  Log::channel('audit')->info('User has had %d failed MFA attempts')
       │  ← 第89行
       │  event(new UserKeepsFailingMFA($user, $counter))
       │
 4    NotifiesUserAboutRepeatedMFAFailures [ShouldQueue]
       → NotificationSender::send($user, new MFAManyFailedAttemptsNotification())
```

---

## 七、关键文件速查表

### 启动与发现

| 文件 | 作用 |
|------|------|
| `bootstrap/app.php` | 应用实例创建，`withEvents(discover:)` 配置事件发现目录 |
| `bootstrap/providers.php` | 服务提供者注册（EventServiceProvider 已注释） |
| `app/Providers/EventServiceProvider.php` | 事件服务提供者（$listen 全注释） |

### 事件定义

| 目录 | 内容 |
|------|------|
| `app/Events/Security/User/` | 用户安全事件 12 种 |
| `app/Events/Security/System/` | 系统安全事件 5 种 |

### 监听器

| 目录 | 内容 |
|------|------|
| `app/Listeners/Security/User/` | 用户安全监听器 14 个 |
| `app/Listeners/Security/System/` | 系统安全监听器 5 个 |

### 持久化

| 文件 | 作用 |
|------|------|
| `config/logging.php` | 审计日志通道配置 |
| `app/Support/Logging/AuditLogger.php` | 审计日志 tap 处理器，注入 AuditProcessor |
| `app/Support/Logging/AuditProcessor.php` | 自动注入 IP/用户/URL 上下文 |
| `app/Models/AuditLogEntry.php` | 数据库审计表模型（交易变更用） |
| `app/Repositories/AuditLogEntry/ALERepository.php` | 审计仓储 |
| `database/migrations/2022_10_01_210238_audit_log_entries.php` | 审计表迁移 |

### 通知分发

| 文件 | 作用 |
|------|------|
| `app/Notifications/NotificationSender.php` | 统一发送入口（本地化+异常处理） |
| `app/Notifications/ReturnsAvailableChannels.php` | 渠道动态决策器 |
| `app/Notifications/Notifiables/OwnerNotifiable.php` | 管理员虚拟通知对象 |
| `app/Notifications/Security/` | 安全通知类（MFA/登录失败等 8 种） |
| `app/Notifications/User/` | 用户通知类（新IP/新令牌等） |
| `app/Notifications/Admin/` | 管理员通知类（未知用户/新注册等） |
| `config/notifications.php` | 通知渠道与类型开关配置 |

### 队列配置

| 文件 | 作用 |
|------|------|
| `config/queue.php` | 队列驱动配置（默认 sync 同步） |

### 触发控制器

| 文件 | 触发事件 |
|------|---------|
| `app/Http/Controllers/Auth/LoginController.php` | UserSuccessfullyLoggedIn, UserFailedLoginAttempt, UnknownUserTriedLogin |
| `app/Http/Controllers/Auth/TwoFactorController.php` | UserHasUsedBackupCode, UserKeepsFailingMFA, UserHasFewMFABackupCodesLeft, UserHasNoMFABackupCodesLeft |
| `app/Http/Controllers/Profile/MfaController.php` | UserHasEnabledMFA, UserHasDisabledMFA, UserHasGeneratedNewBackupCodes |
| `app/Http/Controllers/Profile/OAuthController.php` | createToken() → Passport 触发 AccessTokenCreated |

---

## 八、代码事实总结

### 8.1 核实的关键事实

| 核查项 | 代码事实 | 复核依据 |
|--------|---------|---------|
| 事件发现入口 | `bootstrap/app.php` 的 `withEvents(discover:)`，EventServiceProvider 已被注释 | `bootstrap/providers.php` 第50行 + `bootstrap/app.php` 第166-168行 |
| 监听器绑定依据 | `handle()` 方法参数类型声明 | 每个监听器的 `handle(EventClass $event)` 参数 |
| 令牌创建审计日志 | 无 `Log::channel('audit')` 调用 | `app/Http/Controllers/Profile/OAuthController.php` 第187-194行 |
| 审计日志调用者 | 共100个文件，分布于七类代码位置：Web表单请求(41个)、API请求(17个)、HTTP控制器(~29个)、验证规则(6个)、仓储层(9个)、Console命令(1个)、Factory(1个) | app/ 目录下 `Log::channel('audit')` 全局搜索结果 |
| 用户 Pushover 开关条件 | 检查 `notifications.channels.slack.enabled` 配置，而非 `pushover.enabled` | `app/Notifications/ReturnsAvailableChannels.php` 第116行 |
| 安全监听器数量 | 总计 19 个（User 目录14个 + System 目录5个） | Glob 扫描结果 |
| NotifiesUserAboutNewAccessToken | 未实现 `ShouldQueue` 接口 | `app/Listeners/Security/User/NotifiesUserAboutNewAccessToken.php` 第32行 |
| 默认队列驱动 | `sync`（同步），所有监听器同步执行 | `config/queue.php` 第37行 |

### 8.2 代码实现特征

1. 事件绑定由 `bootstrap/app.php` 的 `withEvents(discover:)` 配置完成，`EventServiceProvider` 未注册
2. 审计日志写入调用共100个文件，分布于七类代码位置（Web表单请求、API请求、HTTP控制器、验证规则、仓储层、Console命令、Factory），与事件监听器无依赖关系
3. 令牌创建路径无直接审计日志写入，仅通过 `AccessTokenCreated` 事件触发通知
4. 19个安全监听器中18个实现 `ShouldQueue`，`NotifiesUserAboutNewAccessToken` 未实现
5. 默认 `QUEUE_CONNECTION=sync`，所有监听器同步执行
6. 用户 Pushover 渠道的开关条件检查的是 Slack 配置项
