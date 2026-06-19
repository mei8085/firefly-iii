# 安全事件到登录异常提醒的走向分析

## 概述

本文档详细分析 Firefly III 系统中安全事件从触发到用户通知的完整流程，包括事件触发机制、监听器处理、用户通知发送以及会话异常处理的关系。重点拆解了 `logoutOtherDevices()` 在 Firefly III 实际配置下的真实生效条件。

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

### 6.3 改密与会话安全的真实关系

```
改密操作（postChangePassword）
    ├─ $user->password = bcrypt($new)   ← 更新密码哈希
    ├─ $user->save()                    ← 保存到数据库
    ├─ ❌ 不调用 logoutOtherDevices()
    ├─ ❌ 不调用 session()->invalidate()
    ├─ ❌ 不触发安全事件
    └─ ❌ 不发送通知
    
    后果：其他已登录设备的会话仍然有效
    原因：既没有 AuthenticateSession 中间件的 password_hash 比对，
          也没有手动清除其他会话
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

## 7. 主动注销其他会话的三层生效条件（核心拆解）

`Auth::logoutOtherDevices($password)` 是 Laravel `Illuminate\Auth\SessionGuard` 提供的方法，但其**实际效果依赖三个层级的配置**。Firefly III 的默认配置在关键层级上存在缺失，导致该方法的实际效果与 Laravel 文档描述不完全一致。

### 7.1 第一层：框架注销调用（SessionGuard 内部逻辑）

**框架源码逻辑**（`Illuminate\Auth\SessionGuard::logoutOtherDevices()`，伪代码还原）：

```php
public function logoutOtherDevices($password, $attribute = 'password')
{
    if (! $user = $this->user()) {
        return;
    }

    // 步骤1：重新 bcrypt 密码并保存到数据库
    // 目的：让 users.password 字段值发生变化（即使密码相同，bcrypt 每次盐值不同，哈希也不同）
    $user->forceFill([
        $attribute => Hash::make($password),
    ])->save();

    // 步骤2：更新当前会话中的 password_hash 指纹
    // 目的：确保当前操作者的会话不被后续的比对逻辑判定为"过期"
    $this->session->put([
        'password_hash_'.$this->getName() => $user->getAuthPassword(),
    ]);

    // ⚠️ 注意：框架本身不会直接删除任何会话文件/记录
    // 全部依赖后续请求时，其他会话的 password_hash 比对失败 → 被中间件踢下线
}
```

**Firefly III 调用分析**：
- ✅ 调用入口正确：[ProfileController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/ProfileController.php#L357)
- ✅ 重新 bcrypt 哈希并保存：通过 `Hash::make($password)` 让 `users.password` 值变化（即使密码没变）
- ✅ 当前会话的 `password_hash_*` 指纹被同步更新
- ❌ **关键缺口**：框架只做了这两件事，后续依赖中间件比对

### 7.2 第二层：会话中间件配置（Firefly III 缺失的关键）

`logoutOtherDevices()` 要生效的**必要前提**是：每个请求都有中间件检查会话中存储的 `password_hash` 与数据库 `users.password` 是否匹配。

**Laravel 提供的标准中间件**：`Illuminate\Auth\Middleware\AuthenticateSession`

其核心逻辑（伪代码还原）：
```php
public function handle($request, Closure $next)
{
    $this->auth->viaRemember(); // 触发 Remember Me 恢复逻辑

    // 关键检查：比对会话指纹与数据库
    if ($this->auth->check() &&
        $request->session()->get('password_hash_'.$this->guard) !== $this->auth->user()->getAuthPassword()
    ) {
        // 不匹配 → 说明密码被改过 → 踢下线
        $this->auth->logout();
        $request->session()->invalidate();
        throw new AuthenticationException;
    }

    return $next($request);
}
```

**Firefly III 的实际中间件配置**（[bootstrap/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/bootstrap/app.php#L95-L105)）：

```php
$middleware->group('web', [
    EncryptCookies::class,
    AddQueuedCookiesToResponse::class,
    StartFireflyIIISession::class,   // ← 自定义，仅改了 storeCurrentUrl
    ShareErrorsFromSession::class,
    VerifyCsrfToken::class,
    Binder::class,
    CreateFreshApiToken::class,
    // ❌ ❌ ❌ 缺少 Illuminate\Auth\Middleware\AuthenticateSession::class
    // ❌ ❌ ❌ 缺少 password_hash 比对逻辑的注入点
]);
```

**Firefly III 自定义 Authenticate 中间件**（[Authenticate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Middleware/Authenticate.php)）的检查项：
- ✅ 检查用户是否登录（`auth()->check()`）
- ✅ 检查用户是否被封锁（`$user->blocked`）
- ❌ **不检查** `password_hash` 指纹比对
- ❌ **不检查** 会话有效性以外的任何条件

**结论（第二层）**：由于缺少 `AuthenticateSession`，即使 `users.password` 哈希变化了，其他设备的会话也不会在请求时被踢下线。`logoutOtherDevices()` 在此配置下对"其他活跃会话的短期失效"**几乎没有作用**。

### 7.3 第三层：其他设备再次请求时的失效判断

其他设备在 `logoutOtherDevices()` 被调用后，再次发起请求时，会经过以下判断链：

```
其他设备浏览器携带 Session Cookie 请求任意页面
    ↓
