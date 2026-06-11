# Firefly III 安全事件：触发、发现、持久化与通知分发协作机制

本文档详细梳理 Firefly III 中安全相关事件从**触发** → **事件-监听器绑定发现** → **记录持久化** → **通知分发**的完整协作链路，并补充三处关键机制的可复核说明。

---

## 一、整体架构全景

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           触发源 (Triggers)                                   │
│  ┌────────────────┐  ┌───────────────┐  ┌──────────────────┐  ┌────────────┐ │
│  │ LoginController│  │ MfaController │  │ OAuthController  │  │ Passport   │ │
│  │ (登录/登出)    │  │ (MFA管理)     │  │ (Token管理)      │  │ (内部事件) │ │
│  └───────┬────────┘  └──────┬────────┘  └────────┬─────────┘  └──────┬─────┘ │
│          │ event()          │ event()            │ event()             │       │
└──────────┼──────────────────┼────────────────────┼─────────────────────┼───────┘
           │                  │                    │                     │
           ▼                  ▼                    ▼                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      Event 事件类 (数据载体 DTO)                              │
│  ┌───────────────────────┐ ┌────────────────────────┐ ┌─────────────────────┐│
│  │FireflyIII\Events\...  │ │Illuminate\Auth\Events\ │ │Laravel\Passport\    ││
│  │(自定义安全事件 17种)   │ │Login (框架内置登录)    │ │Events\AccessTokenCr.││
│  └───────────┬───────────┘ └────────────┬───────────┘ └──────────┬──────────┘│
└──────────────┼───────────────────────────┼─────────────────────────┼───────────┘
               │                           │                         │
               ▼                           ▼                         ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│          事件-监听器自动发现与绑定 (核心机制，详见第二章)                      │
│                                                                              │
│   入口：EventServiceProvider 继承基类 → discoverEvents()                      │
│         扫描 app/Listeners 目录 → 解析 handle() 参数类型 → 建立 Event→Listener映射 │
│                                                                              │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      Listener 监听器 (异步队列 ShouldQueue)                   │
│  ┌───────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │ 1. 写入审计日志文件    │  │ 2. 写入数据库表      │  │ 3. 触发用户通知      │ │
│  │ Log::channel('audit') │  │ ALERepository::store│  │ NotificationSender  │ │
│  └───────────┬───────────┘  └──────────┬──────────┘  └──────────┬──────────┘ │
└──────────────┼───────────────────────────┼────────────────────────┼────────────┘
               │                           │                        │
               ▼                           ▼                        ▼
     ┌──────────────────┐       ┌─────────────────────┐   ┌───────────────────┐
     │ storage/logs/    │       │ audit_log_entries   │   │ Notification 管道 │
     │ ff3-audit.log    │       │ 表 (多态关联)        │   │ Mail/Slack/       │
     │ (90天保留)       │       │                     │   │ Pushover/...      │
     └──────────────────┘       └─────────────────────┘   └───────────────────┘
```

---

## 二、事件-监听器自动发现与绑定机制（可复核）

### 2.1 绑定入口：EventServiceProvider 继承链

Firefly III 并未在 [EventServiceProvider.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Providers/EventServiceProvider.php) 的 `$listen` 数组中显式配置事件映射（第36-50行全部被注释）。真正的绑定逻辑由**父类**提供：

```
FireflyIII\Providers\EventServiceProvider
        │
        │  extends
        ▼
Illuminate\Foundation\Support\Providers\EventServiceProvider
        │
        │  boot() → $this->bootDiscoverEvents()
        │         → discoverEvents()
        ▼
   自动发现机制
