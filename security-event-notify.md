# Firefly III 安全事件：触发、记录持久化与通知分发协作机制

本文档梳理 Firefly III 中安全相关事件从触发到记录再到通知用户的完整协作流程。

---

## 一、整体架构概览

```
┌─────────────────────┐     ┌──────────────────┐     ┌─────────────────────────┐
│   触发源 (Controller│────▶│   Event 事件类    │────▶│   Listener 监听器们     │
│   Service, Action)  │     │  (数据载体)       │     │  (异步队列 ShouldQueue)  │
└─────────────────────┘     └──────────────────┘     └─────────┬───────────────┘
                                                               │
                    ┌──────────────────────────────────────────┼──────────────────┐
                    │                                          │                  │
                    ▼                                          ▼                  ▼
        ┌─────────────────────┐                   ┌────────────────────┐  ┌──────────────┐
        │  1. 审计日志文件     │                   │  2. 数据库持久化     │  │ 3. 通知分发   │
        │  (audit channel)    │                   │  (audit_log_entries)│  │ (邮件/Slack等)│
        └─────────────────────┘                   └────────────────────┘  └──────────────┘
```

核心协作链路：**触发源 → Event → Listener → (日志 + 数据库 + 通知)**

---

## 二、事件触发机制（Event Dispatching）

### 2.1 事件分类

所有安全事件类位于 [app/Events/Security/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security) 目录，分为两大类：

#### 用户级安全事件（User 子目录）

| 事件类 | 触发场景 |
|--------|----------|
| [UserSuccessfullyLoggedIn](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/User/UserSuccessfullyLoggedIn.php) | 用户登录成功 |
| [UserFailedLoginAttempt](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/User/UserFailedLoginAttempt.php) | 已知用户登录失败 |
| [UserLoggedInFromNewIpAddress](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/User/UserLoggedInFromNewIpAddress.php) | 用户从新IP登录 |
| [UserHasEnabledMFA](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/User/UserHasEnabledMFA.php) | 用户启用双因素认证 |
| [UserHasDisabledMFA](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/User/UserHasDisabledMFA.php) | 用户禁用双因素认证 |
| [UserHasUsedBackupCode](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/User/UserHasUsedBackupCode.php) | 用户使用了MFA备用码 |
| [UserHasGeneratedNewBackupCodes](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/User/UserHasGeneratedNewBackupCodes.php) | 用户生成了新的备用码 |
| [UserHasFewMFABackupCodesLeft](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/User/UserHasFewMFABackupCodesLeft.php) | 备用码数量不足 |
| [UserHasNoMFABackupCodesLeft](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/User/UserHasNoMFABackupCodesLeft.php) | 备用码用完 |
| [UserKeepsFailingMFA](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/User/UserKeepsFailingMFA.php) | MFA多次验证失败 |
| [UserChangedEmailAddress](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/User/UserChangedEmailAddress.php) | 用户修改邮箱 |
| [UserRequestedNewPassword](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/User/UserRequestedNewPassword.php) | 用户请求重置密码 |

#### 系统级安全事件（System 子目录）

| 事件类 | 触发场景 |
|--------|----------|
| [UnknownUserTriedLogin](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/System/UnknownUserTriedLogin.php) | 不存在的用户尝试登录 |
| [NewUserRegistered](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/System/NewUserRegistered.php) | 新用户注册 |
| [NewInvitationCreated](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/System/NewInvitationCreated.php) | 创建了新邀请 |
| [SystemRequestedVersionCheck](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/System/SystemRequestedVersionCheck.php) | 系统发起版本检查 |
| [SystemFoundNewVersionOnline](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security/System/SystemFoundNewVersionOnline.php) | 检测到新版本 |

### 2.2 触发方式：全局 `event()` 助手函数

Firefly III 使用 Laravel 的全局 `event()` 助手函数来触发事件。