[EncryptCookies] → 解密 cookie
    ↓
[StartFireflyIIISession] → 从文件存储读取 session（继承 StartSession 默认行为）
    │   ├─ 根据 session ID 读取 storage/framework/sessions/xxxx
    │   ├─ 检查 session 是否过期（默认 lifetime=120 分钟，expire_on_close=true）
    │   └─ 恢复 Session 对象到内存
    ↓
[ShareErrorsFromSession] → 从 session 中读取错误
    ↓
[VerifyCsrfToken] → 验证 CSRF（仅 POST/PUT 等）
    ↓
[Binder] → 路由模型绑定
    ↓
[CreateFreshApiToken] → 刷新 API token cookie
    ↓
[路由匹配] → 命中需要 user-simple-auth 或 user-full-auth 的路由
    ↓
[FireflyIII\Authenticate::handle()]
    ├─ auth()->check() → 根据 session 中的 login_web_* 查找用户
    │   ├─ session 中的 ID 有效 → 返回 User 模型（从 users 表查最新数据）
    │   └─ session 中的 ID 无效 → AuthenticationException → 跳登录页
    ├─ ✅ 检查 $user->blocked === 1 → 踢下线
    └─ ❌ 不比对 password_hash 指纹
    ↓
[后续中间件] → MFA、Range、InterestingMessage 等
    ↓
