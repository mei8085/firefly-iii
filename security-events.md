# 安全事件到登录异常提醒的走向分析

## 概述

本文档详细分析 Firefly III 系统中安全事件从触发到用户通知的完整流程，包括事件触发机制、监听器处理、用户通知发送以及会话异常处理的关系。

---

## 1. 整体架构流程

```
登录请求 → 事件触发 → 事件广播 → 监听器处理 → 通知发送 → 用户接收
                                  ↓
                           会话异常处理
```

---

## 2. 安全事件类型与触发点

### 2.1 登录流程中的事件触发

**核心触发点**：[LoginController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Auth/LoginController.php#L86-L147)

#### 2.1.1 登录成功流程

```php
// 第121行：登录成功时触发 UserSuccessfullyLoggedIn 事件
event(new UserSuccessfullyLoggedIn($this->guard()->user()));
```

**事件**：[UserSuccessfullyLoggedIn](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Events/Security/User/UserSuccessfullyLoggedIn.php)
- 携带数据：用户对象（User）

#### 2.1.2 登录失败流程（已知用户）

```php
// 第132-134行：已知用户登录失败时触发
if ($user instanceof User) {
    event(new UserFailedLoginAttempt($user));
}
```

**事件**：[UserFailedLoginAttempt](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Events/Security/User/UserFailedLoginAttempt.php)
- 携带数据：用户对象（User）

#### 2.1.3 登录失败流程（未知用户）

```php
// 第128-131行：未知用户尝试登录时触发
if (!$user instanceof User) {
    event(new UnknownUserTriedLogin($username));
}
```

**事件**：[UnknownUserTriedLogin](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Events/Security/System/UnknownUserTriedLogin.php)
- 携带数据：尝试登录的邮箱地址（string）

#### 2.1.4 登录锁定处理

```php
// 第106-111行：登录尝试过多时触发锁定
if ($this->hasTooManyLoginAttempts($request)) {
    $this->fireLockoutEvent($request);
    $this->sendLockoutResponse($request);
}
```

### 2.2 双因素认证（MFA）事件触发

**核心触发点**：[TwoFactorController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Auth/TwoFactorController.php#L65-L126)

#### 2.2.1 MFA 多次失败

```php
// 第87-91行：MFA失败3次或10次时触发警告
$counter = $this->getMFAFailureCounter();
if (3 === $counter || 10 === $counter) {
    event(new UserKeepsFailingMFA($user, $counter));
}
```

**事件**：`UserKeepsFailingMFA`
- 携带数据：用户对象、失败次数

#### 2.2.2 备份码使用

```php
// 第107-120行：使用备份码登录时触发
if ($this->isBackupCode($mfaCode)) {
    $this->removeFromBackupCodes($mfaCode);   // ← 内部会根据剩余数量再触发事件
    $authenticator->login();
    $this->resetMFAFailureCounter();
    event(new UserHasUsedBackupCode($user));   // ← 始终触发：通知用户备份码已被使用
}
```

**事件**：[UserHasUsedBackupCode](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Events/Security/User/UserHasUsedBackupCode.php)
- 携带数据：用户对象（User）

### 2.3 MFA 管理事件触发

**核心触发点**：[MfaController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Profile/MfaController.php)

#### 2.3.1 启用 MFA

```php
// MfaController.php 第271行
event(new UserHasEnabledMFA($user));
```

#### 2.3.2 禁用 MFA

```php
// MfaController.php 第180行
event(new UserHasDisabledMFA($user));
```

#### 2.3.3 生成新备份码

```php
// MfaController.php 第125行
event(new UserHasGeneratedNewBackupCodes($user));
```

---

## 3. 事件与监听器的映射关系

Laravel 采用**事件自动发现**机制（无需在 EventServiceProvider 中显式注册），通过类型提示自动匹配事件和监听器。

### 3.1 登录失败事件监听链

| 事件 | 监听器 | 处理逻辑 |
|------|--------|----------|
| `UserFailedLoginAttempt` | [NotifiesUserAboutFailedLogin](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutFailedLogin.php) | 发送登录失败通知给用户 |
| `UnknownUserTriedLogin` | [NotifiesOwnerAboutUnknownUser](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/System/NotifiesOwnerAboutUnknownUser.php) | 发送未知用户登录通知给系统所有者 |

### 3.2 登录成功事件监听链

| 事件 | 监听器 | 处理逻辑 |
|------|--------|----------|
| `UserSuccessfullyLoggedIn` | [StoresNewIpAddress](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/StoresNewIpAddress.php#L36-L80) | 存储登录IP，检查是否为新IP，若是则触发 `UserLoggedInFromNewIpAddress` |
| `UserLoggedInFromNewIpAddress` | [NotifiesUserAboutNewIpAddress](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutNewIpAddress.php#L35-L58) | 发送新IP登录通知给用户 |
| `Illuminate\Auth\Events\Login` | [RespondsToNewLogin](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/RespondsToNewLogin.php) | 检查单用户管理员权限、重置演示用户语言 |

### 3.3 MFA 安全事件完整监听链

| 事件 | 监听器 | 通知类 | 处理逻辑 |
|------|--------|--------|----------|
| `UserKeepsFailingMFA` | [NotifiesUserAboutRepeatedMFAFailures](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutRepeatedMFAFailures.php) | `MFAManyFailedAttemptsNotification` | MFA多次失败警告 |
| `UserHasUsedBackupCode` | [NotifiesUserAboutUsedBackupCode](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutUsedBackupCode.php) | `MFAUsedBackupCodeNotification` | 备份码已被使用提醒 |
| `UserHasFewMFABackupCodesLeft` | [NotifiesUserAboutFewCodesLeft](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutFewCodesLeft.php) | `MFABackupFewLeftNotification` | 备份码剩余不足警告 |
| `UserHasNoMFABackupCodesLeft` | [NotifiesUserAboutNoCodesLeft](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutNoCodesLeft.php) | `MFABackupNoLeftNotification` | 备份码已全部用完警告 |
| `UserHasEnabledMFA` | [NotifiesUserAboutEnabledMFA](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutEnabledMFA.php) | `EnabledMFANotification` | MFA已启用通知 |
| `UserHasDisabledMFA` | [NotifiesUserAboutDisabledMFA](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutDisabledMFA.php) | `DisabledMFANotification` | MFA已禁用通知 |
| `UserHasGeneratedNewBackupCodes` | [NotifiesUserAboutNewBackupCodes](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutNewBackupCodes.php) | `NewBackupCodesNotification` | 新备份码已生成通知 |

---

## 4. 用户通知系统

### 4.1 通知发送核心类

**统一入口**：[NotificationSender.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/NotificationSender.php)

```php
public static function send(OwnerNotifiable|User $user, Notification $notification): void
{
    $lang = config('firefly.default_language');
    if ($user instanceof User) {
        $lang = Preferences::getForUser($user, 'language', $lang)->data;
    }
    try {
        NotificationFacade::locale($lang)->send($user, $notification);
    } catch (Exception $e) {
        // 错误处理：Bcc错误、RFC 2822错误等
    }
}
```

### 4.2 通知类型与内容

#### 4.2.1 登录失败通知

**类**：[UserFailedLoginAttempt](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/UserFailedLoginAttempt.php)

- **邮件主题**：`email.failed_login_subject`
- **包含信息**：
  - IP 地址（`Request::ip()`）
  - 主机名（`Steam::getHostName($ip)`）
  - User Agent
  - 登录时间
- **投递渠道**：mail, slack, pushover（可配置）
- **Demo站点**：排除 mail 渠道

#### 4.2.2 新IP登录通知

**类**：[UserLogin](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/User/UserLogin.php)

- **邮件主题**：`email.login_from_new_ip`
- **包含信息**：IP、主机名、User Agent、时间
- **投递渠道**：mail, slack, pushover

#### 4.2.3 未知用户登录通知

**类**：[UnknownUserLoginAttempt](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Admin/UnknownUserLoginAttempt.php)

- **接收者**：系统所有者（OwnerNotifiable）
- **邮件主题**：`email.unknown_user_subject`
- **包含信息**：尝试的邮箱地址、IP、主机名、User Agent、时间

---

## 5. 备份码使用后的完整通知链

这是之前文档中缺失的关键链路。备份码使用后的通知**不是单一事件**，而是一个**级联触发**的过程。

### 5.1 级联触发源码分析

**入口**：[TwoFactorController.submitMFA()](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Auth/TwoFactorController.php#L107-L121)

```php
// 第107行：检测到用户提交的是备份码
if ($this->isBackupCode($mfaCode)) {
    $this->removeFromBackupCodes($mfaCode);   // ← 关键：此方法内部会级联触发更多事件
    $authenticator->login();                   // ← 认证通过
    $this->resetMFAFailureCounter();           // ← 重置MFA失败计数
    event(new UserHasUsedBackupCode($user));   // ← 始终触发：通知用户备份码已被使用
}
```

### 5.2 removeFromBackupCodes 内部的级联逻辑

**源码**：[TwoFactorController.removeFromBackupCodes()](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Auth/TwoFactorController.php#L208-L230)

```php
private function removeFromBackupCodes(string $mfaCode): void
{
    $list    = Preferences::get('mfa_recovery', [])->data;
    $newList = array_values(array_diff($list, [$mfaCode]));

    // 条件1：剩余 ≤3 且 >0 → 触发"备份码不足"事件
    if (count($newList) <= 3 && count($newList) > 0) {
        event(new UserHasFewMFABackupCodesLeft($user, count($newList)));
    }

    // 条件2：剩余 =0 → 触发"备份码用完"事件
    if (0 === count($newList)) {
        event(new UserHasNoMFABackupCodesLeft($user));
    }

    Preferences::set('mfa_recovery', $newList);
}
```

### 5.3 备份码使用的三级通知时序

```
用户提交MFA码 → 检测为备份码
    ↓
[1] removeFromBackupCodes(mfaCode)
    ├─ 从 mfa_recovery 中移除该码
    ├─ 判断剩余数量
    │   ├─ 剩余 ≤3 且 >0 → event(UserHasFewMFABackupCodesLeft)
    │   │       ↓
    │   │   [NotifiesUserAboutFewCodesLeft]
    │   │       → NotificationSender::send(MFABackupFewLeftNotification)
    │   │       → 通知内容：剩余N个备份码，建议生成新码
    │   │
    │   └─ 剩余 =0 → event(UserHasNoMFABackupCodesLeft)
    │           ↓
    │       [NotifiesUserAboutNoCodesLeft]
    │           → NotificationSender::send(MFABackupNoLeftNotification)
    │           → 通知内容：所有备份码已用完，必须生成新码
    │
    └─ 保存新的备份码列表到偏好
    ↓
[2] authenticator->login()   ← 认证通过，用户可进入系统
    ↓
[3] event(UserHasUsedBackupCode)   ← 始终触发
        ↓
    [NotifiesUserAboutUsedBackupCode]
        → NotificationSender::send(MFAUsedBackupCodeNotification)
        → 通知内容：您的备份码已被使用，包含IP/主机名/UA/时间
```

### 5.4 备份码通知的触发条件汇总

| 条件 | 触发的事件 | 通知类 | 紧急程度 |
|------|-----------|--------|----------|
| 每次使用备份码 | `UserHasUsedBackupCode` | [MFAUsedBackupCodeNotification](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFAUsedBackupCodeNotification.php) | ⚠️ 提醒级 |
| 使用后剩余 ≤3 且 >0 | `UserHasFewMFABackupCodesLeft` | [MFABackupFewLeftNotification](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFABackupFewLeftNotification.php) | ⚠️ 警告级 |
| 使用后剩余 =0 | `UserHasNoMFABackupCodesLeft` | [MFABackupNoLeftNotification](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFABackupNoLeftNotification.php) | 🔴 严重级 |

### 5.5 备份码通知的共同信息

所有备份码相关通知都包含以下安全上下文信息：
- **IP 地址**：`Request::ip()`
- **主机名**：`Steam::getHostName($ip)`
- **User Agent**：`Request::userAgent()`
- **时间戳**：当前时间（按用户时区格式化）

---

## 6. 改密后的会话状态

### 6.1 关键发现：改密后不会主动失效其他会话

**源码**：[ProfileController.postChangePassword()](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/ProfileController.php#L283-L310)

```php
public function postChangePassword(ProfileFormRequest $request, UserRepositoryInterface $repository): RedirectResponse
{
    // ... 省略验证逻辑 ...

    $repository->changePassword($user, $request->get('new_password'));
    session()->flash('success', (string) trans('firefly.password_changed'));

    return redirect(route('profile.index'));   // ← 仅重定向回个人主页，没有其他操作
}
```

**UserRepository.changePassword()** 的实现：

```php
// UserRepository.php 第102-108行
public function changePassword(User $user, #[SensitiveParameter] string $password): bool
{
    $user->password = bcrypt($password);
    $user->save();
    return true;
}
```

**结论**：改密方法**仅更新数据库中的密码哈希**，不执行任何会话操作。具体影响：

| 行为 | 改密后是否发生 | 说明 |
|------|:---:|------|
| 当前会话失效 | ❌ | 当前用户不会登出 |
| 其他设备会话失效 | ❌ | 没有调用 `logoutOtherDevices()` |
| 发送安全通知 | ❌ | 没有触发任何安全事件 |
| MFA 状态变化 | ❌ | MFA 密钥不受影响 |
| MFA Cookie 清除 | ❌ | 没有清除 MFA token cookie |

### 6.2 与"注销其他会话"的对比

`logoutOtherDevices()` 只在用户**主动操作**时调用：

**源码**：[ProfileController.postLogoutOtherSessions()](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/ProfileController.php#L343-L360)

```php
public function postLogoutOtherSessions(Request $request): RedirectResponse
{
    $creds = ['email' => auth()->user()->email, 'password' => $request->get('password')];
    if (Auth::once($creds)) {
        Auth::logoutOtherDevices($request->get('password'));
        session()->flash('info', (string) trans('firefly.other_sessions_logged_out'));
        return redirect(route('profile.index'));
    }
    session()->flash('error', (string) trans('auth.failed'));
    return redirect(route('profile.index'));
}
```

### 6.3 Laravel logoutOtherDevices 的底层原理

`Auth::logoutOtherDevices($password)` 是 Laravel `Illuminate\Auth\SessionGuard` 提供的方法，其工作原理：

1. 使用传入的密码重新哈希用户的 `password` 字段（调用 `$user->setPasswordAttribute($password)`）
2. 在 `password` 列值改变后，其他会话的 `password_hash` 指纹不再匹配
3. 当其他设备的会话尝试恢复时，Laravel 中间件检测到 `password_hash` 不匹配，自动使会话失效
4. **当前会话不受影响**，因为调用后 Laravel 会同步更新当前会话的 `password_hash` 指纹

**注意**：此机制依赖数据库中 `users` 表的 `password` 字段变化作为会话验证依据，且需要 session driver 支持用户级会话管理（如 database 或 redis 驱动）。

### 6.4 改密与会话安全的真实关系

```
改密操作（postChangePassword）
    ├─ $user->password = bcrypt($new)   ← 更新密码哈希
    ├─ $user->save()                    ← 保存到数据库
    ├─ ❌ 不调用 logoutOtherDevices()
    ├─ ❌ 不调用 session()->invalidate()
    ├─ ❌ 不触发安全事件
    └─ ❌ 不发送通知
    
    后果：其他已登录设备的会话仍然有效
    原因：Laravel 的 "remember me" token 和 session 不依赖密码哈希验证
```

与邮箱变更操作的对比（[postChangeEmail()](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/ProfileController.php#L239-L278)）：

```php
// 邮箱变更后：主动强制登出
Auth::guard()->logout();
$request->session()->invalidate();
session()->flash('success', (string) trans('firefly.email_changed'));
return redirect(route('index'));
```

**邮箱变更会强制登出当前用户，但改密不会。**

---

## 7. 主动注销其他会话与自动提醒的区别

### 7.1 本质区别

| 维度 | 主动注销其他会话 | 自动安全提醒 |
|------|------------------|--------------|
| **触发方式** | 用户手动操作 | 系统自动检测 |
| **目的** | 清除其他设备上的活跃会话 | 通知用户存在异常行为 |
| **是否改变会话状态** | ✅ 立即使其他会话失效 | ❌ 不改变任何会话状态 |
| **是否发送通知** | ❌ 不发送通知 | ✅ 发送通知（邮件/Slack/Pushover） |
| **用户感知** | 操作者看到 flash 提示 | 被通知者收到推送/邮件 |
| **代码路径** | `Auth::logoutOtherDevices()` | `event() → Listener → NotificationSender` |

### 7.2 主动注销其他会话的完整流程

```
用户访问"注销其他会话"页面
    ↓
[GET] profile.logout-other-sessions → 展示确认页面
    ↓
用户输入密码并提交
    ↓
[POST] profile.logout-other-sessions
    ↓
[ProfileController.postLogoutOtherSessions()]
    ├─ Auth::once($creds)     ← 单次验证密码，不创建持久会话
    ├─ 验证成功 → Auth::logoutOtherDevices($password)
    │   ├─ 重新哈希用户密码（password 字段值微调）
    │   ├─ 当前会话的 password_hash 指纹同步更新
    │   └─ 其他会话的 password_hash 指纹不再匹配 → 失效
    ├─ session()->flash('info', '其他会话已注销')
    └─ 验证失败 → session()->flash('error', 'auth.failed')
```

**关键细节**：
- `Auth::once()` 只做单次密码验证，不会影响当前认证状态
- `logoutOtherDevices()` 必须传入明文密码，因为它会重新对密码做 bcrypt 哈希
- 此操作**不触发任何安全事件**，不发通知，仅通过 flash message 告知操作者

### 7.3 自动安全提醒的完整流程

以"新IP登录"为例：

```
系统检测到异常（新IP登录）
    ↓
event(UserLoggedInFromNewIpAddress)
    ↓
[NotifiesUserAboutNewIpAddress.handle()]
    ├─ 读取 login_ip_history
    ├─ 查找 notified=false 的记录
    ├─ 对每条未通知记录 → NotificationSender::send(new UserLogin())
    │   ├─ toMail() → Markdown 邮件（包含 IP/主机名/UA/时间）
    │   ├─ toSlack() → Slack 消息
    │   └─ toPushover() → Pushover 推送
    └─ 标记 notified=true，保存回偏好
```

**关键细节**：
- 自动提醒**不改变会话状态**，收到通知的用户仍需手动处理
- 自动提醒**不注销任何会话**，仅提供信息让用户自行判断
- 通知是否发送受用户偏好控制（`notification_user_login`）

### 7.4 两种机制在不同安全场景下的配合

| 安全场景 | 自动提醒 | 主动注销 |
|----------|----------|----------|
| 收到"新IP登录"通知，非本人操作 | 系统自动发送 | 用户手动注销其他会话 |
| 收到"登录失败"通知 | 系统自动发送 | 无直接关联 |
| 收到"备份码被使用"通知 | 系统自动发送 | 用户可能选择注销+改密 |
| 收到"MFA多次失败"通知 | 系统自动发送 | 用户应立即改密+注销 |
| 改密后需清除其他会话 | 无自动提醒 | 用户必须**手动**注销其他会话 |

### 7.5 设计隐患

**改密不自动注销其他会话**是一个值得注意的安全缺口：

```
用户改密（postChangePassword）
    → 数据库密码已更新
    → 但攻击者的旧会话仍然有效！
    → 用户需要额外手动执行"注销其他会话"
```

对比邮箱变更操作的严谨处理：

```
用户变更邮箱（postChangeEmail）
    → 数据库邮箱已更新
    → Auth::guard()->logout()           ← 强制登出
    → $request->session()->invalidate() ← 使会话失效
    → 用户必须重新登录
```

---

## 8. IP 地址跟踪与会话异常检测

### 8.1 IP 存储与检测

**核心逻辑**：[StoresNewIpAddress.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/StoresNewIpAddress.php#L36-L80)

```php
// 存储用户登录IP历史
$preference = Preferences::getForUser($user, 'login_ip_history', [])->data;

// 检查IP是否已存在
foreach ($preference as $index => $row) {
    if ($row['ip'] === $ip) {
        $inArray = true;
        $preference[$index]['time'] = now();
    }
    // 清理6个月前的旧记录
    if ($carbon->diffInMonths(today(), true) > 6) {
        unset($preference[$index]);
    }
}

// 新IP：添加记录并标记未通知
if (false === $inArray) {
    $preference[] = [
        'ip' => $ip, 
        'time' => now(), 
        'notified' => false
    ];
}

// 用户开启登录通知时，触发新IP事件
if (false === $inArray && true === $send) {
    event(new UserLoggedInFromNewIpAddress($user));
}
```

### 8.2 新IP通知发送

**核心逻辑**：[NotifiesUserAboutNewIpAddress.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutNewIpAddress.php#L35-L58)

```php
public function handle(UserLoggedInFromNewIpAddress $event): void
{
    $user = $event->user;
    if ($user->hasRole('demo')) {
        return;  // 演示用户不发送通知
    }

    $list = Preferences::getForUser($user, 'login_ip_history', [])->data;
    foreach ($list as $index => $entry) {
        if (false === $entry['notified']) {
            NotificationSender::send($user, new UserLogin());  // ← 注意：UserLogin 不传剩余码数量等信息
        }
        $list[$index]['notified'] = true;
    }
    Preferences::setForUser($user, 'login_ip_history', $list);
}
```

---

## 9. 完整事件流时序图

### 9.1 登录失败通知流程

```
用户登录失败
    ↓
[LoginController.login]
    ├─ 验证用户存在性
    ├─ 已知用户 → event(UserFailedLoginAttempt)
    └─ 未知用户 → event(UnknownUserTriedLogin)
    ↓
[事件自动分发]
    ↓
[监听器处理]
    ├─ NotifiesUserAboutFailedLogin
    │   └─ NotificationSender::send(UserFailedLoginAttempt)
    │       └─ 邮件/Slack/Pushover 通知
    └─ NotifiesOwnerAboutUnknownUser
        └─ NotificationSender::send(UnknownUserLoginAttempt)
            └─ 发送给系统所有者
```

### 9.2 新IP登录通知流程

```
用户登录成功
    ↓
event(UserSuccessfullyLoggedIn)
    ↓
[StoresNewIpAddress.handle]
    ├─ 检查IP历史
    ├─ IP已存在 → 更新时间戳
    └─ IP不存在 → 添加记录（notified=false）
        └─ 用户开启通知偏好 → event(UserLoggedInFromNewIpAddress)
            ↓
[NotifiesUserAboutNewIpAddress.handle]
    ├─ 排除演示用户
    ├─ 查找未通知的IP记录
    ├─ 发送 UserLogin 通知
    └─ 标记 notified=true
```

### 9.3 MFA 异常与备份码完整通知流程

```
用户提交MFA码
    ↓
[TwoFactorController.submitMFA]
    ├─ 码在5分钟历史中 → 拒绝（防重放）
    ├─ MFA码正确 → 记录历史 + 重置失败计数 → 进入系统
    ├─ MFA码错误 → 增加失败计数
    │   ├─ 3次或10次 → event(UserKeepsFailingMFA)
    │   │       ↓
    │   │   [NotifiesUserAboutRepeatedMFAFailures]
    │   │       → MFAManyFailedAttemptsNotification
    │   └─ 其他次数 → 仅增加计数，不触发事件
    └─ 检测为备份码 → [removeFromBackupCodes]
        ├─ 剩余 ≤3 且 >0 → event(UserHasFewMFABackupCodesLeft)
        │       ↓
        │   [NotifiesUserAboutFewCodesLeft]
        │       → MFABackupFewLeftNotification（含剩余数量）
        │
        ├─ 剩余 =0 → event(UserHasNoMFABackupCodesLeft)
        │       ↓
        │   [NotifiesUserAboutNoCodesLeft]
        │       → MFABackupNoLeftNotification
        │
        └─ 保存新列表
        ↓
    authenticator->login()  ← 认证通过
    resetMFAFailureCounter()
    ↓
    event(UserHasUsedBackupCode)   ← 始终触发
        ↓
    [NotifiesUserAboutUsedBackupCode]
        → MFAUsedBackupCodeNotification（含IP/UA/时间）
```

### 9.4 会话安全操作对比流程

```
场景A：主动注销其他会话
    用户 → [POST /profile/logout-other-sessions]
        → Auth::once() 验证密码
        → Auth::logoutOtherDevices($password)
        → 其他设备会话立即失效
        → 当前会话保持
        → flash 提示"已注销"
    ❌ 不触发事件，不发送通知

场景B：改密
    用户 → [POST /profile/change-password]
        → 验证当前密码 + 新密码
        → $user->password = bcrypt($new)
        → $user->save()
        → flash 提示"密码已更改"
    ❌ 不触发事件，不发送通知
    ❌ 不调用 logoutOtherDevices()
    ❌ 不使任何会话失效
    ⚠️  其他设备上的旧会话仍然有效

场景C：变更邮箱
    用户 → [POST /profile/change-email]
        → 更新邮箱
        → event(UserChangedEmailAddress)
        → Auth::guard()->logout()
        → $request->session()->invalidate()
        → 重定向到首页
    ✅ 强制登出当前用户
    ✅ 使当前会话失效
    ✅ 触发事件 → 发送确认邮件 + 撤销邮件
    ⚠️  但不调用 logoutOtherDevices()，其他设备的会话可能仍然有效
```

---

## 10. 关键配置与偏好设置

| 配置项 | 位置 | 作用 |
|--------|------|------|
| `login_ip_history` | 用户偏好 | 存储登录IP历史记录（自动清理6个月前记录） |
| `notification_user_login` | 用户偏好 | 是否开启新IP登录通知（默认true），控制 `UserLoggedInFromNewIpAddress` 事件是否触发 |
| `mfa_failure_count` | 用户偏好 | MFA失败计数器，成功登录或使用备份码后重置为0 |
| `mfa_history` | 用户偏好 | MFA码使用历史（5分钟窗口防重放攻击） |
| `mfa_recovery` | 用户偏好 | 备份码列表，使用后逐个移除 |

---

## 11. 代码参考索引

### 11.1 控制器
- [LoginController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Auth/LoginController.php) - 登录流程控制
- [TwoFactorController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Auth/TwoFactorController.php) - MFA验证控制 + 备份码级联逻辑
- [MfaController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Profile/MfaController.php) - MFA启用/禁用/备份码管理
- [ProfileController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/ProfileController.php) - 密码修改 + 会话管理

### 11.2 事件类
- [UserFailedLoginAttempt.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Events/Security/User/UserFailedLoginAttempt.php)
- [UserSuccessfullyLoggedIn.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Events/Security/User/UserSuccessfullyLoggedIn.php)
- [UserLoggedInFromNewIpAddress.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Events/Security/User/UserLoggedInFromNewIpAddress.php)
- [UnknownUserTriedLogin.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Events/Security/System/UnknownUserTriedLogin.php)
- [UserHasUsedBackupCode.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Events/Security/User/UserHasUsedBackupCode.php)
- [UserHasFewMFABackupCodesLeft.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Events/Security/User/UserHasFewMFABackupCodesLeft.php)
- [UserHasNoMFABackupCodesLeft.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Events/Security/User/UserHasNoMFABackupCodesLeft.php)
- [UserHasEnabledMFA.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Events/Security/User/UserHasEnabledMFA.php)
- [UserHasDisabledMFA.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Events/Security/User/UserHasDisabledMFA.php)
- [UserHasGeneratedNewBackupCodes.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Events/Security/User/UserHasGeneratedNewBackupCodes.php)

### 11.3 监听器类
- [NotifiesUserAboutFailedLogin.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutFailedLogin.php)
- [NotifiesOwnerAboutUnknownUser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/System/NotifiesOwnerAboutUnknownUser.php)
- [StoresNewIpAddress.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/StoresNewIpAddress.php)
- [NotifiesUserAboutNewIpAddress.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutNewIpAddress.php)
- [NotifiesUserAboutRepeatedMFAFailures.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutRepeatedMFAFailures.php)
- [NotifiesUserAboutUsedBackupCode.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutUsedBackupCode.php)
- [NotifiesUserAboutFewCodesLeft.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutFewCodesLeft.php)
- [NotifiesUserAboutNoCodesLeft.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutNoCodesLeft.php)
- [NotifiesUserAboutEnabledMFA.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutEnabledMFA.php)
- [NotifiesUserAboutDisabledMFA.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutDisabledMFA.php)
- [NotifiesUserAboutNewBackupCodes.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutNewBackupCodes.php)
- [HandlesChangeOfUserEmailAddress.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/HandlesChangeOfUserEmailAddress.php)

### 11.4 通知类
- [UserFailedLoginAttempt.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/UserFailedLoginAttempt.php)
- [MFAUsedBackupCodeNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFAUsedBackupCodeNotification.php)
- [MFABackupFewLeftNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFABackupFewLeftNotification.php)
- [MFABackupNoLeftNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFABackupNoLeftNotification.php)
- [MFAManyFailedAttemptsNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFAManyFailedAttemptsNotification.php)
- [UserLogin.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/User/UserLogin.php)
- [UnknownUserLoginAttempt.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Admin/UnknownUserLoginAttempt.php)
- [NotificationSender.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/NotificationSender.php)

---

## 12. 设计特点总结

1. **事件驱动架构**：使用 Laravel 事件系统实现松耦合，控制器只需触发事件，无需关心后续处理
2. **自动发现机制**：通过类型提示自动匹配事件和监听器，无需手动注册
3. **队列处理**：所有监听器都实现 `ShouldQueue` 接口，通知发送异步处理，不阻塞登录流程
4. **多渠道通知**：支持邮件、Slack、Pushover 等多种通知渠道
5. **偏好控制**：用户可自主选择是否接收登录通知（`notification_user_login`）
6. **历史记录**：IP历史记录保留6个月，MFA历史记录保留5分钟（防重放），便于审计和异常检测
7. **分级警告**：MFA失败在3次和10次时分别发送警告；备份码剩余 ≤3 和 =0 时分别通知，渐进式提醒
8. **级联触发**：备份码使用后，`removeFromBackupCodes()` 根据剩余数量级联触发不同级别的安全事件
9. **会话安全缺口**：改密操作不自动注销其他会话、不触发事件、不发送通知，与邮箱变更操作的严谨处理形成对比
10. **主动防御与被动通知的分工**：主动注销其他会话是"清除"动作，自动提醒是"告知"动作，两者互补但不联动