**示例：登录成功/失败流程** — 见 [LoginController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Http/Controllers/Auth/LoginController.php#L86-L147)

```php
// 第121行：登录成功
event(new UserSuccessfullyLoggedIn($this->guard()->user()));

// 第130行：未知用户登录失败
event(new UnknownUserTriedLogin($username));

// 第133行：已知用户登录失败
event(new UserFailedLoginAttempt($user));
```

**示例：MFA启用流程** — 见 [MfaController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Http/Controllers/Profile/MfaController.php#L223-L274)

```php
// 第271行：启用MFA
event(new UserHasEnabledMFA($user));

// 第180行：禁用MFA
event(new UserHasDisabledMFA($user));

// 第125行：生成新备用码
event(new UserHasGeneratedNewBackupCodes($user));
```

### 2.3 事件链：事件触发事件

`StoresNewIpAddress` 监听器在处理登录成功事件时，如果检测到新IP，会**再触发一个新事件**：

见 [StoresNewIpAddress.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/User/StoresNewIpAddress.php#L36-L80)

```
UserSuccessfullyLoggedIn 事件
       │
       ▼
StoresNewIpAddress 监听器
  ├── 检查 login_ip_history 偏好设置
  └── 如果是新IP ──▶ 触发 UserLoggedInFromNewIpAddress 事件
                         │
                         ▼
                  NotifiesUserAboutNewIpAddress 监听器
```

### 2.4 事件-监听器绑定机制

值得注意的是，[EventServiceProvider.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Providers/EventServiceProvider.php) 中的 `$listen` 数组大部分被注释掉了。Firefly III 使用 **Laravel 的事件自动发现（Event Discovery）** 机制：通过监听器类名上的 `#[Subscribe]` 属性（或传统方式），由 Laravel 容器自动扫描并绑定事件与监听器。

监听器通过 `handle()` 方法的**类型声明**来标识监听哪个事件：

```php
class NotifiesUserAboutFailedLogin implements ShouldQueue
{
    // 参数类型声明 = 监听的事件
    public function handle(UserFailedLoginAttempt $event): void
    {
        NotificationSender::send($event->user, new NotificationFailedLoginAttempt($event->user));
    }
}
```

---

## 三、记录持久化机制（Persistence）

Firefly III 采用**双通道持久化**策略：文件审计日志 + 数据库审计表。

### 3.1 通道一：审计日志文件（Audit Log File）

#### 配置入口：[config/logging.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/config/logging.php)

定义了独立的 `audit` 日志通道（第88-91行）：

```php
'audit' => [
    'driver'   => 'stack',
    'channels' => $auditChannels,  // 默认: ['audit_daily', 'audit_stdout']
],
```

具体输出通道配置（均挂载了 `AuditLogger` 自定义处理器）：

| 通道 | 输出位置 | 保留天数 |
|------|----------|----------|
| `audit_daily` | `storage/logs/ff3-audit.log` | 90天 |
| `audit_stdout` | `php://stdout` (标准输出) | - |
| `audit_syslog` | 系统 syslog | - |
| `audit_papertrail` | Papertrail 远程日志 | - |
| `audit_errorlog` | PHP error_log | - |

#### 自定义处理器：[AuditLogger.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Support/Logging/AuditLogger.php) + [AuditProcessor.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Support/Logging/AuditProcessor.php)

`AuditLogger` 将 `AuditProcessor` 注入到每个日志 Handler 中。`AuditProcessor` 在日志写入前**自动附加安全上下文**：

```php
// 已登录用户
AUDIT: {原始消息} ({IP} ({邮箱} -> {HTTP方法}:{URL})

// 未登录用户
AUDIT: {原始消息} ({IP} -> {HTTP方法}:{URL})
```

#### 触发记录：代码中直接使用 `Log::channel('audit')`

如 [LoginController.php:89](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Http/Controllers/Auth/LoginController.php#L89)：

```php
Log::channel('audit')->info(sprintf('User is trying to login using "%s"', $username));
Log::channel('audit')->warning(sprintf('Login for user "%s" was locked out.', $username));
Log::channel('audit')->info(sprintf('User "%s" has been logged in.', $username));
```

### 3.2 通道二：数据库审计表（audit_log_entries）

#### 表结构：[2022_10_01_210238_audit_log_entries.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/database/migrations/2022_10_01_210238_audit_log_entries.php)

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint | 主键 |
| `auditable_id` + `auditable_type` | int + varchar | **被审计对象**（多态关联，如 TransactionJournal）|
| `changer_id` + `changer_type` | int + varchar | **操作者**（多态关联，如 User 或 Rule）|
| `action` | varchar(255) | 操作类型（如 `update_description`, `set_budget`）|
| `before` | text (JSON cast) | 修改前数据 |
| `after` | text (JSON cast) | 修改后数据 |
| `created_at` / `updated_at` | timestamp | 时间戳 |
| `deleted_at` | timestamp | 软删除 |

#### 模型与仓储

- 模型：[AuditLogEntry.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Models/AuditLogEntry.php) — 使用 `morphTo()` 多态关联
- 仓储接口：[ALERepositoryInterface.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Repositories/AuditLogEntry/ALERepositoryInterface.php)
- 仓储实现：[ALERepository.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Repositories/AuditLogEntry/ALERepository.php)

#### 触发流程：交易规则自动审计

数据库审计主要用于**交易修改追踪**（非用户安全事件），典型链路：

```
1. 交易规则执行 (app/TransactionRules/Actions/*.php)
       │
       ▼  例如: SetDescription.php:73
2. event(new TransactionGroupRequestsAuditLogEntry(
       $this->action->rule,    // changer: 触发规则
       $object,                // auditable: 交易对象
       'update_description',   // action
       $before,                // before
       $after                  // after
   ));
       │
       ▼
3. StoresAuditLogEntry 监听器（异步队列）
   [StoresAuditLogEntry.php:35-65]
   ├── before == after ? 跳过（无变化不记录）
   ├── Carbon 对象转换为 ISO8601 字符串
   └── ALERepository::store() → 写入 audit_log_entries 表
```

目前数据库审计机制**仅用于交易规则操作追踪**，用户安全事件（登录、MFA等）**尚未接入数据库审计表**，仅通过文件日志记录。

---

## 四、通知分发机制（Notification Delivery）

### 4.1 通知分发完整链路

```
Listener 监听器
     │
     ▼  NotificationSender::send($user, new XxxNotification())
NotificationSender 统一入口
     │
     ├── 1. 获取用户语言偏好（本地化）
     │
     └── 2. NotificationFacade::locale($lang)->send($user, $notification)
              │
              ▼
        XxxNotification 通知类
         ├── via()          → 通过 ReturnsAvailableChannels 获取可用渠道
         ├── toMail()       → 邮件内容（Markdown模板）
         ├── toSlack()      → Slack 消息
         ├── toPushover()   → Pushover 推送
         └── toArray()      → 数据库存储（可选）
```

### 4.2 统一发送入口：[NotificationSender.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/NotificationSender.php)

职责：
1. **语言本地化**：从用户偏好读取 `language` 设置通知语言
2. **统一错误处理**：捕获邮件发送异常（Bcc 格式错误、RFC 2822 无效等）
3. **支持两种接收者类型**：`User`（普通用户）和 `OwnerNotifiable`（系统所有者虚拟对象）

```php
public static function send(OwnerNotifiable|User $user, Notification $notification): void
{
    // 获取语言
    $lang = Preferences::getForUser($user, 'language', $lang)->data;
    
    try {
        NotificationFacade::locale($lang)->send($user, $notification);
    } catch (ClientException $e) {
        // 处理发送错误
    }
}
```

### 4.3 渠道动态决策：[ReturnsAvailableChannels.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/ReturnsAvailableChannels.php)

根据配置和用户偏好**动态决定**使用哪些通知渠道。核心方法：

```php
ReturnsAvailableChannels::returnChannels(string $type, ?User $user = null): array
// $type = 'owner' → 系统所有者渠道
// $type = 'user'  → 普通用户渠道
```

#### 系统所有者渠道 (`returnOwnerChannels`)

| 渠道 | 启用条件 | 配置来源 |
|------|----------|----------|
| `mail` | 始终启用 | 系统邮件配置 |
| `slack` | `notifications.channels.slack.enabled = true` + 配置了有效 webhook URL | `FireflyConfig` 加密存储 |
| `pushover` | `notifications.channels.pushover.enabled = true` + 配置了 AppToken + UserToken | `FireflyConfig` 加密存储 |

#### 普通用户渠道 (`returnUserChannels`)

| 渠道 | 启用条件 | 配置来源 |
|------|----------|----------|
| `mail` | 始终启用 | 用户邮箱 |
| `slack` | 全局 Slack 开启 + 用户配置了有效 webhook | `Preferences` 加密存储 |
| `pushover` | 全局 Pushover 开启 + 用户配置了双 Token | `Preferences` 加密存储 |

> **注意**：Ntfy 渠道代码存在但被注释禁用。

### 4.4 安全通知类型总览

#### 用户安全通知 ([app/Notifications/Security/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Security))

| 通知类 | 触发监听器 | 通知场景 |
|--------|-----------|----------|
| [UserFailedLoginAttempt](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Security/UserFailedLoginAttempt.php) | `NotifiesUserAboutFailedLogin` | 登录失败（含IP、UA、时间）|
| [EnabledMFANotification](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Security/EnabledMFANotification.php) | `NotifiesUserAboutEnabledMFA` | MFA 已启用 |
| [DisabledMFANotification](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Security/DisabledMFANotification.php) | `NotifiesUserAboutDisabledMFA` | MFA 已禁用 |
| [MFAUsedBackupCodeNotification](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Security/MFAUsedBackupCodeNotification.php) | `NotifiesUserAboutUsedBackupCode` | 备用码已被使用 |
| [NewBackupCodesNotification](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Security/NewBackupCodesNotification.php) | `NotifiesUserAboutNewBackupCodes` | 新备用码已生成 |
| [MFABackupFewLeftNotification](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Security/MFABackupFewLeftNotification.php) | `NotifiesUserAboutFewCodesLeft` | 备用码不足警告 |
| [MFABackupNoLeftNotification](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Security/MFABackupNoLeftNotification.php) | `NotifiesUserAboutNoCodesLeft` | 备用码用完警告 |
| [MFAManyFailedAttemptsNotification](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Security/MFAManyFailedAttemptsNotification.php) | `NotifiesUserAboutRepeatedMFAFailures` | MFA 多次失败 |

#### 管理员通知 ([app/Notifications/Admin/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Admin))

| 通知类 | 触发监听器 | 通知场景 |
|--------|-----------|----------|
| [UnknownUserLoginAttempt](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Admin/UnknownUserLoginAttempt.php) | `NotifiesOwnerAboutUnknownUser` | 未知用户尝试登录 |
| [UserRegistration](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Admin/UserRegistration.php) | `HandlesNewUserRegistration` | 新用户注册通知 |
| [VersionCheckResult](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Admin/VersionCheckResult.php) | `NotifiesOwnerAboutNewVersion` | 新版本检测结果 |
| [UserInvitation](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Admin/UserInvitation.php) | `NotifiesAboutNewInvitation` | 创建了新邀请 |

#### 用户级其他通知 ([app/Notifications/User/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/User))

| 通知类 | 触发监听器 | 通知场景 |
|--------|-----------|----------|
| [UserLogin](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/User/UserLogin.php) | `NotifiesUserAboutNewIpAddress` | 新IP地址登录 |
| [UserNewPassword](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/User/UserNewPassword.php) | `SendsUserNewPassword` | 密码重置成功 |

### 4.5 通知多渠道实现示例

以 [UserFailedLoginAttempt](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Security/UserFailedLoginAttempt.php#L38-L124) 为例，实现了三种消息格式：

```php
class UserFailedLoginAttempt extends Notification
{
    // 1. 决定渠道
    public function via(User $notifiable): array
    {
        $channels = ReturnsAvailableChannels::returnChannels('user', $notifiable);
        if ($isDemoSite) {
            return array_diff($channels, ['mail']);  // Demo站点不发邮件
        }
        return $channels;
    }

    // 2. 邮件渠道
    public function toMail(User $notifiable): MailMessage
    {
        return new MailMessage()
            ->markdown('emails.security.failed-login', [...])
            ->subject($subject);
    }

    // 3. Slack 渠道
    public function toSlack(User $notifiable): SlackMessage
    {
        return new SlackMessage()->content($message);
    }

    // 4. Pushover 渠道
    public function toPushover(User $notifiable): PushoverMessage
    {
        return PushoverMessage::create($body)->title($title);
    }
}
```

---

## 五、典型场景完整走查

### 场景A：用户登录失败（已知用户）

```
 步骤   组件                                    操作/文件
────────────────────────────────────────────────────────────────────────
 1      LoginController::login()               attemptLogin() 返回 false
        [LoginController.php:125-134]
        │
 2      查找用户                                $repository->findByEmail($username)
        │
 3      ┌─ 未知用户 ──▶ event(new UnknownUserTriedLogin($username))
        │
        └─ 已知用户 ──▶ event(new UserFailedLoginAttempt($user))
                              │
 4                            │   审计日志文件
                              ├──▶ Log::channel('audit')->warning('Login failed...')
                              │    [LoginController.php:141]
                              │
 5                            │   监听器（异步队列）
                              └──▶ NotifiesUserAboutFailedLogin::handle()
                                        │
 6                                      ▼   统一入口
                                        NotificationSender::send($user, new UserFailedLoginAttempt())
                                        │
 7                                      ▼   渠道决策
                                        ReturnsAvailableChannels::returnChannels('user', $user)
                                        │   → [mail, slack, pushover]
                                        │
 8                                      ▼   消息渲染 + 发送
                                        ├── toMail() → markdown emails.security.failed-login
                                        ├── toSlack() → Slack webhook
                                        └── toPushover() → Pushover API
```

### 场景B：用户从新IP地址登录成功（事件链 + 偏好存储）

```
 步骤   组件                                    操作
────────────────────────────────────────────────────────────────────────
 1      LoginController::login()               event(new UserSuccessfullyLoggedIn($user))
        [第121行]
        │
 2      ├──▶ Log::channel('audit')->info(...)  写入审计文件
        │
 3      └──▶ StoresNewIpAddress::handle()      监听器1：IP历史管理
                ├── 读取 login_ip_history 偏好
                ├── 遍历检查：IP已存在？刷新时间
                ├── 清理超过6个月的旧条目
                ├── 新IP？追加到数组 + 'notified' => false
                ├── 保存偏好：Preferences::setForUser(...)
                │
 4              └── 新IP && 用户开启通知？
                        event(new UserLoggedInFromNewIpAddress($user))
                              │
 5                              ▼
                        NotifiesUserAboutNewIpAddress::handle()
                        ├── 再次读取 login_ip_history
                        ├── 遍历 notified=false 的条目
                        │     └── NotificationSender::send($user, new UserLogin())
                        ├── 全部标记 notified=true
                        └── 保存偏好
```

### 场景C：交易规则修改描述 → 数据库审计

```
 步骤   组件                                    操作
────────────────────────────────────────────────────────────────────────
 1      TransactionRule 引擎 执行 SetDescription Action
        [SetDescription.php]
        │
 2      event(new TransactionGroupRequestsAuditLogEntry(
                  changer: Rule对象,      （谁发起的修改）
                  auditable: Journal对象, （被修改的东西）
                  field: 'update_description',
                  before: $before,
                  after: $text
              ))
        │
 3      ▼  监听器（异步队列）
        StoresAuditLogEntry::handle()
        ├── before === after ? 跳过（优化）
        ├── Carbon → ISO8601 格式转换
        └── ALERepository::store($data)
                │
 4              ▼  数据库写入
                AuditLogEntry 模型
                → auditable_id/auditable_type  (Journal)
                → changer_id/changer_type      (Rule)
                → action='update_description'
                → before / after (JSON cast)
                → INSERT INTO audit_log_entries
```

---

## 六、设计要点总结

### 6.1 架构优势

1. **解耦分层**：事件（数据）→ 监听器（逻辑）→ 通知（分发）三层分离，单一职责
2. **异步处理**：几乎所有安全监听器都实现 `ShouldQueue` 接口，避免阻塞用户请求
3. **双轨记录**：文件日志（人可读+完整上下文）+ 数据库表（结构化查询）互补
4. **动态渠道**：`ReturnsAvailableChannels` 按配置/偏好动态决定通知渠道，扩展性好
5. **多语言支持**：通知发送前自动读取用户语言偏好

### 6.2 值得注意的细节

1. **用户安全事件未接入数据库审计表**：登录/MFA等安全事件仅写入 `audit` 日志文件，未进入 `audit_log_entries` 表。目前该表只服务于交易规则变更追踪。
2. **事件自动发现**：`EventServiceProvider::$listen` 基本为空，依赖 Laravel 事件自动发现机制，监听器与事件通过 `handle()` 方法参数类型隐式绑定
3. **Demo 站点特殊处理**：通知类的 `via()` 方法中会判断 `is_demo_site`，Demo 环境不发送邮件
4. **IP 历史存储在 Preferences**：不是独立数据表，而是用户偏好 JSON 数组（带过期清理）
5. **双重通知对象**：系统级事件（如未知用户登录）通知发给虚拟的 `OwnerNotifiable`，而非真实 User 对象
6. **交易审计支持"无变化跳过"优化**：`StoresAuditLogEntry` 在 `before === after` 时直接跳过，减少无效记录

### 6.3 关键文件速查表

| 层级 | 文件路径 |
|------|---------|
| **事件定义** | [app/Events/Security/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Events/Security) |
| **用户安全监听器** | [app/Listeners/Security/User/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/User) |
| **系统安全监听器** | [app/Listeners/Security/System/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Security/System) |
| **交易审计监听器** | [app/Listeners/Model/TransactionGroup/StoresAuditLogEntry.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Listeners/Model/TransactionGroup/StoresAuditLogEntry.php) |
| **通知发送入口** | [app/Notifications/NotificationSender.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/NotificationSender.php) |
| **渠道决策器** | [app/Notifications/ReturnsAvailableChannels.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/ReturnsAvailableChannels.php) |
| **安全通知类** | [app/Notifications/Security/](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Notifications/Security) |
| **审计日志配置** | [config/logging.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/config/logging.php) |
| **审计处理器** | [app/Support/Logging/AuditProcessor.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Support/Logging/AuditProcessor.php) |
| **审计日志模型** | [app/Models/AuditLogEntry.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Models/AuditLogEntry.php) |
| **审计日志仓储** | [app/Repositories/AuditLogEntry/ALERepository.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Repositories/AuditLogEntry/ALERepository.php) |
| **事件服务提供者** | [app/Providers/EventServiceProvider.php](file:///d:/fz/0508-2/solo-dogfeeding/code/129-firefly-iii/app/Providers/EventServiceProvider.php) |