[控制器] → 正常处理请求（用户仍然被视为登录状态）
```

### 7.4 Firefly III 默认配置下 logoutOtherDevices 的实际效果

| 会话类型 | logoutOtherDevices 后是否立即失效 | 原因分析 |
|----------|:---:|------|
| **其他设备（短期会话，浏览器未关闭）** | ❌ 不失效 | 缺少 AuthenticateSession 中间件，password_hash 不比对；StartSession 仅按 session ID 和过期时间恢复 |
| **其他设备（短期会话，浏览器关闭）** | ⚠️ 大概率失效 | `expire_on_close=true`（[config/session.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/session.php#L28)），关闭浏览器即销毁 session cookie → 重新打开需要再次登录 → 新密码才生效，旧密码无法登录 |
| **其他设备（Remember Me cookie）** | ⚠️ 间接失效 | Laravel `SessionGuard::user()` 在 viaRemember() 时，会**隐式比对密码哈希**（如果 database 中密码与 cookied remember_token 对应的用户记录不匹配 → 恢复失败 → 用户需重新登录） |
| **当前设备（操作者）** | ✅ 不失效 | password_hash 指纹被同步更新 |
| **API Token（Passport）** | ❌ 不失效 | access_token / refresh_token 独立于 web session，完全不受影响 |
| **MFA Cookie** | ❌ 不失效 | `twoFactorRemember` cookie 独立管理，不受 password_hash 变化影响 |

### 7.5 与 Session Driver 的关系

**当前配置**（[config/session.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/session.php#L26)）：
```php
'driver' => env('SESSION_DRIVER', 'file'),  // 默认 file
```

不同 driver 对 logoutOtherDevices 的影响：
- **file（默认）**：session 存储在 `storage/framework/sessions/` 目录，文件名是 session ID。**无法按 user_id 查询**某用户有哪些活跃 session。即使更换为 AuthenticateSession 中间件，也无法主动批量删除。
- **database**：session 存储在 `sessions` 表，有 `user_id` 字段。**理论上可以**在 logoutOtherDevices 中额外执行 `DB::table('sessions')->where('user_id', $userId)->where('id', '!=', $currentId)->delete()`，但 Laravel 框架默认不做。
- **redis**：与 file 类似，键为 session ID，**无法按 user_id 索引**（除非额外维护 user_id → session_ids 的反向索引）。

**在 Firefly III 默认 file driver 下**：即使补上 AuthenticateSession 中间件，也只能在其他设备下次请求时被动失效，无法主动从存储层批量清除。

---

## 8. 主动注销其他会话与自动提醒的区别

### 8.1 本质区别

| 维度 | 主动注销其他会话（实际效果） | 自动安全提醒 |
|------|------------------|--------------|
| **触发方式** | 用户手动操作 | 系统自动检测 |
| **目的** | 试图清除其他设备上的活跃会话 | 通知用户存在异常行为 |
| **是否改变会话状态** | ⚠️ 仅部分生效（详见第7章） | ❌ 不改变任何会话状态 |
| **是否发送通知** | ❌ 不发送通知 | ✅ 发送通知（邮件/Slack/Pushover） |
| **用户感知** | 操作者看到 flash 提示 | 被通知者收到推送/邮件 |
| **代码路径** | `Auth::logoutOtherDevices()` | `event() → Listener → NotificationSender` |

### 8.2 主动注销其他会话的完整流程

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
    │   ├─ [层1] 重新 bcrypt 哈希密码 → users.password 字段变化
    │   ├─ [层1] 当前会话的 password_hash_* 指纹同步更新
    │   ├─ [层2] ❌ 无 AuthenticateSession → 无每次请求的指纹比对
    │   └─ [层3] 其他设备下次请求时：
    │           ├─ 浏览器未关闭 → ❌ 仍然登录（Authenticate不检查指纹）
    │           ├─ 浏览器已关闭，无 Remember Me → ✅ 需重新登录
    │           └─ 有 Remember Me cookie → ⚠️ 多数情况下需重新登录
    ├─ session()->flash('info', '其他会话已注销')
    └─ 验证失败 → session()->flash('error', 'auth.failed')
```

**关键细节**：
- `Auth::once()` 只做单次密码验证，不会影响当前认证状态
- `logoutOtherDevices()` 必须传入明文密码，因为它会重新对密码做 bcrypt 哈希
- 此操作**不触发任何安全事件**，不发通知，仅通过 flash message 告知操作者
- **用户看到 flash 消息以为所有其他会话都失效了，但实际上……**（真实效果见7.4节）

### 8.3 自动安全提醒的完整流程

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

### 8.4 两种机制在不同安全场景下的配合

| 安全场景 | 自动提醒 | 主动注销（实际效果） |
|----------|----------|----------|
| 收到"新IP登录"通知，非本人操作 | 系统自动发送 | 用户操作后，需对方关闭浏览器或下次请求才可能失效 |
| 收到"登录失败"通知 | 系统自动发送 | 无直接关联 |
| 收到"备份码被使用"通知 | 系统自动发送 | 用户可能选择注销+改密，但实际失效有限 |
| 收到"MFA多次失败"通知 | 系统自动发送 | 用户应立即改密+注销，但建议配合修改邮箱等操作 |
| 改密后需清除其他会话 | 无自动提醒 | 用户必须**手动**注销其他会话，但效果有限 |

### 8.5 设计隐患

**改密与 logoutOtherDevices 的实际效果存在双重认知差**：

```
用户视角：
    改密 → 应该所有其他设备都被踢下线了吧？
    点击"注销其他会话" → 应该所有其他设备都立即登出了吧？

实际情况（Firefly III 默认配置）：
    改密 → ❌ 其他设备短期会话完全不受影响
    注销其他会话 → ⚠️ 只有对方关闭浏览器/无Remember Me时才有效
                 ❌ 对方继续使用（浏览器未关闭）则完全不失效

根本原因：
    1. 缺少 AuthenticateSession 中间件 → 每次请求不比对 password_hash 指纹
    2. Session driver 是 file → 无法按 user_id 主动批量删除
    3. 没有调用 remember_token 重置 → Remember Me cookie 未被撤销
```

对比邮箱变更操作的严谨处理：

```
用户变更邮箱（postChangeEmail）
    → 数据库邮箱已更新
    → Auth::guard()->logout()           ← 强制登出（确定生效）
    → $request->session()->invalidate() ← 使当前会话失效（确定生效）
    → 用户必须重新登录
```

