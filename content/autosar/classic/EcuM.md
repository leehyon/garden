---
title: ECU Manager
---

EcuM 是 ECU 生命周期的底层执行者，负责把 ECU 从复位入口带到 OS 和 BSW 可运行状态，并执行休眠、唤醒、下电和复位等不可逆或强时序约束操作。它解决的核心痛点不是“决定什么时候切换模式”，而是：
- 在 OS 启动前，以确定顺序初始化 MCU 驱动和基础 BSW
- 接收 [[BswM]] 已经做出的模式决策，并将其落实为睡眠、关机或复位动作
- 管理唤醒源的 Pending、Validated、Expired 状态
- 在多核系统中同步各核的 Startup、Sleep 和 Shutdown 阶段
- 通过 Callout 将电源控制、复位、低功耗和唤醒验证等硬件相关动作留给集成代码实现

## Mechanism

### 模块边界

EcuM 负责：
- 分阶段初始化驱动和基础 BSW
- 启动 OS，并在 OS 启动后完成 Startup II
- 保存和执行 Shutdown Target
- 进入 Halt 睡眠模式
- 管理和验证唤醒事件
- 触发 ECU 下电或复位
- 多核生命周期同步

EcuM 不负责：
- 不负责业务级模式仲裁，仲裁通常由 BswM 完成
- 不负责网络关闭条件判断，通常由 [[BswM]] 结合 [[ComM]]、[[Nm]] 等状态判断
- 不直接定义芯片低功耗寄存器操作，这些动作位于 MCU 或 EcuM Callout

### 上下游接口契约

#### 上游模块

- BswM
    - 根据系统模式、网络状态和应用条件选择 Shutdown Target
    - 调用 `EcuM_SelectShutdownCause()`、`EcuM_SelectShutdownTarget()`
    - 调用 `EcuM_GoHalt()` 进入休眠
    - 调用 `EcuM_GoDown()` 进入关机或复位流程
    - 契约重点：必须先选择 Target/Mode，再触发执行动作
- 唤醒源所属模块
    - 例如 CanIf、CanTrcv、LinIf、Icu 或其他硬件驱动
    - 检测到潜在唤醒后，通过 `EcuM_SetWakeupEvent()` 报告 Pending 事件
    - 完成真实性验证后，通过 `EcuM_ValidateWakeupEvent()` 报告验证成功
    - 契约重点：传入的是 32 bit 唤醒源位图，不是普通枚举值。每一位代表一个唤醒源
- OS
    - Reset Handler 或启动代码调用 `EcuM_Init()`
    - DefaultTask 必须调用 `EcuM_StartupTwo()`
    - Shutdown Hook 必须调用 `EcuM_Shutdown()`
    - 周期任务必须调度 `EcuM_MainFunction()`

#### 下游模块

- MCU
    - 提供正常运行模式、睡眠模式和复位能力
    - `EcuMSleepModeMcuModeRef` 和 `EcuMNormalMcuModeRef` 必须指向有效的 `McuModeSettingConf`
    - EcuM 负责选择模式，MCU 负责真正修改芯片状态
- OS / SchM / RTE
    - EcuM 调用 `StartOS()` 启动 OS
    - `EcuM_StartupTwo()` 完成 BswM 初始化、SchM 初始化和 `Rte_Start()`
    - 下电流程中停止 RTE、SchM，并调用 `ShutdownOS()`
    - OS Shutdown Hook 再将控制权交回 `EcuM_Shutdown()`
- ComM
    - 若唤醒源配置了 `EcuMComMChannelRef`，有效网络唤醒最终需要通知对应 ComM Channel
    - 该引用为空通常表示该唤醒源不是网络唤醒源
- Det
    - 当 `EcuMDevErrorDetect = true` 时，EcuM 对未初始化调用、空指针、非法参数和未知唤醒源等开发错误进行上报
- 硬件集成层
    - 通过 `EcuM_AL_SwitchOff()`、`EcuM_AL_Reset()`、`EcuM_EnableWakeupSources()` 等 Callout 实现
    - EcuM 只规定函数名、调用时机和参数语义，硬件动作由系统集成者实现

### 任务模型

EcuM 的执行上下文可分为四类。

#### OS 启动前同步上下文

入口为 `EcuM_Init()`，主要完成：
- EcuM 内部变量初始化
- 可选循环复位检测
- 可选可编程中断配置
- 按顺序调用 `EcuM_AL_DriverInitZero()`
- 按顺序调用 `EcuM_AL_DriverInitOne()`
- 调用 `StartOS()`

这一阶段通常不具备完整 OS 调度能力，因此初始化函数不能隐式依赖尚未启动的周期任务、事件、资源或异步处理。

#### OS DefaultTask 上下文

OS 启动后，由用户在 DefaultTask 中调用 `EcuM_StartupTwo()`。该阶段完成 BswM、SchM 和 RTE 相关启动，并把运行期模式控制权交给 BswM。

#### 周期任务上下文

`EcuM_MainFunction()` 由 OS 周期调度，核心职责是处理唤醒源验证：
- 检查 Pending Wakeup
- 启动唤醒源验证
- 调用集成层验证逻辑
- 更新验证计时
- 将事件归类为 Validated 或 Expired
- 停止失败唤醒源的验证

因此它不是 ECU 主状态机的唯一驱动函数，而主要是唤醒事件验证引擎。

#### Shutdown Hook 上下文

`EcuM_GoDown()` 调用 `ShutdownOS()` 后，OS 通过 Shutdown Hook 调用 `EcuM_Shutdown()`。此时：
- 正常 OS 服务已经不可假定可用
- EcuM 根据保存的 Shutdown Target 调用 `EcuM_AL_SwitchOff()` 或 `EcuM_AL_Reset()`
- Callout 原则上不应返回，或者返回后必须进入明确的故障保护路径

### 状态机

![[ecum-fixed-state-machine.png]]

![[ecum-flexible-phases.png]]

![[ecum-wakeup-handling.png]]

## Sequence

### Power Up Sequence

![[ecum-power-up-sequence.png]]

### Run Sequence

![[ecum-run-sequence.png]]

### Shutdown Sequence

![[ecum-shutdown-sequence.png]]

### Sleep Sequence

![[ecum-sleep-sequence.png]]

### Wakeup Sequence

![[ecum-wakeup-sequence.png]]

### CAN Wakeup

![[ecum-can-wakeup.png]]
