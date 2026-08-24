# Android 防沉迷 / 防熬夜 App V1 开发规划

## 1. 项目目标

开发一款仅用于个人 Android 设备的防沉迷 / 防熬夜应用。

核心理念：

> **在清醒、自律的时候制定规则，由系统在容易沉迷的时候替自己执行规则。**

应用不关注“今天用了多少分钟抖音”“哪个 App 使用时间最长”等传统数字健康统计。

V1 只解决一个问题：

> **到达设定时间后，自动让手机变得不好玩。**

整个过程不依赖用户主动点击“开始防沉迷”。

---

# 2. 核心使用流程

例如设置：

```text
防沉迷时间：

23:00 → 次日 07:00

提前提醒：

10 分钟

白名单：

电话
短信
时钟
相机
计算器
日历
其他用户指定 App
```

每天自动执行：

```text
22:50
│
▼
全屏预告

“距离防沉迷开启还有 10 分钟”
│
▼
23:00
│
▼
自动进入防沉迷模式
│
▼
暂停所有非白名单 App
│
▼
手机进入低可用状态
│
├── 电话      ✓
├── 闹钟      ✓
├── 相机      ✓
├── 计算器    ✓
│
├── 抖音      ✗
├── B站       ✗
├── 游戏      ✗
└── 其他 App  ✗

用户仍尝试继续使用手机
│
▼
全屏强警告

“现在已经 00:37”
“距离 07:00 起床只剩 6 小时 23 分钟”
“继续刷手机是在透支明天。”

│
▼
07:00
│
▼
自动解除防沉迷
│
▼
恢复被暂停 App
```

---

# 3. V1 功能范围

V1 严格控制功能范围，不开发传统“数字健康”类复杂功能。

## 3.1 防沉迷时间规则

支持：

- 开启 / 关闭防沉迷

- 设置开始时间

- 设置结束时间

- 支持跨天

- 设置生效星期

- 设置提前提醒时间

例如：

```text
开启：✓

开始：
23:00

结束：
07:00

重复：
周一 周二 周三 周四 周五 周六 周日

提前提醒：
10 分钟
```

核心数据模型：

```kotlin
data class FocusRule(
    val enabled: Boolean,
    val startTime: LocalTime,
    val endTime: LocalTime,
    val days: Set<DayOfWeek>,
    val preWarningMinutes: Int
)
```

---

# 4. 自动触发

防沉迷不能依赖用户手动开启。

用户只负责：

```text
第一次配置规则
```

之后全部由系统自动执行。

核心流程：

```text
AlarmManager
│
├── 22:50 → PRE_WARNING
│
├── 23:00 → ENTER_FOCUS
│
└── 07:00 → EXIT_FOCUS
```

不使用：

```text
while(true)
Service 每秒检查
常驻前台 Service
```

采用事件驱动方式。

---

# 5. 白名单

采用：

> **默认限制 + 白名单放行**

不设计：

```text
娱乐 App
学习 App
工作 App
工具 App
```

等分类体系。

系统只判断：

```text
packageName 是否存在于白名单？
```

例如：

```text
com.android.dialer
com.android.mms
com.android.deskclock
com.android.camera
com.android.calculator

+
用户手动选择的 App
```

Focus App 自身必须始终处于保护名单。

---

# 6. 应用限制

进入防沉迷：

```text
获取已安装 App
        │
        ▼
排除系统不可限制应用
        │
        ▼
排除 Focus App
        │
        ▼
排除白名单
        │
        ▼
剩余应用
        │
        ▼
DevicePolicyManager
        │
        ▼
setPackagesSuspended()
```

退出防沉迷：

```text
读取本次由 Focus App 暂停的 Package
        │
        ▼
setPackagesSuspended(false)
```

注意：

> 不要退出时粗暴恢复所有应用。

必须记录：

```text
Focus App 本次 suspend 了哪些 Package
```

只恢复自己修改过的状态。

---

# 7. FullScreenIntervention

“防沉迷开启预告”和“防沉迷期间强警告”统一属于：

```text
FullScreenIntervention
```

职责：