**邮箱变更的处理简单且确定生效，logoutOtherDevices 则依赖多层隐含假设。**

---

## 9. IP 地址跟踪与会话异常检测

### 9.1 IP 存储与检测

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

### 9.2 新IP通知发送

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
            NotificationSender::send($user, new UserLogin());
        }
        $list[$index]['notified'] = true;
    }
    Preferences::setForUser($user, 'login_ip_history', $list);
}
```

---

## 10. 完整事件流时序图

### 10.1 登录失败通知流程

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

### 10.2 新IP登录通知流程

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

### 10.3 MFA 异常与备份码完整通知流程

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

### 10.4 会话安全操作对比流程

```
场景A：主动注销其他会话（postLogoutOtherSessions）
    用户 → [POST /profile/logout-other-sessions]
        → Auth::once() 验证密码
        → Auth::logoutOtherDevices($password)
            → 重新 bcrypt 密码，保存到 users 表
            → 更新当前会话的 password_hash 指纹
            → ❌ 其他设备短期会话（浏览器未关闭）不会立即失效
            → ⚠️  关闭浏览器或无Remember Me才需要重新登录
        → flash 提示"已注销"（但效果依赖隐含假设）
    ❌ 不触发事件，不发送通知
    ❌ 没有 remember_token 重置

场景B：改密（postChangePassword）
    用户 → [POST /profile/change-password]
        → 验证当前密码 + 新密码
        → $user->password = bcrypt($new)
        → $user->save()
        → flash 提示"密码已更改"
    ❌ 不触发事件，不发送通知
    ❌ 不调用 logoutOtherDevices()
    ❌ 不使任何会话失效
    ❌ 不重置 remember_token
    ❌ 其他设备上的旧会话仍然完全有效

场景C：变更邮箱（postChangeEmail）
    用户 → [POST /profile/change-email]
        → 更新邮箱
        → event(UserChangedEmailAddress)
        → Auth::guard()->logout()
        → $request->session()->invalidate()
        → 重定向到首页
    ✅ 强制登出当前用户（确定生效）
    ✅ 使当前会话失效（确定生效）
    ✅ 触发事件 → 发送确认邮件 + 撤销邮件
    ⚠️  但不调用 logoutOtherDevices()，其他设备的短期会话可能仍然有效
```

### 10.5 logoutOtherDevices 三层生效判断链

```
[层1] 框架调用
    Auth::logoutOtherDevices($password)
      ├─ users.password = bcrypt($password)  ← 值一定变化（bcrypt盐不同）
      └─ session['password_hash_web'] = $user->password  ← 同步当前会话指纹
    ↓
[层2] 中间件检查（Firefly III ❌ 缺失）
    对每一个进入请求：
    AuthenticateSession（不存在）
      └─ session['password_hash_web'] === $user->password？
          ├─ 相等 → 放行
          └─ 不等 → logout() + invalidate() + 踢下线
    ❌ 由于中间件不存在，此判断链在 Firefly III 中永远不会执行
    ↓
[层3] 其他设备请求时的实际路径
    其他设备发起请求
      ├─ Session Cookie 是否仍存在？
      │   ├─ 浏览器关闭 + expire_on_close=true → Cookie 丢失 → 需重新登录（✅ 生效）
      │   └─ 浏览器未关闭 → Cookie 仍在 → 继续下一步
      ├─ session 文件仍在 storage/framework/sessions？
      │   ├─ 超过 lifetime=120 分钟未活动 → 过期 → 需重新登录（✅ 生效）
      │   └─ session 文件未过期 → 继续下一步
      ├─ Authenticate 中间件检查
      │   ├─ auth()->check() 按 session ID 恢复 → 成功（✅ 仍登录）
      │   └─ $user->blocked 检查 → 通过
      ├─ ❌ password_hash 指纹未被检查 → 继续下一步
      └─ 控制器正常执行 → 会话对用户而言完全有效（❌ 未失效）
