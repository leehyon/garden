---
title: Diagnostic Event Manager
---

Dem 是 ECU 内部诊断事实的“状态机数据库”：上游 Monitor 只报告测试结果，Dem 负责防抖、状态迁移、故障存储、环境数据采集、老化清除和 [[NvM]] 持久化，并向 [[Dcm]]、FiM、Dlt 等消费者提供统一的诊断视图。

## Mechanism

### 解决什么痛点

SW-C 和 BSW 中存在大量故障监视器，但 Monitor 不应该各自实现防抖、DTC 状态机、故障确认、老化、故障内存管理和掉电保存。Dem 将这些共性能力集中起来，使故障监视器只需报告 `PASSED/FAILED/PREPASSED/PREFAILED`，其余生命周期由 Dem 统一处理。

同时，Dcm 并不保存故障信息。Dcm 负责实现 UDS、OBD、J1939 等诊断服务，真正的 DTC 状态、冻结帧、扩展数据和故障内存均由 Dem 管理，并通过接口提供给 Dcm。

### 内部任务模型

Dem 是典型的“同步入口 + 周期任务 + 异步存储”的混合任务模型。

![[dem-event-processing.png]]

Dem 处理一次 Event 报告的时序图

![[dem-process-event.png]]

上层 Monitor 在正式上报事件状态前，可以选择预存故障现场数据；随后调用 `Dem_SetEventStatus()` 上报测试结果。Dem 内部完成防抖、事件状态更新、冻结帧及计数器处理，并在状态变化时通知 FiM。

### 初始化

`FiM_Init()` 只完成 FiM 自身的基础初始化；在 `Dem_Init()` 完成并通过 `FiM_DemInit()` 通知 FiM 之前，FiM 还不能提供有效的功能许可查询，因此 `FiM_GetFunctionPermission()` 返回 `E_NOT_OK`。等 Dem 恢复诊断状态并完成初始化后，FiM 才读取所有相关 Event 状态、计算 Permission，并开始正常服务。

![[fim-dem-init-sequence.png]]

### UDS Status Byte

|Bit|名称|工程含义|
|---|---|---|
|Bit0|TF|当前最终测试结果是否 Failed|
|Bit1|TFTOC|当前 Operation Cycle 是否出现过 Failed|
|Bit2|PDTC|故障处于 Pending 状态|
|Bit3|CDTC|故障达到跨循环确认条件|
|Bit4|TNCSLC|自上次清除后是否尚未完成测试|
|Bit5|TFSLC|自上次清除后是否曾经失败|
|Bit6|TNCTOC|当前 Operation Cycle 是否尚未完成测试|
|Bit7|WIR|是否请求故障指示灯|

### 防抖机制

#### Counter-based Debouncing

Monitor 报告：
```text
PREFAILED -> Counter 增加
PREPASSED -> Counter 减少
FAILED    -> 直接到 Failed Threshold
PASSED    -> 直接到 Passed Threshold
```

当计数达到失败阈值时，Dem 才把测试结果判定为最终 `FAILED`。`JumpUp` 和 `JumpDown` 的作用不是改变最终阈值，而是在检测方向发生反转时，把计数器快速移动到指定值，从而塑造故障建立和恢复的动态响应。

#### Time-based Debouncing

```text
PREFAILED -> 启动失败方向计时
PREPASSED -> 启动通过方向计时
FAILED    -> 直接判定失败
PASSED    -> 直接判定通过
```

最大误差至少包含：`DemTaskTime` + OS 调度抖动，因此，若 `DemTaskTime = 10 ms`，配置 `5 ms` 的时间型防抖在实现层面没有实际意义。

#### Monitor Internal Debounce

由 Monitor 自己维护 FDC，Dem 通过 `DemCallbackGetFDCFnc` 获取 `-128...127` 范围的 FDC，并根据结果更新事件状态。