> 在关键时间节点通过全屏 UI 强制打断当前手机使用行为。

统一定义：

```kotlin
enum class InterventionType {

    PRE_WARNING,

    ACTIVE_WARNING
}
```

---

## 7.1 PRE_WARNING

防沉迷开启之前出现。

例如：

```text
━━━━━━━━━━━━━━━━━━━━

        22:50

距离防沉迷开启还有

        10:00

请结束当前内容，
准备放下手机。

━━━━━━━━━━━━━━━━━━━━
```

V1 默认：

```text
提前 10 分钟
```

用户可以修改提前时间。

---

## 7.2 ACTIVE_WARNING

已经进入防沉迷状态后，用于强干预。

例如：

```text
━━━━━━━━━━━━━━━━━━━━

        00:37

你已经晚睡

     1 小时 37 分钟

距离 07:00 起床

只剩 6 小时 23 分钟


继续刷手机，

是在透支明天。

━━━━━━━━━━━━━━━━━━━━
```

后续可以支持：

```text
动态时间
随机警告文案
震动
声音
动画
倒计时
```

V1 优先完成：

```text
全屏
+
时间计算
+
强提醒文案
```

---

# 8. 紧急解除

不能设计成真正无法退出。

必须考虑：

```text
紧急工作
紧急联系人
扫码
支付
导航
设备异常
```

但也不能：

```text
点击一下
↓
关闭防沉迷
```

建议：

```text
紧急解除
│
▼
长按 10 秒
│
▼
等待 60 秒
│
▼
再次确认
│
▼
解除本次防沉迷
```

目的不是提供绝对安全，而是：

> **增加沉迷状态下解除规则的行为成本。**

---

# 9. 防沉迷期间配置保护

正常状态：

```text
修改时间        ✓
修改星期        ✓
修改白名单      ✓
关闭规则        ✓
```

防沉迷状态：

```text
修改时间        ✗
修改星期        ✗
修改白名单      ✗
关闭规则        ✗

紧急解除        ✓
```

避免出现：

```text
23:00

“再刷一会……”

↓

打开 Focus

↓

23:00 - 07:00

改成：

02:00 - 07:00
```

从而绕过规则。

---

# 10. 系统状态恢复

不能假设 Alarm 一定执行。

系统应该始终能够根据：

```text
当前时间
+
FocusRule
```

重新计算正确状态。

统一入口：

```kotlin
fun reconcileFocusState()
```

逻辑：

```text
读取 FocusRule
        │
        ▼
RuleEngine.shouldFocusNow()
        │
        ├── TRUE
        │      ↓
        │   确保 Focus 已开启
        │
        └── FALSE
               ↓
            确保 Focus 已关闭
```

以下事件发生时重新计算：

```text
BOOT_COMPLETED

TIME_CHANGED

TIMEZONE_CHANGED

规则修改

Alarm 触发
```

---

# 11. 总体模块架构

```text
Focus App
│
├── presentation
│
│   ├── Home
│   ├── RuleSetting
│   ├── WhiteList
│   ├── FullScreenIntervention
│   └── EmergencyExit
│
├── domain
│
│   ├── RuleEngine
│   ├── FocusState
│   ├── FocusManager
│   └── InterventionEngine
│
├── device
│
│   ├── DeviceController
│   ├── AppController
│   └── DeviceOwnerManager
│
├── scheduler
│
│   ├── FocusScheduler
│   ├── AlarmReceiver
│   ├── BootReceiver
│   └── TimeChangeReceiver
│
└── data
    │
    ├── FocusRuleRepository
    ├── WhiteListRepository
    └── DataStore
```

---

# 12. 核心模块职责

## RuleEngine

负责：

> **现在是否应该处于防沉迷状态？**

输入：

```text
FocusRule
CurrentTime
```

输出：

```text
true / false
```

不依赖 Android Framework。

方便单元测试。

---

## FocusManager

整个业务的核心协调器。

负责：

```text
enterFocusMode()

exitFocusMode()

reconcileFocusState()
```

关系：

