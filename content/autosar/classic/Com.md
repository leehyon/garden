---
title: Communication
---

Com 是 AUTOSAR 通信栈中的信号语义层：
- 向上屏蔽报文布局、字节序和发送模式，为 RTE/CDD 提供面向 Signal、SignalGroup 的访问接口
- 向下把信号组包成 I-PDU[^1] ，或把接收到的 I-PDU 解包成信号，并通过 [[PduR]] 与具体总线协议栈解耦

它解决的核心痛点不是“把数据发到总线上”，而是：将应用侧的变量访问，转换为通信矩阵定义的 I-PDU 数据交换，同时管理发送策略、数据新鲜度、超时、无效值和通知。

[^1]: Interaction Layer Protocol Data Unit，也就是通俗意义的报文

![[communication-stack.png]]

Com 可粗略理解为按通信矩阵配置执行的位级打包与解包层；它不是通用序列化框架，不负责 JSON/Protobuf 一类格式的 schema 编码或动态对象解析。

Com 不直接关心 CAN ID、LIN Frame、Ethernet Socket 或具体控制器硬件。Com 看到的是 `PduIdType`、`PduInfoType` 和 I-PDU，具体路由目标由 PduR 配置决定。

![[com-context-view.png]]

## 工作机制

### 上下游接口契约

`Com_SendSignal()` 是同步 API：调用返回时，Com 已处理该信号更新请求；`E_OK` 不表示报文已经在总线上发送完成。I-PDU 何时提交给下层，取决于发送模式、Transfer Property、I-PDU Group 状态及配置的触发条件。真正的发送结果由下层通过 `Com_TxConfirmation()` 异步反馈。

### 核心数据组织

```text
I-PduGroup
└── I-PDU
    ├── Signal
    ├── Signal
    └── SignalGroup
        ├── GroupSignal
        └── GroupSignal
```

I-PDU Group 是通信控制域，用于批量启停一组 I-PDU。一个 I-PDU 可以通过 `ComIPduGroupRef` 引用多个 I-PDU Group。

I-PDU Group 决定通信是否开放，MainFunction 决定延迟处理何时推进。

### Com 与 PduR 的交互

接收时 PduR 把 I-PDU 推给 Com；发送时 Com 先向 PduR 提交请求；CAN 路径直接携带数据，图中的 LIN/FlexRay 路径在调度时刻通过 `Com_TriggerTransmit` 再拉取数据；真正的发送结束统一由 `Com_TxConfirmation` 通知 Com。

![[com-pdur-interaction.png]]

- `PduR_ComTransmit`：提交发送 I-PDU
- `Com_RxIndication`：完整普通 I-PDU 接收通知
- `Com_TxConfirmation`：普通 I-PDU 发送完成
- `Com_TriggerTransmit`：下层向 Com 拉取发送数据

### 周期处理与超时监控

Com 的 MainFunction 推进需要周期处理的发送、接收及监控工作；具体分工由配置的 `Com_MainFunctionTx()`、`Com_MainFunctionRx()` 等入口决定。

- Tx 周期、最小延迟时间和重发/触发属性共同影响 I-PDU 的发送时机。
- Rx 接收后，Com 更新 I-PDU 缓冲区并解码配置的 Signal/SignalGroup；接收超时监控按配置的超时参数和周期任务推进。
- `Com_TxConfirmation()` 表示下层报告了发送完成或失败，不等价于接收 ECU 已确认业务数据。

因此，信号值更新、I-PDU 提交、总线发送完成和对端业务确认是不同阶段，端到端可靠性不能仅依赖 Com 的 Tx Confirmation。
