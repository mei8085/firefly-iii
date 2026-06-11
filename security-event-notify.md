# Firefly III 安全事件：触发、发现、持久化与通知分发协作机制

本文档详细梳理安全相关事件从**触发** → **事件发现与绑定** → **记录持久化** → **通知分发**的完整协作链路。

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
  │ (Controller 直写)│ │ (Listener 间写)      │ │ (Listener 触发)   │
  │ Log::channel     │ │ ALERepository       │ │ NotificationSender│
  │ ('audit')        │ │ ::store()           │ │ ::send()          │
  └──────────────────┘ └─────────────────────┘ └───────────────────┘
```

**关键分工**：审计日志写入由**控制器直接调用** `Log::channel('audit')` 完成（与事件监听器无关），通知分发由**监听器**触发。

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

### 2.2 核心发现：withEvents(discover:) 才是真正入口

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

### 2.3 发现规则的完整执行流程

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

### 3.1 审计日志不由监听器写入，由控制器直写

审计日志的写入**发生在事件触发之前**（或在事件触发的同一个控制器方法中），是通过 `Log::channel('audit')` **直接调用**完成的，**不经过监听器**。

**示例** — `app/Http/Controllers/Auth/LoginController.php` 登录流程：

```php
// 第89行：尝试登录前就写审计
Log::channel('audit')->info(sprintf('User is trying to login using "%s"', $username));

// 第107行：被锁定也写审计
Log::channel('audit')->warning(sprintf('Login for user "%s" was locked out.', ...));

// 第114行：登录成功也写审计
Log::channel('audit')->info(sprintf('User "%s" has been logged in.', ...));
event(new UserSuccessfullyLoggedIn(...));  // ← 事件触发在日志之后

// 第141行：登录失败也写审计
Log::channel('audit')->warning(sprintf('Login failed. Attempt for user "%s" failed.', ...));
event(new UserFailedLoginAttempt(...));     // ← 事件触发在日志之后
```

**示例** — `app/Http/Controllers/Auth/TwoFactorController.php` MFA 验证：

```php
// 第89行：MFA失败计数审计
Log::channel('audit')->info(sprintf('User "%s" has had %d failed MFA attempts.', ...));
event(new UserKeepsFailingMFA(...));  // ← 审计日志先于事件

