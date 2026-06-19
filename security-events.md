# 安全事件到登录异常提醒的走向分析

## 概述

本文档详细分析 Firefly III 系统中安全事件从触发到用户通知的完整流程，包括事件触发机制、监听器处理、用户通知发送以及会话异常处理的关系。重点拆解了 `logoutOtherDevices()` 的三层生效条件和自动安全提醒的五层送达边界。

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
    } catch (ClientException $e) {
        Log::error(sprintf('[a] Error sending notification: %s', $e->getMessage()));
    } catch (Exception $e) {
        // 按异常消息类型分类处理：Bcc错误、RFC 2822错误、其他错误
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

### 5.1 级联触发源码分析

**入口**：[TwoFactorController.submitMFA()](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Auth/TwoFactorController.php#L107-L121)

```php
if ($this->isBackupCode($mfaCode)) {
    $this->removeFromBackupCodes($mfaCode);   // ← 级联触发更多事件
    $authenticator->login();
    $this->resetMFAFailureCounter();
    event(new UserHasUsedBackupCode($user));   // ← 始终触发
}
```

### 5.2 removeFromBackupCodes 内部的级联逻辑

**源码**：[TwoFactorController.removeFromBackupCodes()](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Auth/TwoFactorController.php#L208-L230)

```php
private function removeFromBackupCodes(string $mfaCode): void
{
    $list    = Preferences::get('mfa_recovery', [])->data;
    $newList = array_values(array_diff($list, [$mfaCode]));

    if (count($newList) <= 3 && count($newList) > 0) {
        event(new UserHasFewMFABackupCodesLeft($user, count($newList)));
    }
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
    ├─ 剩余 ≤3 且 >0 → event(UserHasFewMFABackupCodesLeft)
    │       ↓
    │   [NotifiesUserAboutFewCodesLeft]
    │       → MFABackupFewLeftNotification（含剩余数量）
    │
    └─ 剩余 =0 → event(UserHasNoMFABackupCodesLeft)
            ↓
        [NotifiesUserAboutNoCodesLeft]
            → MFABackupNoLeftNotification
    ↓
[2] authenticator->login()  ← 认证通过
    ↓
[3] event(UserHasUsedBackupCode)  ← 始终触发
        ↓
    [NotifiesUserAboutUsedBackupCode]
        → MFAUsedBackupCodeNotification（含IP/UA/时间）
```

### 5.4 备份码通知的触发条件汇总

| 条件 | 触发的事件 | 通知类 | 紧急程度 |
|------|-----------|--------|----------|
| 每次使用备份码 | `UserHasUsedBackupCode` | [MFAUsedBackupCodeNotification](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFAUsedBackupCodeNotification.php) | ⚠️ 提醒级 |
| 使用后剩余 ≤3 且 >0 | `UserHasFewMFABackupCodesLeft` | [MFABackupFewLeftNotification](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFABackupFewLeftNotification.php) | ⚠️ 警告级 |
| 使用后剩余 =0 | `UserHasNoMFABackupCodesLeft` | [MFABackupNoLeftNotification](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFABackupNoLeftNotification.php) | 🔴 严重级 |

---

## 6. 改密后的会话状态

### 6.1 关键发现：改密后不会主动失效其他会话

**源码**：[ProfileController.postChangePassword()](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/ProfileController.php#L283-L310)

```php
public function postChangePassword(ProfileFormRequest $request, UserRepositoryInterface $repository): RedirectResponse
{
    $repository->changePassword($user, $request->get('new_password'));
    session()->flash('success', (string) trans('firefly.password_changed'));
    return redirect(route('profile.index'));
}
```

**UserRepository.changePassword()** 的实现：
```php
public function changePassword(User $user, #[SensitiveParameter] string $password): bool
{
    $user->password = bcrypt($password);
    $user->save();
    return true;
}
```

**结论**：改密方法**仅更新数据库中的密码哈希**，不执行任何会话操作。

| 行为 | 改密后是否发生 |
|------|:---:|
| 当前会话失效 | ❌ |
| 其他设备会话失效 | ❌ |
| 发送安全通知 | ❌ |
| MFA 状态变化 | ❌ |
| MFA Cookie 清除 | ❌ |

### 6.2 改密与会话安全的真实关系

```
改密操作（postChangePassword）
    ├─ $user->password = bcrypt($new)   ← 更新密码哈希
    ├─ $user->save()                    ← 保存到数据库
    ├─ ❌ 不调用 logoutOtherDevices()
    ├─ ❌ 不调用 session()->invalidate()
    ├─ ❌ 不触发安全事件
    └─ ❌ 不发送通知
    
    后果：其他已登录设备的会话仍然有效
```

与邮箱变更操作的对比（[postChangeEmail()](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/ProfileController.php#L239-L278)）：
```php
// 邮箱变更后：主动强制登出
Auth::guard()->logout();
$request->session()->invalidate();
```

**邮箱变更会强制登出当前用户，但改密不会。**

---

## 7. 主动注销其他会话的三层生效条件（核心拆解）

`Auth::logoutOtherDevices($password)` 是 Laravel `Illuminate\Auth\SessionGuard` 提供的方法，但其**实际效果依赖三个层级的配置**。Firefly III 的默认配置在关键层级上存在缺失。

### 7.1 第一层：框架注销调用（SessionGuard 内部逻辑）

**框架源码逻辑**（伪代码还原）：

```php
public function logoutOtherDevices($password, $attribute = 'password')
{
    if (! $user = $this->user()) {
        return;
    }

    // 步骤1：重新 bcrypt 密码并保存到数据库
    // 即使密码相同，bcrypt 每次盐值不同，哈希也不同
    $user->forceFill([
        $attribute => Hash::make($password),
    ])->save();

    // 步骤2：更新当前会话中的 password_hash 指纹
    // 确保当前操作者的会话不被后续的比对逻辑判定为"过期"
    $this->session->put([
        'password_hash_'.$this->getName() => $user->getAuthPassword(),
    ]);

    // ⚠️ 框架本身不会直接删除任何会话文件/记录
    // 全部依赖后续请求时，其他会话的 password_hash 比对失败 → 被中间件踢下线
}
```

**Firefly III 调用分析**（[ProfileController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/ProfileController.php#L357)）：
- ✅ 调用入口正确
- ✅ 重新 bcrypt 哈希并保存
- ✅ 当前会话的 `password_hash_*` 指纹被同步更新
- ❌ 框架只做了这两件事，后续依赖中间件比对

### 7.2 第二层：会话中间件配置（Firefly III 缺失的关键）

`logoutOtherDevices()` 要生效的**必要前提**是：每个请求都有中间件检查会话中存储的 `password_hash` 与数据库 `users.password` 是否匹配。

**Laravel 提供的标准中间件**：`Illuminate\Auth\Middleware\AuthenticateSession`

核心逻辑（伪代码还原）：
```php
public function handle($request, Closure $next)
{
    $this->auth->viaRemember();

    // 关键检查：比对会话指纹与数据库
    if ($this->auth->check() &&
        $request->session()->get('password_hash_'.$this->guard) !== $this->auth->user()->getAuthPassword()
    ) {
        // 不匹配 → 密码被改过 → 踢下线
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
    StartFireflyIIISession::class,
    ShareErrorsFromSession::class,
    VerifyCsrfToken::class,
    Binder::class,
    CreateFreshApiToken::class,
    // ❌ ❌ ❌ 缺少 Illuminate\Auth\Middleware\AuthenticateSession::class
]);
```

**Firefly III 自定义 Authenticate 中间件**（[Authenticate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Middleware/Authenticate.php)）的检查项：
- ✅ 检查用户是否登录（`auth()->check()`）
- ✅ 检查用户是否被封锁（`$user->blocked`）
- ❌ **不检查** `password_hash` 指纹比对

**结论（第二层）**：由于缺少 `AuthenticateSession`，即使 `users.password` 哈希变化了，其他设备的会话也不会在请求时被踢下线。

### 7.3 第三层：其他设备再次请求时的失效判断

其他设备在 `logoutOtherDevices()` 被调用后，再次发起请求时，会经过以下判断链：

```
其他设备浏览器携带 Session Cookie 请求任意页面
    ↓
[EncryptCookies] → 解密 cookie
    ↓
[StartFireflyIIISession] → 从文件存储读取 session
    │   ├─ 根据 session ID 读取 storage/framework/sessions/xxxx
    │   ├─ 检查 session 是否过期（默认 lifetime=120 分钟，expire_on_close=true）
    │   └─ 恢复 Session 对象到内存
    ↓
[路由匹配] → 命中需要认证的路由
    ↓
[FireflyIII\Authenticate::handle()]
    ├─ auth()->check() → 根据 session 中的 login_web_* 查找用户
    │   ├─ session 中的 ID 有效 → 返回 User 模型（从 users 表查最新数据）
    │   └─ session 中的 ID 无效 → 跳登录页
    ├─ ✅ 检查 $user->blocked === 1 → 踢下线
    └─ ❌ 不比对 password_hash 指纹
    ↓
[控制器] → 正常处理请求（用户仍然被视为登录状态）
```

### 7.4 Firefly III 默认配置下 logoutOtherDevices 的实际效果

| 会话类型 | logoutOtherDevices 后是否立即失效 | 原因分析 |
|----------|:---:|------|
| **其他设备（短期会话，浏览器未关闭）** | ❌ 不失效 | 缺少 AuthenticateSession 中间件，password_hash 不比对 |
| **其他设备（短期会话，浏览器关闭）** | ⚠️ 大概率失效 | `expire_on_close=true`（[config/session.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/session.php#L28)），关闭浏览器即销毁 session cookie |
| **其他设备（Remember Me cookie）** | ⚠️ 间接失效 | Laravel `SessionGuard::user()` 在 viaRemember() 时，会隐式比对密码哈希 |
| **当前设备（操作者）** | ✅ 不失效 | password_hash 指纹被同步更新 |
| **API Token（Passport）** | ❌ 不失效 | access_token / refresh_token 独立于 web session |
| **MFA Cookie** | ❌ 不失效 | `twoFactorRemember` cookie 独立管理 |

### 7.5 与 Session Driver 的关系

**当前配置**（[config/session.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/session.php#L26)）：
```php
'driver' => env('SESSION_DRIVER', 'file'),  // 默认 file
```

不同 driver 对 logoutOtherDevices 的影响：
- **file（默认）**：session 存储在 `storage/framework/sessions/` 目录，文件名是 session ID。**无法按 user_id 查询**。
- **database**：session 存储在 `sessions` 表，有 `user_id` 字段。**理论上可以**额外执行批量删除，但 Laravel 框架默认不做。
- **redis**：与 file 类似，键为 session ID，**无法按 user_id 索引**。

**在 Firefly III 默认 file driver 下**：即使补上 AuthenticateSession 中间件，也只能在其他设备下次请求时被动失效，无法主动从存储层批量清除。

---

## 8. 自动安全提醒的五层送达边界（核心补全）

自动安全提醒的送达不是"事件触发就一定收到通知"，而是经过**五层边界**的逐层过滤。任何一层的条件不满足或出现异常，都可能导致通知无法送达。

### 8.1 五层送达边界总览

```
安全事件触发
    ↓
[层1] 事件触发层 → 偏好控制 + Demo用户跳过
    ↓
[层2] 监听器层 → ShouldQueue + 无重试配置
    ↓
[层3] 渠道过滤层 → ReturnsAvailableChannels + 演示站点过滤
    ↓
[层4] 发送层 → NotificationSender 异常捕获 + 不重抛
    ↓
[层5] 去重层 → notified 标志（仅限新IP登录）
    ↓
用户收到通知（或不收到，且不知道）
```

### 8.2 第一层：用户偏好控制与 Demo 用户跳过

这一层决定**事件是否触发**或**监听器是否继续执行**。

#### 8.2.1 StoresNewIpAddress 中的双重过滤

**源码**：[StoresNewIpAddress.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/StoresNewIpAddress.php#L41-L78)

```php
public function handle(UserSuccessfullyLoggedIn $event): void
{
    $user = $event->user;

    // 边界1：Demo用户 → 直接 return，不记录IP，不触发新IP事件
    if ($user->hasRole('demo')) {
        Log::debug('Do not log demo user logins');
        return;
    }

    // ... 记录IP逻辑 ...

    // 边界2：用户偏好 notification_user_login（默认 true）
    $send = Preferences::getForUser($user, 'notification_user_login', true)->data;
    Preferences::setForUser($user, 'login_ip_history', $preference);

    // 只有新IP + 用户开启通知，才触发 UserLoggedInFromNewIpAddress 事件
    if (false === $inArray && true === $send) {
        event(new UserLoggedInFromNewIpAddress($user));
    }
}
```

**关键细节**：
- Demo 用户在这一层就被拦截，`UserLoggedInFromNewIpAddress` 事件根本不会触发
- 即使 IP 是新的，如果用户关闭了 `notification_user_login` 偏好，事件也不会触发
- 这一层是**事件触发前**的过滤

#### 8.2.2 NotifiesUserAboutNewIpAddress 中的 Demo 再次检查

**源码**：[NotifiesUserAboutNewIpAddress.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutNewIpAddress.php#L39-L41)

```php
public function handle(UserLoggedInFromNewIpAddress $event): void
{
    $user = $event->user;

    // 边界：Demo用户 → 即使事件触发了也不发送
    if ($user->hasRole('demo')) {
        return; // do not email demo user.
    }
```

**关键细节**：
- 即使事件绕过了第一层（比如手动触发），这一层还有第二次拦截
- 双重保险，确保 Demo 用户不会收到新IP登录通知

#### 8.2.3 其他监听器的 Demo 用户检查情况

| 监听器 | 是否检查 Demo 用户 | 代码位置 |
|--------|:---:|--------|
| `StoresNewIpAddress` | ✅ 是 | 第41行 |
| `NotifiesUserAboutNewIpAddress` | ✅ 是 | 第39行 |
| `NotifiesUserAboutFailedLogin` | ❌ 否 | - |
| `NotifiesUserAboutUsedBackupCode` | ❌ 否 | - |
| `NotifiesUserAboutRepeatedMFAFailures` | ❌ 否 | - |
| `NotifiesUserAboutFewCodesLeft` | ❌ 否 | - |
| `NotifiesUserAboutNoCodesLeft` | ❌ 否 | - |
| `NotifiesUserAboutEnabledMFA` | ❌ 否 | - |
| `NotifiesUserAboutDisabledMFA` | ❌ 否 | - |
| `NotifiesUserAboutNewBackupCodes` | ❌ 否 | - |
| `NotifiesOwnerAboutUnknownUser` | N/A | 发给所有者，不发给用户 |

**重要发现**：只有"新IP登录"相关的两个监听器检查 Demo 用户，其他所有安全通知（登录失败、备份码使用、MFA多次失败等）都**不检查 Demo 用户**。这意味着 Demo 用户仍然会收到这些通知。

### 8.3 第二层：队列监听器

这一层决定**通知何时发送**以及**发送失败后的重试行为**。

#### 8.3.1 所有安全监听器都实现 ShouldQueue

```php
// 所有安全监听器都有这个接口
class NotifiesUserAboutFailedLogin implements ShouldQueue
```

#### 8.3.2 队列连接配置

**源码**：[config/queue.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/queue.php#L37)

```php
'default' => env('QUEUE_CONNECTION', 'sync'),  // 默认 sync
```

| 模式 | 表现 | 对登录流程的影响 |
|------|------|----------------|
| **sync（默认）** | 同步执行，通知发送在登录请求线程内完成 | ❌ 发送失败会阻塞登录流程（等待SMTP/Guzzle超时） |
| **database/redis** | 异步执行，通知写入队列，由 worker 处理 | ✅ 不阻塞登录流程 |

#### 8.3.3 无自定义重试配置

所有安全监听器都**没有**定义以下属性：
- `public $tries = 3;` → Laravel 默认 3 次（但被 try-catch 吃掉了，见下一层）
- `public $backoff = [30, 60, 120];` → 无回退时间
- `public $maxExceptions = 3;` → 无最大异常数
- `public function failed(Throwable $exception) { }` → 无失败处理方法

**实际表现**：监听器类本身没有配置任何重试逻辑。

### 8.4 第三层：通知渠道过滤

这一层决定**通过哪些渠道发送通知**。

#### 8.4.1 ReturnsAvailableChannels 渠道选择器

**源码**：[ReturnsAvailableChannels.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/ReturnsAvailableChannels.php)

```php
public static function returnChannels(string $type, ?User $user = null): array
{
    $channels = ['mail'];  // mail 是基础渠道，始终启用

    if ('user' === $type && $user instanceof User) {
        return self::returnUserChannels($user);
    }
    return $channels;
}

private static function returnUserChannels(User $user): array
{
    $channels = ['mail'];

    // Slack：需配置启用 + webhook URL 有效
    if (true === config('notifications.channels.slack.enabled', false)) {
        $slackUrl = (string) Preferences::getEncryptedForUser($user, 'slack_webhook_url', '')->data;
        if (UrlValidator::isValidWebhookURL($slackUrl)) {
            $channels[] = 'slack';
        }
    }

    // Pushover：需配置启用 + app token + user token 都非空
    if (true === config('notifications.channels.pushover.enabled', false)) {
        $pushoverAppToken  = (string) Preferences::getEncryptedForUser($user, 'pushover_app_token', '')->data;
        $pushoverUserToken = (string) Preferences::getEncryptedForUser($user, 'pushover_user_token', '')->data;
        if ('' !== $pushoverAppToken && '' !== $pushoverUserToken) {
            $channels[] = PushoverChannel::class;
        }
    }

    return $channels;
}
```

**关键细节**：
- `mail` 渠道始终启用
- `slack` 需要：`notifications.channels.slack.enabled=true` + 有效的 webhook URL
- `pushover` 需要：`notifications.channels.pushover.enabled=true` + app token + user token 都非空
- `ntfy` 渠道：代码已注释，实际不可用

#### 8.4.2 通知类 via() 方法的额外过滤

以 [UserFailedLoginAttempt.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/UserFailedLoginAttempt.php#L114-L123) 为例：

```php
public function via(User $notifiable): array
{
    $channels   = ReturnsAvailableChannels::returnChannels('user', $notifiable);
    $isDemoSite = FireflyConfig::get('is_demo_site', false)->data;
    
    // 演示站点：排除 mail 渠道，只剩 Slack/Pushover（如果配置了）
    if (true === $isDemoSite) {
        return array_diff($channels, ['mail']);
    }

    return $channels;
}
```

**其他通知类**（新IP登录、备份码使用等）的 `via()` 方法没有这层过滤，直接返回 `ReturnsAvailableChannels::returnChannels()` 的结果。

**渠道过滤汇总**：

| 通知类 | 演示站点是否排除 mail | 代码位置 |
|--------|:---:|--------|
| `UserFailedLoginAttempt` | ✅ 是 | 第117-120行 |
| `UserLogin`（新IP） | ❌ 否 | 第112-115行 |
| `MFAUsedBackupCodeNotification` | ❌ 否 | 第110-113行 |
| `MFAManyFailedAttemptsNotification` | ❌ 否 | - |
| 其他 MFA 相关通知 | ❌ 否 | - |

### 8.5 第四层：通知发送异常捕获（吃掉异常导致不重试）

这一层是**送达边界中最关键的一环**，也是最容易导致通知"静默失败"的地方。

**源码**：[NotificationSender.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/NotificationSender.php#L48-L68)

```php
public static function send(OwnerNotifiable|User $user, Notification $notification): void
{
    // ... 语言设置 ...

    try {
        NotificationFacade::locale($lang)->send($user, $notification);
    } catch (ClientException $e) {
        // Guzzle HTTP 客户端异常（Slack/Pushover 等 Webhook 失败）
        Log::error(sprintf('[a] Error sending notification: %s', $e->getMessage()));
        // ❌ 不重新抛出异常
    } catch (Exception $e) {
        $message = $e->getMessage();
        if (str_contains($message, 'Bcc')) {
            // 邮件 Bcc 配置错误
            Log::warning('[Bcc] Could not send notification. Please validate your email settings...');
            return;  // ❌ 不重新抛出异常
        }
        if (str_contains($message, 'RFC 2822')) {
            // 邮箱格式不符合 RFC 2822 标准
            Log::warning('[RFC] Could not send notification. Please validate your email settings...');
            return;  // ❌ 不重新抛出异常
        }
        // 其他所有异常
        Log::error('Could not send notification :(.');
        Log::error($e->getMessage());
        Log::error($e->getTraceAsString());
        // ❌ 不重新抛出异常
    }
    // 方法结束，没有抛出任何异常
}
```

**关键发现**：
- ✅ **所有异常都被捕获**：`ClientException`（Guzzle）和 `Exception`（其他所有异常）
- ❌ **所有异常都不重新抛出**：catch 块中没有 `throw $e;`
- ✅ **只有日志记录**：所有错误信息只写入 `storage/logs/laravel.log`
- ❌ **队列任务标记为"已成功"**：因为没有异常抛出，Laravel 队列认为任务执行成功
- ❌ **不会触发 Laravel 队列重试机制**：即使 `$tries = 3` 也不会重试，因为没有异常
- ❌ **不会调用监听器的 failed() 方法**：没有异常就不会触发
- ❌ **不会写入 failed_jobs 表**：任务没有"失败"，只是静默地没有送达

**对队列重试的影响**：

| 模式 | 异常被捕获后的表现 | 是否重试 |
|------|------------------|---------|
| **sync** | 异常被吃掉，登录流程继续（但被阻塞了SMTP超时时间） | ❌ 不重试 |
| **database/redis** | 队列 worker 捕获到正常返回，删除队列任务，标记为成功 | ❌ 不重试 |
| **database/redis（worker未运行）** | 任务永远 pending，不会被处理 | ⚠️ 需重启 worker |

### 8.6 第五层：去重机制（notified 标志）

这一层决定**同一条异常是否发送多次通知**。

#### 8.6.1 新IP登录：有 notified 标志去重

**源码**：[NotifiesUserAboutNewIpAddress.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutNewIpAddress.php#L49-L57)

```php
foreach ($list as $index => $entry) {
    if (false === $entry['notified']) {
        NotificationSender::send($user, new UserLogin());
    }
    // ⚠️ 无论发送成功还是失败，都标记为已通知
    $list[$index]['notified'] = true;
}
Preferences::setForUser($user, 'login_ip_history', $list);
```

**关键缺陷**：
- `NotificationSender::send()` 即使发送失败也会正常返回（因为异常被吃掉了）
- 然后代码立即执行 `$list[$index]['notified'] = true`
- **即使发送完全失败**，这条 IP 记录也被标记为"已通知"
- 不会有第二次发送机会，用户永远收不到这条通知，也不知道有新IP登录

#### 8.6.2 其他通知：无去重机制

所有其他安全通知（登录失败、备份码使用、MFA多次失败等）都没有去重逻辑。每次事件触发，就调用一次 `NotificationSender::send()`。

**示例**：
- MFA 失败 3 次 → 触发 `UserKeepsFailingMFA` → 发送一次 `MFAManyFailedAttemptsNotification`
- MFA 失败 10 次 → 再次触发 `UserKeepsFailingMFA` → 再发送一次（含失败次数=10）
- 这是设计行为（分级警告），不是缺陷

### 8.7 发送失败时的实际表现汇总

| 失败场景 | 用户感知 | 日志记录 | 重试机制 | 对登录流程的影响 |
|---------|---------|---------|---------|----------------|
| SMTP 服务器无响应/超时 | ❌ 收不到邮件，无任何提示 | ✅ `storage/logs/laravel.log` 有 error | ❌ 不重试 | ❌ sync 模式阻塞登录 2-3 秒 |
| Pushover token 无效 | ❌ 收不到推送，无任何提示 | ✅ error log | ❌ 不重试 | ❌ sync 模式阻塞登录（Guzzle 超时） |
| Slack webhook URL 失效 | ❌ 收不到通知，无任何提示 | ✅ error log | ❌ 不重试 | ❌ sync 模式阻塞登录 |
| Bcc 配置错误 | ❌ 收不到邮件，无任何提示 | ✅ warning log（带配置引导） | ❌ 不重试 | ❌ sync 模式阻塞登录 |
| RFC 2822 邮箱格式错误 | ❌ 收不到邮件，无任何提示 | ✅ warning log（带配置引导） | ❌ 不重试 | ❌ sync 模式阻塞登录 |
| database 队列 worker 未运行 | ❌ 永远收不到通知 | ⚠️ 无 log（任务 pending） | ⚠️ 需重启 supervisor | ✅ 不阻塞登录（异步） |
| 通知类语法错误/依赖缺失 | ❌ 收不到通知，无任何提示 | ✅ error log + 堆栈 | ❌ 不重试 | ❌ sync 模式阻塞登录 |

### 8.8 通知送达的不确定性总览

```
安全事件触发（如 UserFailedLoginAttempt）
    ↓
[层1 事件触发层]
    ├─ 监听器检查用户类型
    │   ├─ 新IP登录 + Demo用户 → ❌ return（在 StoresNewIpAddress 就被拦截）
    │   ├─ 新IP登录 + 关闭 notification_user_login → ❌ 不触发新IP事件
    │   └─ 其他通知 → ✅ 继续（不检查 Demo）
    ↓
[层2 监听器层]
    ├─ implements ShouldQueue
    ├─ QUEUE_CONNECTION=sync → 同步调用
    │   └─ 发送过程中抛出异常 → 下一层捕获
    └─ QUEUE_CONNECTION=database → 入队列
        └─ worker 未运行 → ❌ 永远不处理
    ↓
[层3 渠道过滤层]
    ├─ ReturnsAvailableChannels → 过滤掉未配置的渠道
    │   ├─ mail → 始终有
    │   ├─ slack → 需 enabled=true + webhook 有效
    │   └─ pushover → 需 enabled=true + 两个 token 非空
    └─ 通知类 via()
        └─ UserFailedLoginAttempt + 演示站点 → ❌ 排除 mail 渠道
    ↓
[层4 发送层]
    ├─ NotificationFacade::send()
    │   ├─ 对每个渠道调用 toMail()/toSlack()/toPushover()
    │   └─ 单个渠道失败 → Laravel 继续尝试其他渠道
    └─ NotificationSender try-catch
        ├─ ClientException → Log::error
        ├─ Bcc 异常 → Log::warning + return
        ├─ RFC 2822 异常 → Log::warning + return
        └─ 其他异常 → Log::error + 堆栈
        → 所有情况都 ❌ 不重新抛出异常
        → 队列任务标记成功 → ❌ 不重试
    ↓
[层5 去重层]
    ├─ 新IP登录 → 标记 notified=true（⚠️ 即使发送失败也标记）
    └─ 其他通知 → 无去重，事件触发多少次就发送多少次
```

**送达保证级别**：**尽力而为（Best Effort）**
- ❌ 不是至少一次（At-least-once）——发送失败不会重试
- ❌ 不是恰好一次（Exactly-once）——sync模式下异常捕获前可能已经发出去了
- ✅ 是最多一次（At-most-once）——发送失败后标记为已通知，不会重发
- ⚠️ 用户对送达失败**完全不知情**，只能通过日志发现

---

## 9. 主动注销其他会话与自动提醒的区别

### 9.1 本质区别

| 维度 | 主动注销其他会话（实际效果） | 自动安全提醒 |
|------|------------------|--------------|
| **触发方式** | 用户手动操作 | 系统自动检测 |
| **目的** | 试图清除其他设备上的活跃会话 | 通知用户存在异常行为 |
| **是否改变会话状态** | ⚠️ 仅部分生效（详见第7章） | ❌ 不改变任何会话状态 |
| **是否发送通知** | ❌ 不发送通知 | ✅ 发送通知（但送达边界见第8章） |
| **用户感知** | 操作者看到 flash 提示 | 被通知者可能收到推送/邮件，也可能静默失败 |
| **代码路径** | `Auth::logoutOtherDevices()` | `event() → Listener → NotificationSender` |

### 9.2 主动注销其他会话的完整流程

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
    │           ├─ 浏览器未关闭 → ❌ 仍然登录
    │           ├─ 浏览器已关闭，无 Remember Me → ✅ 需重新登录
    │           └─ 有 Remember Me cookie → ⚠️ 多数情况下需重新登录
    ├─ session()->flash('info', '其他会话已注销')
    └─ 验证失败 → session()->flash('error', 'auth.failed')
```

**关键细节**：
- `Auth::once()` 只做单次密码验证，不会影响当前认证状态
- `logoutOtherDevices()` 必须传入明文密码，因为它会重新对密码做 bcrypt 哈希
- 此操作**不触发任何安全事件**，不发通知，仅通过 flash message 告知操作者

---

## 10. IP 地址跟踪与会话异常检测

### 10.1 IP 存储与检测

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

### 10.2 新IP通知发送

**核心逻辑**：[NotifiesUserAboutNewIpAddress.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Listeners/Security/User/NotifiesUserAboutNewIpAddress.php#L35-L58)

```php
public function handle(UserLoggedInFromNewIpAddress $event): void
{
    $user = $event->user;
    if ($user->hasRole('demo')) {
        return;
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

## 11. 完整事件流时序图

### 11.1 登录失败通知流程

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
    │       └─ 邮件/Slack/Pushover 通知（经过5层送达边界）
    └─ NotifiesOwnerAboutUnknownUser
        └─ NotificationSender::send(UnknownUserLoginAttempt)
            └─ 发送给系统所有者
```

### 11.2 新IP登录通知流程

```
用户登录成功
    ↓
event(UserSuccessfullyLoggedIn)
    ↓
[StoresNewIpAddress.handle]
    ├─ Demo用户 → return
    ├─ 检查IP历史
    ├─ IP已存在 → 更新时间戳
    └─ IP不存在 → 添加记录（notified=false）
        └─ 用户开启通知偏好 → event(UserLoggedInFromNewIpAddress)
            ↓
[NotifiesUserAboutNewIpAddress.handle]
    ├─ Demo用户 → return
    ├─ 查找未通知的IP记录
    ├─ 发送 UserLogin 通知（经过5层送达边界）
    └─ 标记 notified=true（即使发送失败）
```

### 11.3 MFA 异常与备份码完整通知流程

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
    │   │       → MFAManyFailedAttemptsNotification（经过5层边界）
    │   └─ 其他次数 → 仅增加计数，不触发事件
    └─ 检测为备份码 → [removeFromBackupCodes]
        ├─ 剩余 ≤3 且 >0 → event(UserHasFewMFABackupCodesLeft)
        │       ↓
        │   [NotifiesUserAboutFewCodesLeft]
        │       → MFABackupFewLeftNotification（经过5层边界）
        │
        ├─ 剩余 =0 → event(UserHasNoMFABackupCodesLeft)
        │       ↓
        │   [NotifiesUserAboutNoCodesLeft]
        │       → MFABackupNoLeftNotification（经过5层边界）
        │
        └─ 保存新列表
        ↓
    authenticator->login()  ← 认证通过
    resetMFAFailureCounter()
    ↓
    event(UserHasUsedBackupCode)   ← 始终触发
        ↓
    [NotifiesUserAboutUsedBackupCode]
        → MFAUsedBackupCodeNotification（经过5层边界）
```

---

## 12. 关键配置与偏好设置

| 配置项 | 位置 | 作用 |
|--------|------|------|
| `login_ip_history` | 用户偏好 | 存储登录IP历史记录（自动清理6个月前记录，带 `notified` 标志） |
| `notification_user_login` | 用户偏好 | 是否开启新IP登录通知（默认true），控制 `UserLoggedInFromNewIpAddress` 事件是否触发 |
| `mfa_failure_count` | 用户偏好 | MFA失败计数器，成功登录或使用备份码后重置为0 |
| `mfa_history` | 用户偏好 | MFA码使用历史（5分钟窗口防重放攻击） |
| `mfa_recovery` | 用户偏好 | 备份码列表，使用后逐个移除 |
| `SESSION_DRIVER` | .env | 会话驱动（默认 file，无法按 user_id 批量删除） |
| `QUEUE_CONNECTION` | .env | 队列连接（默认 sync，通知发送同步阻塞登录） |
| `expire_on_close` | [config/session.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/session.php#L28) | 关闭浏览器即销毁 session cookie（默认 true） |
| `AuthenticateSession::class` | [bootstrap/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/bootstrap/app.php#L95-L105) | 密码哈希指纹比对中间件（**未注册**） |
| `notifications.channels.slack.enabled` | [config/notifications.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/notifications.php) | Slack 通知全局开关 |
| `notifications.channels.pushover.enabled` | [config/notifications.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/notifications.php) | Pushover 通知全局开关 |
| `is_demo_site` | FireflyConfig | 是否为演示站点（影响 UserFailedLoginAttempt 的 mail 渠道排除） |

---

## 13. 代码参考索引

### 13.1 控制器
- [LoginController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Auth/LoginController.php) - 登录流程控制
- [TwoFactorController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Auth/TwoFactorController.php) - MFA验证控制 + 备份码级联逻辑
- [MfaController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/Profile/MfaController.php) - MFA启用/禁用/备份码管理
- [ProfileController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Controllers/ProfileController.php) - 密码修改 + 会话管理

### 13.2 中间件与配置
- [bootstrap/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/bootstrap/app.php#L75-L165) - 全局中间件配置（`AuthenticateSession` **未注册**）
- [Authenticate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Middleware/Authenticate.php) - 自定义认证中间件（只检查 blocked，不检查 password_hash）
- [StartFireflyIIISession.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Http/Middleware/StartFireflyIIISession.php) - 自定义Session启动
- [config/session.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/session.php) - Session配置
- [config/queue.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/queue.php) - 队列配置
- [config/notifications.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/config/notifications.php) - 通知渠道配置

### 13.3 事件类
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

### 13.4 监听器类
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

### 13.5 通知类与发送器
- [UserFailedLoginAttempt.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/UserFailedLoginAttempt.php)
- [MFAUsedBackupCodeNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFAUsedBackupCodeNotification.php)
- [MFABackupFewLeftNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFABackupFewLeftNotification.php)
- [MFABackupNoLeftNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFABackupNoLeftNotification.php)
- [MFAManyFailedAttemptsNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Security/MFAManyFailedAttemptsNotification.php)
- [UserLogin.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/User/UserLogin.php)
- [UnknownUserLoginAttempt.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/Admin/UnknownUserLoginAttempt.php)
- [NotificationSender.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/NotificationSender.php) - 异常捕获逻辑（第48-68行）
- [ReturnsAvailableChannels.php](file:///d:/fz/0601-2/solo-dogfeeding/code/45-firefly-iii/app/Notifications/ReturnsAvailableChannels.php) - 渠道选择器

---

## 14. 设计特点总结

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
11. **通知送达的五层边界**（核心补全）：
    - 层1：Demo用户跳过（仅新IP登录有，其他通知无）+ 用户偏好控制
    - 层2：`ShouldQueue` 但默认 sync 模式阻塞登录，无自定义重试配置
    - 层3：渠道过滤（mail始终启用，Slack/Pushover需配置）+ 演示站点排除mail
    - 层4：`NotificationSender` 的 try-catch 吃掉所有异常，不重抛导致队列不重试
    - 层5：新IP登录有 `notified` 去重但**发送失败也标记**，其他通知无去重
12. **尽力而为的送达保证**：通知送达是 Best Eff