```text
RuleEngine
    │
    ▼
FocusManager
    │
    ├── DeviceController
    │
    ├── Scheduler
    │
    └── Repository
```

---

## DeviceController

负责真正修改设备状态。

```kotlin
interface DeviceController {

    suspend fun enterFocusMode()

    suspend fun exitFocusMode()
}
```

内部主要使用：

```text
DevicePolicyManager
PackageManager
```

---

## FocusScheduler

负责：

> 下一次什么时候执行任务？

例如：

```text
当前：
20:00

规则：
23:00 - 07:00

计算：

22:50 PRE_WARNING
23:00 ENTER_FOCUS
07:00 EXIT_FOCUS
```

底层：

```text
AlarmManager
```

---

## FullScreenIntervention

负责全屏行为干预。

```text
PRE_WARNING

ACTIVE_WARNING
```

负责：

```text
全屏 UI
倒计时
动态时间
警告文案
震动/声音（后续）
```

---

## Repository

V1 使用：

```text
DataStore
```

暂时不引入 Room。

保存：

```text
FocusRule

WhiteList

FocusState

SuspendedPackages

EmergencyOverride
```

---

# 13. Android 技术栈

第一版：

```text
Kotlin

Jetpack Compose

MVVM / 简单分层架构

Coroutines / Flow

DataStore

AlarmManager

BroadcastReceiver

PackageManager

DevicePolicyManager

Device Owner
```

明确暂时不使用：

```text
AccessibilityService

UsageStatsManager

Foreground Service

WorkManager

Room
```

除非后续发现系统限制导致某个功能必须引入。

---

# 14. Device Owner

这是整个项目最重要的系统能力。

开发阶段计划通过 ADB 将 Focus App 设置为 Device Owner。

目标：

```text
Focus APK
     │
     ▼
Device Owner
     │
     ▼
DevicePolicyManager
     │
     ▼
控制其他应用状态
```

第一阶段重点验证：

```text
Device Owner 是否成功

↓

能否获取已安装 App

↓

能否 suspend 普通第三方 App

↓

能否恢复

↓

系统 App 有哪些不能 suspend
```

这一步验证成功之后，再开发完整 UI。

---

# 15. V1 不做的功能

为了防止项目失控，明确以下功能暂不开发：

```text
App 使用时长统计

每日屏幕时间

App 分类

学习模式

番茄钟

排行榜

积分

连续打卡

账号系统

云同步

UsageStats

应用使用报告

AI 分析

跨设备同步
```

V1 不做“数字健康平台”。

只做：

> **自动防沉迷。**

---

# 16. 开发阶段规划

## Phase 0：系统能力 POC

第一优先级。

建立最小 Android Demo。

只验证：

```text
Device Owner
        ↓
DevicePolicyManager
        ↓
获取 App
        ↓
Suspend 一个测试 App
        ↓
恢复测试 App
```

验收：

```text
点击按钮

↓

抖音 / B站等测试 App 无法正常使用

↓

点击恢复

↓

App 恢复
```

如果这一阶段存在设备厂商限制，立即调整技术路线。

不要先写 UI。

---

## Phase 1：RuleEngine

实现：

```text
普通时间段

跨天时间段

星期规则

shouldFocusNow()
```

并编写单元测试。

重点测试：

```text
23:00 - 07:00

22:59 → false

23:00 → true

01:00 → true

06:59 → true

07:00 → false
```

---

## Phase 2：FocusManager

实现：

```text
enterFocusMode()

exitFocusMode()

reconcileFocusState()
```

打通：

```text
RuleEngine
+
DeviceController
```

做到：

```text
调用 enterFocusMode()

↓

手机立即进入限制状态
```

---

## Phase 3：白名单

实现：

```text
PackageManager 获取 App

↓

Compose App 列表

↓

选择白名单

↓

DataStore 保存

↓

DeviceController 根据白名单限制
```

---

## Phase 4：Scheduler

实现：

```text
AlarmManager

PRE_WARNING

ENTER_FOCUS

EXIT_FOCUS
```

达到：

```text
不用打开 App

↓

时间到

↓

自动进入防沉迷
```