// 第117行：使用备用码审计
Log::channel('audit')->info(sprintf('User "%s" has used a backup code.', ...));
event(new UserHasUsedBackupCode(...));  // ← 审计日志先于事件
```

### 3.2 AuditProcessor 上下文注入时机

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

**关键细节**：`AuditProcessor` 在每条日志写入**之前**注入上下文，它通过 `auth()->check()` 判断用户是否已登录，通过 `request()->ip()` 获取 IP，通过 `request()->method()` 和 `request()->url()` 获取请求信息。这意味着审计日志的上下文是**实时捕获**的，反映的是日志写入那一刻的请求状态。

### 3.3 审计日志通道选择

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

### 3.4 数据库审计表（仅交易规则变更）

`audit_log_entries` 数据库表**仅用于交易规则变更追踪**（由 `StoresAuditLogEntry` 监听器写入），用户安全事件**不写入此表**。

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
       │     ↑ 注意：这是 Passport 包内部触发的事件，非 Firefly 代码
       │
步骤3  NotifiesUserAboutNewAccessToken（同步执行！）
       app/Listeners/Security/User/NotifiesUserAboutNewAccessToken.php#L32-L43
       │  ※ 未实现 ShouldQueue 接口 → 在当前请求周期内同步执行
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

`NotifiesUserAboutNewAccessToken` 是安全监听器中**唯一未实现 `ShouldQueue`** 的监听器，它**同步执行**：

| 监听器 | 实现 ShouldQueue? | 执行方式 |
|--------|------------------|---------|
| `NotifiesUserAboutFailedLogin` | ✅ | 可异步 |
| `NotifiesUserAboutEnabledMFA` | ✅ | 可异步 |
| `NotifiesUserAboutDisabledMFA` | ✅ | 可异步 |
| `NotifiesUserAboutUsedBackupCode` | ✅ | 可异步 |
| `NotifiesUserAboutNewBackupCodes` | ✅ | 可异步 |
| `NotifiesUserAboutFewCodesLeft` | ✅ | 可异步 |
| `NotifiesUserAboutNoCodesLeft` | ✅ | 可异步 |
| `NotifiesUserAboutRepeatedMFAFailures` | ✅ | 可异步 |
| `NotifiesUserAboutNewIpAddress` | ✅ | 可异步 |
| `StoresNewIpAddress` | ✅ | 可异步 |
| `RespondsToNewLogin` | ✅ | 可异步 |
| `SendsUserNewPassword` | ✅ | 可异步 |
| `HandlesChangeOfUserEmailAddress` | ✅ | 可异步 |
| `NotifiesOwnerAboutUnknownUser` | ✅ | 可异步 |
| `HandlesNewUserRegistration` | ✅ | 可异步 |
| `NotifiesOwnerAboutNewVersion` | ✅ | 可异步 |
| `ChecksForNewVersion` | ✅ | 可异步 |
| `NotifiesAboutNewInvitation` | ✅ | 可异步 |
| **`NotifiesUserAboutNewAccessToken`** | **❌** | **同步** |

### 4.3 队列配置对异步的实际影响

即使监听器实现了 `ShouldQueue`，**实际是否异步取决于队列驱动配置**：

```php
// config/queue.php 第37行
'default' => env('QUEUE_CONNECTION', 'sync'),
```

| `QUEUE_CONNECTION` 值 | ShouldQueue 监听器行为 | NotifiesUserAboutNewAccessToken 行为 |
|----------------------|----------------------|-------------------------------------|
| `sync`（默认） | **同步执行**（派发后立即在同进程执行） | 同步执行 |
| `database` | 异步执行（写入 jobs 表，由 queue worker 消费） | 同步执行 |
| `redis` | 异步执行（写入 Redis 队列） | 同步执行 |

**结论**：默认配置下所有监听器都是同步执行的。`NotifiesUserAboutNewAccessToken` 无论队列配置如何都**始终同步**执行——这意味着创建访问令牌时，通知发送会阻塞 HTTP 响应，用户需等待邮件/Slack/Pushover 发送完成后才能收到响应。

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

`ReturnsAvailableChannels::returnChannels($type, $user)` 根据类型和配置动态返回渠道列表：

**owner 类型**（管理员通知）— 从 `FireflyConfig` 读取系统级配置：

| 渠道 | 启用条件 | 配置来源 |
|------|---------|---------|
| `mail` | 始终启用 | `config('firefly.site_owner')` 邮箱 |
| `slack` | `notifications.channels.slack.enabled` + 有效 webhook URL | `FireflyConfig` 加密存储 |
| `pushover` | `notifications.channels.pushover.enabled` + 双 Token | `FireflyConfig` 加密存储 |

**user 类型**（用户通知）— 从 `Preferences` 读取用户级配置：

| 渠道 | 启用条件 | 配置来源 |
|------|---------|---------|
| `mail` | 始终启用（Demo站点除外） | 用户邮箱 |
| `slack` | 全局 Slack 开启 + 用户配置有效 webhook | `Preferences` 加密存储 |
| `pushover` | 全局 Pushover 开启 + 用户配置双 Token | `Preferences` 加密存储 |

### 5.3 通知可配置性

`config/notifications.php` 定义了所有通知类型的开关和可配置性：

```php
// config/notifications.php notifications.user 部分
'new_access_token'     => ['enabled' => true, 'configurable' => true],   // 用户可在UI关闭
'user_login'           => ['enabled' => true, 'configurable' => true],   // 用户可在UI关闭
'login_failure'        => ['enabled' => true, 'configurable' => true],   // 用户可在UI关闭
'enabled_mfa'          => ['enabled' => true, 'configurable' => false],  // 不可关闭
'disabled_mfa'         => ['enabled' => true, 'configurable' => false],  // 不可关闭
'new_password'         => ['enabled' => true, 'configurable' => false],  // 不可关闭
'few_left_mfa'         => ['enabled' => true, 'configurable' => false],  // 不可关闭
'no_left_mfa'          => ['enabled' => true, 'configurable' => false],  // 不可关闭
'many_failed_mfa'      => ['enabled' => true, 'configurable' => false],  // 不可关闭
'new_backup_codes'     => ['enabled' => true, 'configurable' => false],  // 不可关闭
```

- `configurable: true` — 用户可在偏好设置中关闭该类通知
- `configurable: false` — 安全关键通知，不允许用户关闭

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
 4                  │  审计日志（控制器直写，先于事件）
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
 4          └── 新IP && 用户开启通知通知？
                  event(new UserLoggedInFromNewIpAddress($user))
                       │
 5                  NotifiesUserAboutNewIpAddress [ShouldQueue]
                       ├── 读取 login_ip_history
                       ├── 遍历 notified=false 的条目
                       │     └── NotificationSender::send($user, new UserLogin())
                       ├── 全部标记 notified=true
                       └── Preferences::setForUser() 保存偏好
```