```

基类 `EventServiceProvider` 在 `boot()` 生命周期中调用 `discoverEvents()`，这是**所有监听器注册的真正入口**。

### 2.2 发现规则：目录扫描 + 参数类型推断

Laravel 框架的自动发现遵循以下可验证规则：

1. **扫描目录**：`app_path('Listeners')` — 即 Firefly III 的 [app/Listeners/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners)

2. **类过滤**：该目录下所有非抽象类均被视为候选监听器

3. **绑定依据**：解析每个监听器 `handle()` 方法的**参数类型声明**，该参数的类名就是监听的事件

**绑定依据示例** — 以 [NotifiesUserAboutNewAccessToken.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutNewAccessToken.php#L32-L43) 为例：

```php
class NotifiesUserAboutNewAccessToken
{
    //  ↓↓↓ 这里的参数类型就是"绑定依据" ↓↓↓
    public function handle(AccessTokenCreated $event): void
    //                      ↑↑↑ Laravel 解析到这里 → 建立映射：
    //                          AccessTokenCreated → NotifiesUserAboutNewAccessToken
    {
        $repository = app(UserRepositoryInterface::class);
        $user       = $repository->find((int) $event->userId);
        if (null !== $user) {
            NotificationSender::send($user, new NewAccessToken());
        }
    }
}
```

Laravel 通过反射读取 `handle()` 的参数类型 `Laravel\Passport\Events\AccessTokenCreated`，自动建立事件类到监听器类的映射关系。

### 2.3 三类事件来源的绑定实例对照

下表逐一列出三类事件及其监听器的绑定（均可通过阅读 `handle()` 参数类型复核）：

| 事件来源 | 事件类 | 监听器类 (handle 参数) | 所在文件 |
|---------|--------|----------------------|---------|
| **Firefly 自定义** | `UserFailedLoginAttempt` | `NotifiesUserAboutFailedLogin` | [Listeners/Security/User/NotifiesUserAboutFailedLogin.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutFailedLogin.php) |
| **Firefly 自定义** | `UserSuccessfullyLoggedIn` | `StoresNewIpAddress` | [Listeners/Security/User/StoresNewIpAddress.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/User/StoresNewIpAddress.php) |
| **Firefly 自定义** | `UserLoggedInFromNewIpAddress` | `NotifiesUserAboutNewIpAddress` | [Listeners/Security/User/NotifiesUserAboutNewIpAddress.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutNewIpAddress.php) |
| **Firefly 自定义** | `UserHasEnabledMFA` | `NotifiesUserAboutEnabledMFA` | [Listeners/Security/User/NotifiesUserAboutEnabledMFA.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutEnabledMFA.php) |
| **Firefly 自定义** | `UserHasUsedBackupCode` | `NotifiesUserAboutUsedBackupCode` | [Listeners/Security/User/NotifiesUserAboutUsedBackupCode.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutUsedBackupCode.php) |
| **Laravel 内置** | `Illuminate\Auth\Events\Login` | `RespondsToNewLogin` | [Listeners/Security/User/RespondsToNewLogin.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/User/RespondsToNewLogin.php#L30-L38) |
| **Passport 内置** | `Laravel\Passport\Events\AccessTokenCreated` | `NotifiesUserAboutNewAccessToken` | [Listeners/Security/User/NotifiesUserAboutNewAccessToken.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutNewAccessToken.php#L30-L42) |

### 2.4 验证方法（可复核）

要手动复核某个监听器到底监听哪个事件，只需两步：
1. 打开该监听器文件
2. 查看 `public function handle(XXX $event)` 中 `XXX` 的完整类名 —— 就是它监听的事件

---

## 三、生成访问令牌后的安全通知路径（可复核）

### 3.1 场景背景

用户在"个人设置 → OAuth → Personal Access Tokens"页面创建 API Token 时，系统需要通过邮件等渠道通知用户"有新的访问令牌被创建"，以防止令牌被恶意生成。

### 3.2 完整协作链路

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 步骤1：前端请求创建 Token                                                  │
│                                                                         │
│   [OAuthController::storePersonalAccessToken]                           │
│   file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/            │
│          app/Http/Controllers/Profile/OAuthController.php#L187-L194      │
│                                                                         │
│   public function storePersonalAccessToken(Request $request): JsonResponse│
│   {                                                                     │
│       $this->validation->make(...)->validate();                        │
│       return response()->json(                                          │
│           $request->user()->createToken($request->name)  ← 核心调用     │
│       );                                                                │
│   }                                                                     │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 步骤2：Laravel Passport 内部创建 Token 并触发事件                         │
│                                                                         │
│   User::createToken() (由 Laravel Passport HasApiTokens trait 提供)      │
│     → 创建 Token 记录到 oauth_access_tokens 表                          │
│     → 触发事件：Laravel\Passport\Events\AccessTokenCreated              │
│         携带参数：$userId, $tokenId, $clientId                           │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 步骤3：事件被自动发现 → 路由到监听器                                       │
│                                                                         │
│   [NotifiesUserAboutNewAccessToken]                                     │
│   file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/            │
│          app/Listeners/Security/User/NotifiesUserAboutNewAccessToken.php│
│                                                                         │
│   public function handle(AccessTokenCreated $event): void               │
│   {                                                                     │
│       $repository = app(UserRepositoryInterface::class);                │
│       $user       = $repository->find((int) $event->userId);  ← 通过事件 │
│       if (null !== $user) {                                                          中的 userId 查找 用户   │
│           NotificationSender::send($user, new NewAccessToken());  ← 发送 │
│       }                                                                             通知   │
│   }                                                                     │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 步骤4：NotificationSender 统一入口 → 多渠道分发                           │
│                                                                         │
│   [NotificationSender::send]                                            │
│   file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/            │
│          app/Notifications/NotificationSender.php#L36-L69               │
│                                                                         │
│   1. 读取用户语言偏好（本地化通知内容）                                    │
│   2. NotificationFacade::locale($lang)->send($user, $notification)      │
│   3. 捕获异常（Bcc格式错误、RFC 2822 无效等）并记录日志                    │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 步骤5：NewAccessToken 通知类 → 按渠道渲染消息                             │
│                                                                         │
│   [NewAccessToken]                                                       │
│   file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/            │
│          app/Notifications/User/NewAccessToken.php#L41-L102              │
│                                                                         │
│   via()       → ReturnsAvailableChannels::returnChannels('user', $user) │
│                   返回: [mail, slack, pushover] (根据配置动态)            │
│                                                                         │
│   toMail()    → markdown('emails.token-created', [                      │
│                   'ip', 'host', 'userAgent', 'time', 'link'             │
│                 ]) → 邮件模板含 IP、UA、时间                               │
│                                                                         │
│   toSlack()   → SlackMessage 内容                                        │
│   toPushover()→ PushoverMessage 推送                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 复核方法

1. 打开 [OAuthController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Http/Controllers/Profile/OAuthController.php#L187-L194) → 确认 `storePersonalAccessToken` 调用 `createToken()`
2. 打开 [NotifiesUserAboutNewAccessToken.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutNewAccessToken.php#L34) → 确认 `handle()` 参数为 `AccessTokenCreated`
3. 打开 [NewAccessToken.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/User/NewAccessToken.php#L53-L71) → 确认 `toMail()` 渲染 `emails.token-created` 模板

---

## 四、记录持久化机制（双通道）

### 4.1 通道一：审计日志文件（所有安全事件）

**配置**：[config/logging.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/config/logging.php#L88-L91)

```php
'audit' => [
    'driver'   => 'stack',
    'channels' => $auditChannels,  // 默认: ['audit_daily', 'audit_stdout']
],
```

所有审计通道均挂载 [AuditLogger](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Support/Logging/AuditLogger.php) → 注入 [AuditProcessor](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Support/Logging/AuditProcessor.php)，后者自动在每条日志前补全安全上下文：

```
已登录用户格式:
AUDIT: {原始消息} ({IP} ({邮箱} -> {HTTP方法}:{完整URL})

未登录用户格式:
AUDIT: {原始消息} ({IP} -> {HTTP方法}:{完整URL})
```

**使用方式**：代码中显式调用 `Log::channel('audit')`，例如：
- 登录页展示：[LoginController.php:191](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Http/Controllers/Auth/LoginController.php#L191)
- 登录失败：[LoginController.php:141](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Http/Controllers/Auth/LoginController.php#L141)
- 启用MFA：[MfaController.php:270](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Http/Controllers/Profile/MfaController.php#L270)

### 4.2 通道二：数据库审计表（交易规则变更）

**表结构**：[database/migrations/2022_10_01_210238_audit_log_entries.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/database/migrations/2022_10_01_210238_audit_log_entries.php#L48-L61)

| 字段 | 说明 |
|------|------|
| `auditable_id` + `auditable_type` | 被审计对象（多态关联，如 TransactionJournal）|
| `changer_id` + `changer_type` | 操作者（多态关联，如 Rule 或 User）|
| `action` | 操作类型（如 `update_description`, `set_budget`）|
| `before` | 修改前数据（JSON cast）|
| `after` | 修改后数据（JSON cast）|

**模型**：[AuditLogEntry.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Models/AuditLogEntry.php)
**仓储**：[ALERepository.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Repositories/AuditLogEntry/ALERepository.php#L48-L60)

**注意**：当前数据库审计仅用于**交易规则变更追踪**（如 `SetDescription`、`SetCategory` 等 Action），用户安全事件（登录/MFA/令牌）**未写入此表**，仅通过文件日志记录。

---

## 五、通知分发机制（可复核）

### 5.1 分发全链路

```
Listener::handle()
    │
    ▼  NotificationSender::send($user, new XxxNotification())
[NotificationSender.php]
    ├── 读取用户语言偏好（Preferences::getForUser($user, 'language')）
    └── NotificationFacade::locale($lang)->send($user, $notification)
              │
              ▼
[XxxNotification 类]
    ├── via()                    → 调用 ReturnsAvailableChannels::returnChannels()
    ├── toMail() / toSlack()     → 按渠道渲染消息内容
    │   / toPushover()
    └── toArray()                → 存入 notifications 表（可选）
              │
              ▼
[ReturnsAvailableChannels.php]
    ├── 'owner' 类型 → 从 FireflyConfig 读取系统级 Slack/Pushover 配置
    └── 'user'  类型 → 从 Preferences 读取用户级 Slack/Pushover 配置
                      mail 始终启用，Demo 站点禁用 mail
```

### 5.2 核心组件文件（可复核）

| 组件 | 文件路径 |
|------|---------|
| 统一发送入口 | [app/Notifications/NotificationSender.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/NotificationSender.php) |
| 渠道动态决策 | [app/Notifications/ReturnsAvailableChannels.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/ReturnsAvailableChannels.php) |
| 登录失败通知 | [app/Notifications/Security/UserFailedLoginAttempt.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Security/UserFailedLoginAttempt.php) |
| 新IP登录通知 | [app/Notifications/User/UserLogin.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/User/UserLogin.php) |
| 新令牌通知 | [app/Notifications/User/NewAccessToken.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/User/NewAccessToken.php) |
| MFA启用通知 | [app/Notifications/Security/EnabledMFANotification.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Security/EnabledMFANotification.php) |
| 未知用户登录（管理员通知） | [app/Notifications/Admin/UnknownUserLoginAttempt.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Admin/UnknownUserLoginAttempt.php) |

### 5.3 复核方法

以"登录失败通知"为例：
1. 打开 [LoginController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Http/Controllers/Auth/LoginController.php#L133) → 确认触发 `event(new UserFailedLoginAttempt($user))`
2. 打开 [NotifiesUserAboutFailedLogin.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutFailedLogin.php#L34-L37) → 确认调用 `NotificationSender::send()`
3. 打开 [NotificationSender.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/NotificationSender.php#L49) → 确认调用 `NotificationFacade::send()`
4. 打开 [ReturnsAvailableChannels.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/ReturnsAvailableChannels.php#L91-L132) → 确认渠道决策逻辑

---

## 六、典型场景完整走查

### 场景A：用户登录失败（已知用户）

```
步骤  组件                                    操作
─────  ─────────────────────────────────────   ───────────────────────────────────
 1    LoginController::login()                attemptLogin() 返回 false
       [LoginController.php:125-134]
       │
 2    查找用户                                 $repository->findByEmail($username)
       │
 3    ├─ 未知用户 → event(new UnknownUserTriedLogin($username))
       │     └──→ NotifiesOwnerAboutUnknownUser → 管理员通知
       │
       └─ 已知用户 → event(new UserFailedLoginAttempt($user))
                    │
 4                  ├── Log::channel('audit')->warning()   写入审计文件
                  │    [LoginController.php:141]
                  │
 5                  └── handle() 参数推断 → 自动绑定到
                         NotifiesUserAboutFailedLogin
                              │
 6                            ▼
                         NotificationSender::send($user,
                             new UserFailedLoginAttempt())
                              │
 7                            ▼
                         ReturnsAvailableChannels::returnChannels('user', $user)
                             → [mail, slack, pushover] (动态)
                              │
 8                            ▼
                         UserFailedLoginAttempt 通知渲染
                         ├── toMail() → emails.security.failed-login
                         │     (含 IP、Hostname、UserAgent、时间)
                         ├── toSlack() → Slack webhook
                         └── toPushover() → Pushover API
```

### 场景B：新IP登录成功（事件链 + 偏好存储）

```
步骤  组件                                    操作
─────  ─────────────────────────────────────   ───────────────────────────────────
 1    LoginController::login()                event(new UserSuccessfullyLoggedIn($user))
       [第121行]
       │
 2    ├── Log::channel('audit')->info()       写入审计文件
       │
 3    └── 自动发现 → StoresNewIpAddress
            ├── 读取 login_ip_history 偏好 (JSON数组)
            ├── 遍历检查：IP已存在？刷新时间
            ├── 清理 >6个月 的旧条目
            ├── 新IP？追加: ['ip'=>..., 'time'=>..., 'notified'=>false]
            └── Preferences::setForUser()     保存偏好
                 │
 4               └── 新IP && 用户开启通知？
                        event(new UserLoggedInFromNewIpAddress($user))
                              │
 5                            ▼
                       NotifiesUserAboutNewIpAddress
                       ├── 读取 login_ip_history
                       ├── 遍历 notified=false 的条目
                       │     └── NotificationSender::send($user, new UserLogin())
                       ├── 全部标记 notified=true
                       └── Preferences::setForUser()   保存偏好
```

### 场景C：生成新访问令牌（Passport 内部事件）

详见第三章完整链路。核心要点：
- **触发点**：`OAuthController::storePersonalAccessToken()` 调用 `$user->createToken()`
- **事件源**：Passport 内部触发 `Laravel\Passport\Events\AccessTokenCreated`
- **监听器绑定**：通过 `handle(AccessTokenCreated $event)` 参数类型自动发现
- **通知渠道**：mail / slack / pushover，内容包含创建者 IP、UA、时间

---

## 七、关键文件速查表（链接可复核）

### 事件发现与绑定

| 文件 | 作用 |
|------|------|
| [app/Providers/EventServiceProvider.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Providers/EventServiceProvider.php) | 继承 Laravel 基类，获得自动发现能力 |
| [app/Listeners/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners) | 此目录下所有类被自动扫描为监听器候选 |

### 用户安全事件

| 事件类 | 所在目录 |
|--------|---------|
| UserSuccessfullyLoggedIn / UserFailedLoginAttempt 等 12 种 | [app/Events/Security/User/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/User) |
| UnknownUserTriedLogin / NewUserRegistered 等 5 种 | [app/Events/Security/System/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/System) |

### 用户安全监听器

| 监听器目录 |
|-----------|
| [app/Listeners/Security/User/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/User) |
| [app/Listeners/Security/System/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/System) |

### 持久化相关

| 文件 | 作用 |
|------|------|
| [config/logging.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/config/logging.php) | 审计日志通道配置 |
| [app/Support/Logging/AuditProcessor.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Support/Logging/AuditProcessor.php) | 自动注入 IP/用户/URL 上下文 |
| [app/Models/AuditLogEntry.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Models/AuditLogEntry.php) | 数据库审计表模型 |
| [app/Repositories/AuditLogEntry/ALERepository.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Repositories/AuditLogEntry/ALERepository.php) | 审计仓储实现 |

### 通知分发相关

| 文件 | 作用 |
|------|------|
| [app/Notifications/NotificationSender.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/NotificationSender.php) | 统一发送入口（本地化+异常处理） |
| [app/Notifications/ReturnsAvailableChannels.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/ReturnsAvailableChannels.php) | 渠道动态决策器 |
| [app/Notifications/Security/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Security) | 安全通知类（MFA/登录等） |
| [app/Notifications/User/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/User) | 用户通知类（新IP/新令牌等） |
| [app/Notifications/Admin/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Admin) | 管理员通知类（未知用户/新注册等） |

### 关键触发控制器

| 文件 | 触发事件 |
|------|---------|
| [app/Http/Controllers/Auth/LoginController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Http/Controllers/Auth/LoginController.php) | UserSuccessfullyLoggedIn, UserFailedLoginAttempt, UnknownUserTriedLogin |
| [app/Http/Controllers/Profile/MfaController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Http/Controllers/Profile/MfaController.php) | UserHasEnabledMFA, UserHasDisabledMFA, UserHasGeneratedNewBackupCodes |
| [app/Http/Controllers/Profile/OAuthController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Http/Controllers/Profile/OAuthController.php) | createToken() → Passport 触发 AccessTokenCreated |
| [app/Http/Controllers/Auth/TwoFactorController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Http/Controllers/Auth/TwoFactorController.php) | UserHasUsedBackupCode |

---

## 八、设计要点总结

### 8.1 已修正的三处关键信息

| 原描述问题 | 修正后实际情况 | 复核依据 |
|-----------|--------------|---------|
| 事件绑定通过 `#[Subscribe]` 属性 | 绑定通过 `handle()` 方法参数类型声明完成，由基类 `discoverEvents()` 扫描 | 每个监听器的 `handle(EventClass $event)` 参数 |
| 未提及访问令牌通知路径 | `OAuthController::storePersonalAccessToken()` → `createToken()` → Passport `AccessTokenCreated` 事件 → `NotifiesUserAboutNewAccessToken` → `NewAccessToken` 通知 | 第三章完整链路 + 对应源码文件 |
| 链接使用 Windows 反斜杠路径 | 统一使用 `file:///d:/...` 格式正斜杠绝对路径 | 本文档所有链接 |

### 8.2 架构设计特征

1. **零配置绑定**：依赖 Laravel 约定优于配置，`$listen` 数组全注释仍能工作
2. **异步非阻塞**：几乎所有监听器实现 `ShouldQueue`，登录/MFA/令牌等关键操作不阻塞 HTTP 请求
3. **双轨记录**：文件日志（全量安全事件，含上下文）+ 数据库表（结构化交易变更）
4. **渠道可配置**：通知渠道按 owner/user 分级，从 FireflyConfig（系统级）和 Preferences（用户级）分别读取
5. **事件可链式触发**：一个监听器可在内部 `event()` 触发新事件（如 `StoresNewIpAddress` → `UserLoggedInFromNewIpAddress`）
6. **Demo 环境保护**：通知类 `via()` 方法中判断 `is_demo_site`，自动禁用邮件渠道防止垃圾邮件