这是 V1 第二个关键里程碑。

---

## Phase 5：系统恢复

实现：

```text
BOOT_COMPLETED

TIME_CHANGED

TIMEZONE_CHANGED
```

统一调用：

```text
reconcileFocusState()
```

验证：

```text
重启手机

↓

当前正处于防沉迷时间

↓

系统启动后自动重新进入限制
```

---

## Phase 6：FullScreenIntervention

先实现：

```text
InterventionActivity
```

支持：

```text
PRE_WARNING

ACTIVE_WARNING
```

然后实现：

```text
倒计时

当前时间

距离起床剩余时间

动态警告文案
```

---

## Phase 7：紧急解除

实现：

```text
长按

↓

等待

↓

确认

↓

EmergencyOverride
```

本次解除后：

```text
当天 / 本次 Focus Session 不再重新进入
```

下一次规则周期自动恢复。

---

## Phase 8：完整 UI

最后再完善：

```text
首页

时间设置

星期设置

白名单

提前提醒

当前状态

Device Owner 状态

紧急解除
```

---

# 17. 推荐开发顺序

严格按照：

```text
① Device Owner POC

        ↓

② Suspend / Resume App

        ↓

③ RuleEngine

        ↓

④ FocusManager

        ↓

⑤ 白名单

        ↓

⑥ AlarmManager

        ↓

⑦ Boot / Time 恢复

        ↓

⑧ FullScreenIntervention

        ↓

⑨ EmergencyExit

        ↓

⑩ UI 完善
```

不要反过来先做漂亮 UI。

这个项目最大的未知因素不是 Compose，而是：

> **目标 Android 设备上的 Device Owner + DevicePolicyManager 到底能够做到什么程度。**

所以必须优先把系统能力验证掉。

---

# 18. V1 验收标准

最终必须通过以下测试：

### 自动进入

```text
设置：
23:00 - 07:00

23:00

↓

无需用户操作

↓

自动进入防沉迷
```

### 自动退出

```text
07:00

↓

自动恢复 App
```

### 白名单

```text
防沉迷期间：

电话     ✓
时钟     ✓
计算器   ✓

抖音     ✗
B站      ✗
游戏     ✗
```

### 进程死亡

```text
杀死 Focus App

↓

23:00

↓

仍然能够触发
```

### 重启

```text
01:00 重启手机

↓

系统启动

↓

检测当前处于 23:00 - 07:00

↓

重新进入防沉迷
```

### 修改时间

```text
修改系统时间

↓

重新计算 FocusState
```

### 全屏预告

```text
22:50

↓

自动出现全屏提醒
```

### 强警告

```text
防沉迷期间继续尝试操作

↓

出现 FullScreenIntervention
```

### 紧急解除

```text
不能一键退出

↓

完成解除流程

↓

本次 Focus Session 结束
```

---

# 19. V1 最终架构原则

整个项目始终遵循五条原则。

### 1. 不依赖用户主动开启

```text
系统自动触发
```

### 2. 不依赖 App 常驻

```text
事件驱动
```

### 3. 默认限制，白名单放行

```text
Default Deny
```

### 4. 当前状态由规则决定

```text
Rule → State
```

而不是：

```text
上一次 App 做了什么 → State
```

### 5. 增加绕过成本，而不是追求绝对无法绕过

最终目标不是打造企业 MDM。

而是：

> **让沉迷状态下的自己，没有那么容易推翻清醒状态下制定的规则。**

---

# 20. 第一个开发任务

不要从首页开始。

创建 Android 工程之后，第一个任务：

> **验证 Device Owner + DevicePolicyManager + setPackagesSuspended()。**

第一阶段只需要三个按钮：

```text
Device Owner 状态

[ Suspend 测试 App ]

[ Resume 测试 App ]
```

当能够做到：

```text
点击 Suspend
↓
目标 App 无法正常使用

点击 Resume
↓
目标 App 恢复
```

这个项目最关键的技术风险就已经验证了一大半。

然后正式进入：

> **RuleEngine → FocusManager → 自动调度**

的开发。