### 场景C：生成访问令牌（Passport 内部事件 + 同步监听器）

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
 3    NotifiesUserAboutNewAccessToken（★ 同步！无 ShouldQueue）
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
| `bootstrap/app.php` | 应用实例创建，**withEvents(discover:) 配置事件发现目录** |
| `bootstrap/providers.php` | 服务提供者注册（EventServiceProvider 已注释） |
| `app/Providers/EventServiceProvider.php` | 事件服务提供者（$listen 全注释，实际不生效） |

### 事件定义

| 目录 | 内容 |
|------|------|
| `app/Events/Security/User/` | 用户安全事件 12 种 |
| `app/Events/Security/System/` | 系统安全事件 5 种 |

### 监听器

| 目录 | 内容 |
|------|------|
| `app/Listeners/Security/User/` | 用户安全监听器 13 个 |
| `app/Listeners/Security/System/` | 系统安全监听器 6 个 |

### 持久化

| 文件 | 作用 |
|------|------|
| `config/logging.php` | 审计日志通道配置（audit_daily/audit_stdout 等） |
| `app/Support/Logging/AuditLogger.php` | 审计日志 tap 处理器，注入 AuditProcessor |
| `app/Support/Logging/AuditProcessor.php` | 自动注入 IP/用户/URL 上下文 |
| `app/Models/AuditLogEntry.php` | 数据库审计表模型（仅交易变更用） |
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

## 八、设计要点总结

### 8.1 本次修正的四处关键信息

| 原问题 | 修正后实际情况 | 复核依据 |
|--------|--------------|---------|
| 事件发现入口在 EventServiceProvider | 真正入口在 `bootstrap/app.php` 的 `withEvents(discover:)`，EventServiceProvider 已被注释 | `bootstrap/providers.php` 第50行 + `bootstrap/app.php` 第166-168行 |
| 绑定通过 `#[Subscribe]` 属性 | 绑定通过 `handle()` 方法参数类型声明完成 | 每个监听器的 `handle(EventClass $event)` 参数 |
| 所有安全监听器都异步 | `NotifiesUserAboutNewAccessToken` 未实现 `ShouldQueue`，始终同步执行 | `app/Listeners/Security/User/NotifiesUserAboutNewAccessToken.php` 第32行 |
| 链接使用本机绝对路径 | 使用仓库相对路径 | 本文档所有链接 |
| 审计日志通过监听器写入 | 审计日志由控制器**直写** `Log::channel('audit')`，先于事件触发 | LoginController/TwoFactorController/MfaController 中的 `Log::channel('audit')` 调用 |

### 8.2 架构设计特征

1. **启动配置即发现配置**：`bootstrap/app.php` 中 `withEvents(discover:)` 一行即完成事件绑定，`EventServiceProvider` 实际不生效
2. **审计与通知职责分离**：审计日志（文件）由控制器直写，通知分发由监听器触发，互不依赖
3. **默认同步执行**：队列默认 `sync` 驱动，所有 `ShouldQueue` 监听器实际同步执行，需配置 `database`/`redis` 驱动才真正异步
4. **令牌通知同步保证**：`NotifiesUserAboutNewAccessToken` 刻意不实现 `ShouldQueue`，确保令牌创建时通知立即发出
5. **安全通知不可关闭**：MFA 相关通知 `configurable: false`，用户无法在 UI 中关闭
6. **审计上下文实时捕获**：`AuditProcessor` 在日志写入前通过 `auth()`/`request()` 实时获取当前请求状态
