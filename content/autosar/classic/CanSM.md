---
title: CAN State Manager
---

CanSM 是 [[ComM]] 的通信模式请求与 CanIf/CanTrcv 的硬件状态控制之间的协调层。它解决的不是 CAN 报文收发问题，而是将“网络应该开、关或只收不发”的抽象意图，可靠地转换为控制器模式、收发器模式和 PDU 模式的一组有序异步操作，同时负责 BusOff 监控与恢复。可以将其理解为：
- ComM 决定“网络要不要通信”
- CanSM 决定“通过什么状态序列实现”
- [[CanIf]] 执行“控制器、收发器和 PDU 模式切换”

CanSM 的复杂性源于它必须同时协调三个并不等价的状态维度：

|维度|典型状态|管理目的|
|---|---|---|
|ComM 通信模式|`NO_COMMUNICATION`、`SILENT_COMMUNICATION`、`FULL_COMMUNICATION`|对上层表达网络能力|
|CAN Controller 模式|`STOPPED`、`STARTED`、`SLEEP`|控制 CAN 控制器硬件状态|
|CanIf PDU 模式|`OFFLINE`、`TX_OFFLINE`、`ONLINE`|控制报文路径是否允许收发|
|CAN Transceiver 模式|`NORMAL`、`STANDBY`、`SLEEP`|控制外部收发器物理工作状态|

关键点是：
- `Controller STARTED` 不等于网络已经进入 `FULL_COMMUNICATION`
- `FULL_COMMUNICATION` 需要控制器、收发器和 PDU 路径均达到预期状态
- `SILENT_COMMUNICATION` 的核心语义是允许接收、禁止发送，对应 PDU 模式通常为 `CANIF_TX_OFFLINE`
- CanSM 按配置的 Network 管理逻辑网络；Network 与 CAN Controller、CanTrcv 的映射必须一致，且一个网络的模式不应被误当作整个 ECU 的唯一通信状态

## 工作机制

### 上下游接口契约

上游契约
- **EcuM → CanSM**：EcuM 在 ECU 启动阶段调用 `CanSM_Init()`，并通过 `CanSM_StartWakeupSource()`、`CanSM_StopWakeupSource()`驱动唤醒验证。NeuSAR 要求 `CanSM_Init()`只能调用一次，且 Post-Build 配置未使用，因此 `ConfigPtr` 传入 `NULL_PTR`
- **ComM → CanSM**：ComM 通过 `CanSM_RequestComMode()`提交通信模式请求。`E_OK`只表示请求被接收，不表示物理网络已经完成切换。真正完成后，由 CanSM 调用 `ComM_BusSM_ModeIndication()`反馈实际模式
- **OS/SchM → CanSM**：周期调用 `CanSM_MainFunction()`推进各网络状态机。模式请求、BusOff 事件和底层模式回调通常先被记录，再由后续 MainFunction 周期消费

下游契约
- **CanSM → CanIf**：调用 `CanIf_SetControllerMode()`、`CanIf_SetTrcvMode()`和 `CanIf_SetPduMode()`，分别控制 Controller、Transceiver 和 PDU 通道
- **CanIf → CanSM**：控制器和收发器模式切换是异步完成的，CanIf 必须通过 `CanSM_ControllerModeIndication()`、`CanSM_TransceiverModeIndication()`等回调返回完成通知。CanSM 不能将下行 API 的 `E_OK`理解为模式已经生效
- **CanIf → CanSM BusOff**：底层通过 `CanSM_ControllerBusOff()`上报 BusOff。回调只记录事件，真正的恢复动作由 FULLCOM 子状态机在后续周期执行

旁路通知与诊断
- **CanSM → BswM**：网络模式和 BusOff 状态变化可通过 `BswM_CanSM_CurrentState()`通知系统模式管理器，用于联动其他 BSW ActionList
- **CanSM → Dem**：配置 `CANSM_E_BUS_OFF` 后，每次 BusOff 可向 Dem 上报 `DEM_EVENT_STATUS_PRE_FAILED`，恢复后由状态机更新相应故障状态
- **CanSM → Det**：当 `CanSMDevErrorDetect`启用时，上报未初始化、非法网络句柄、空指针、非法 Controller/Transceiver ID 等开发期错误

### 核心任务模型

CanSM 不是“调用即完成”的同步控制模块，而是一个典型的 **请求、回调和周期任务协同模型**：
1. `CanSM_RequestComMode()`接收并保存 ComM 请求
2. `CanSM_ControllerBusOff()`等回调记录异步事件或底层状态
3. `CanSM_MainFunction()`周期扫描每个网络
4. 状态机根据请求、回调标志和计时器推进
5. 调用 CanIf 发起下一步硬件模式转换
6. 最终通过 ComM/BswM Indication 发布稳定状态

### 状态机

![[cansm-state-machine.png]]

### 模式切换与确认

模式切换不是一次原子操作，而是由 CanSM 按配置分阶段推进：

1. 接收 ComM 的目标模式，更新对应 Network 的请求状态。
2. 依据目标模式进入 PRE_FULLCOM、PRE_NOCOM 等过渡状态，按配置控制 PDU 路径、Controller 和 Transceiver。
3. 等待 CanIf/CanTrcv 的异步模式指示，并在 `CanSM_MainFunction()` 中处理超时或继续下一步。
4. 到达稳定状态后，通过 `ComM_BusSM_ModeIndication()`发布实际通信模式；配置了 BswM 通知时，也同步发布 Network 状态。

具体调用次序受配置、唤醒路径和硬件能力影响，不能只根据目标模式推断固定的 Controller/Transceiver 操作顺序。排查时应区分“请求已接受”“底层模式已确认”和“稳定态已通知”三个时点。

### BusOff 恢复

BusOff 通常由 CanIf 转发给 CanSM，随后由 FULLCOM 子状态机执行恢复，而不是在中断回调中直接完成整套切换。L1/L2 恢复使用不同的等待时间和重试阶段；实际恢复速度、重试次数及恢复失败后的行为由 CanSM 配置决定。BusOff 期间还要检查 PDU Mode、ComM 实际模式、Dem 事件状态和应用侧超时，避免把“控制器重新启动”误认为“网络已恢复”。

## 核心结论

- CanSM 的本质是异步通信状态编排器，而不是 CAN 报文处理模块
- [[ComM]] 提供目标，CanSM 编排状态，[[CanIf]] 执行硬件操作，回调确认结果
- `PRE_FULLCOM`/`PRE_NOCOM` 负责将抽象通信模式转换成合法的硬件模式切换序列
- `FULLCOM` 内部的核心风险控制机制是 BusOff 的 L1/L2 快慢恢复
- `CanSM_RequestComMode()` 返回 `E_OK` 不等于开网完成，最终状态必须以 ComM/BswM Indication 为准
- 配置开发的核心不是参数填值，而是保证 ComMChannel、CanSM Network、CanIf Controller、CanTrcv 和 OS 周期之间的映射及时间约束一致
- 排查问题时，应沿“请求记录 → MainFunction 推进 → CanIf 请求 → 底层回调 → 稳定态通知”的闭环逐段确认