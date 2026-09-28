---
title: Run-Time Environment
---

RTE 位于应用软件组件 SW-C 与基础软件 BSW 之间，是 VFB 在单 ECU 上的具体实现。它通过生成类型安全的 Port API、通信缓存、事件触发代码和 OS Task 包装代码，使 SW-C 不直接依赖网络拓扑、BSW API 和具体的任务调度实现。

RTE 主要解决四类耦合：
- 接口耦合：SW-C 只看到 `Rte_Read/Write/Call/Switch` 等生成接口
- 调度耦合：Runnable 只描述触发事件，不直接操作 OS Task
- 通信耦合：本地通信、跨核通信、CAN/LIN/FlexRay 通信和以太网通信可保持相似的 SW-C 接口形态
- 数据实现耦合：ApplicationDataType 通过 DataTypeMappingSet 映射到 ImplementationDataType，再落到 C 基础类型

## Mechanism

### 上下游接口

SW-C 对 RTE 的契约：
- **Sender-Receiver**
    - 显式非队列通信：`Rte_Write`、`Rte_Read`
    - 隐式通信：`Rte_IWrite`、`Rte_IRead`
    - 队列通信：`Rte_Send`、`Rte_Receive`
    - 更新检测：`Rte_IsUpdated`
    - Runnable 内部通信：`Rte_IrvWrite`、`Rte_IrvRead`
- **Client-Server**
    - 同步调用：`Rte_Call`
    - 异步调用：`Rte_Call` 发起，`Rte_Result` 获取结果
    - 服务端运行实体通常由 `OperationInvokedEvent` 触发
- **Mode-Switch**
    - 模式请求或通知：`Rte_Switch`
    - 模式读取：`Rte_Mode`
    - 模式变化可通过 `ModeSwitchEvent` 唤醒 Extended Task 中的 Runnable
- **NvData / NvBlock**
    - SW-C 可通过 NvM 标准服务映射调用 `NvM_ReadBlock`、`NvM_WriteBlock` 等
    - 配置 `NVBlockDescriptor` 后，NvM 可通过 `Rte_SetMirror`、`Rte_GetMirror` 和 `Rte_NvMNotifyJobFinished` 与 SW-C 数据交互

### RTE 的本质：生成式胶水层

RTE 的运行实体不能简单理解成一个“模块线程”。其生成结果大致包含：

```text
RTE Generated Code
├── 生命周期
│   ├── Rte_Start / Rte_Stop
│   └── SchM_Init / SchM_Deinit
├── SW-C 接口
│   ├── Rte_<SW-C>.h
│   └── Rte_<SW-C>_Type.h
├── 通信实现
│   ├── RTE Buffer
│   ├── Read / Write / Send / Receive
│   ├── Call / Result
│   └── COM / LdCom / NvM callbacks
├── 调度实现
│   ├── TASK() wrapper
│   ├── Runnable 调用序列
│   └── Alarm / Event 关联
└── 隔离与一致性
    ├── IOC
    ├── Spinlock
    ├── ExclusiveArea
    ├── IRV
    └── OsApplication memory section
```

### 事件到任务的执行模型

Runnable 本身不是 OS Task。RTE Event 先与 Runnable 关联，Runnable 再通过 `MapTask` 映射到 OS Task。

## Sequence

![[rte-sender-receiver-data.png]]

![[rte-sender-receiver-event.png]]

![[rte-client-server-sync.png]]

![[rte-client-server-async.png]]