```

---

## 11. 关键配置与偏好设置

| 配置项 | 位置 | 作用 |
|--------|------|------|
| `login_ip_history` | 用户偏好 | 存储登录IP历史记录（自动清理6个月前记录） |
| `notification_user_login` | 用户偏好 | 是否开启新IP登录通知（默认true），控制 `UserLoggedInFromNewIpAddress` 事件是否触发 |
| `mfa_failure_count` | 用户偏好 | MFA失败计数器，成功登录或使用备份码后重置为0 |
| `mfa_history` | 用户偏好 | MFA码使用历史（5分钟窗口防重放攻击） |
| `mfa_recovery` | 用户偏好 | 备份码列表，使用后逐个移除 |
| `SESSION_DRIVER` | .env | 会话驱动（默认 file，无法按 user_id 批量删除） |
| `SESSION_LIFETIME` | .env | 会话不活动过期时间（默认 120 分钟） |
| `expire_on_close` | [config/session.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/session.php#L28) | 关闭浏览器即销毁 session cookie（默认 true） |
| `AuthenticateSession::class` | [bootstrap/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/bootstrap/app.php#L95-L105) | 密码哈希指纹比对中间件（**未注册**） |
| `web.guard.remember` | [config/auth.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/auth.php#L67) | Remember Me cookie 有效期（默认 364 天） |

---

## 12. 代码参考索引

### 12.1 控制器
- [LoginController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Auth/LoginController.php) - 登录流程控制
- [TwoFactorController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Auth/TwoFactorController.php) - MFA验证控制 + 备份码级联逻辑
- [MfaController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Profile/MfaController.php) - MFA启用/禁用/备份码管理
- [ProfileController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/ProfileController.php) - 密码修改 + 会话管理（改密/注销其他会话/变更邮箱）

### 12.2 中间件与配置
- [bootstrap/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/bootstrap/app.php#L75-L165) - 全局中间件配置（`AuthenticateSession` **未注册**）
- [Authenticate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Middleware/Authenticate.php) - 自定义认证中间件（只检查 blocked，不检查 password_hash）
- [StartFireflyIIISession.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Middleware/StartFireflyIIISession.php) - 自定义Session启动（仅重写 storeCurrentUrl）
- [config/session.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/session.php) - Session配置（driver=file, expire_on_close=true）
- [config/auth.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/auth.php) - Guard配置（session driver）

### 12.3 事件类
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

### 12.4 监听器类
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

### 12.5 通知类
- [UserFailedLoginAttempt.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/UserFailedLoginAttempt.php)
- [MFAUsedBackupCodeNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFAUsedBackupCodeNotification.php)
- [MFABackupFewLeftNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFABackupFewLeftNotification.php)
- [MFABackupNoLeftNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFABackupNoLeftNotification.php)
- [MFAManyFailedAttemptsNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFAManyFailedAttemptsNotification.php)
- [UserLogin.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/User/UserLogin.php)
- [UnknownUserLoginAttempt.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Admin/UnknownUserLoginAttempt.php)
- [NotificationSender.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/NotificationSender.php)

---

## 13. 设计特点总结

1. **事件驱动架构**：使用 Laravel 事件系统实现松耦合，控制器只需触发事件，无需关心后续处理
2. **自动发现机制**：通过类型提示自动匹配事件和监听器，无需手动注册
3. **队列处理**：所有监听器都实现 `ShouldQueue` 接口，通知发送异步处理，不阻塞登录流程
4. **多渠道通知**：支持邮件、Slack、Pushover 等多种通知渠道
5. **偏好控制**：用户可自主选择是否接收登录通知（`notification_user_login`）
6. **历史记录**：IP历史记录保留6个月，MFA历史记录保留5分钟（防重放），便于审计和异常检测
7. **分级警告**：MFA失败在3次和10次时分别发送警告；备份码剩余 ≤3 和 =0 时分别通知，渐进式提醒
8. **级联触发**：备份码使用后，`removeFromBackupCodes()` 根据剩余数量级联触发不同级别的安全事件
9. **会话安全的三层机制认知差**：
   - 框架层 `logoutOtherDevices()` 只做"密码重哈希+指纹同步"
   - 中间件层缺少 `AuthenticateSession`，导致 password_hash 比对从未执行
   - 请求层只有浏览器关闭/session过期才会让其他设备失效，与用户"点击按钮立即全部下线"的认知不一致
10. **改密与会话处理不一致**：改密不自动注销其他会话、不触发事件、不发送通知，与邮箱变更操作的严谨处理形成明显对比
11. **主动防御与被动通知的分工**：主动注销其他会话试图做"清除"动作（但实际效果有限），自动提醒是"告知"动作（100%确定送达），两者互补但均不能单独解决"账户被盗后立即止损"的问题
