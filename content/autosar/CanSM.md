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
- CanSM 管理的是逻辑网络

## Mechanism

### 上下游接口契约

上游契约
- **EcuM → CanSM**：EcuM 在 ECU 启动阶段调用 `CanSM_Init()`，并通过 `CanSM_StartWakeupSource()`、`CanSM_StopWakeupSource()`驱动唤醒验证。NeuSAR 要求 `CanSM_Init()`只能调用一次，且 Post-Build 配置未使用，因此 `ConfigPtr` 传入 `NULL_PTR`
- **ComM → CanSM**：ComM 通过异步接口 `CanSM_RequestComMode()`提交 `NO`、`SILENT` 或 `FULL` 请求。`E_OK`只表示请求被接收，不表示物理网络已经完成切换。真正完成后，由 CanSM 调用 `ComM_BusSM_ModeIndication()`反馈
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

## 核心结论

- CanSM 的本质是异步通信状态编排器，而不是 CAN 报文处理模块
- [[ComM]] 提供目标，CanSM 编排状态，[[CanIf]] 执行硬件操作，回调确认结果
- `PRE_FULLCOM`/`PRE_NOCOM` 负责将抽象通信模式转换成合法的硬件模式切换序列
- `FULLCOM` 内部的核心风险控制机制是 BusOff 的 L1/L2 快慢恢复
- `CanSM_RequestComMode()` 返回 `E_OK` 不等于开网完成，最终状态必须以 ComM/BswM Indication 为准
- 配置开发的核心不是参数填值，而是保证 ComMChannel、CanSM Network、CanIf Controller、CanTrcv 和 OS 周期之间的映射及时间约束一致
- 排查问题时，应沿“请求记录 → MainFunction 推进 → CanIf 请求 → 底层回调 → 稳定态通知”的闭环逐段确认