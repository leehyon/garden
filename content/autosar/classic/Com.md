---
title: Communication
---

Com 是 AUTOSAR 通信栈中的信号语义层：
- 向上屏蔽报文布局、字节序和发送模式，为 RTE/CDD 提供面向 Signal、SignalGroup 的访问接口
- 向下把信号组包成 I-PDU[^1] ，或把接收到的 I-PDU 解包成信号，并通过 [[PduR]] 与具体总线协议栈解耦

它解决的核心痛点不是“把数据发到总线上”，而是：将应用侧的变量访问，转换为通信矩阵定义的 I-PDU 数据交换，同时管理发送策略、数据新鲜度、超时、无效值和通知。

[^1]: Interaction Layer Protocol Data Unit，也就是通俗意义的报文

![[communication-stack.png]]

> 可以把 Com 模块看成 AUTOSAR 的序列化/反序列化层，类似 `protobuf`、`json` 做的事。

Com 不直接关心 CAN ID、LIN Frame、Ethernet Socket 或具体控制器硬件。Com 看到的是 `PduIdType`、`PduInfoType` 和 I-PDU，具体路由目标由 PduR 配置决定。

![[com-context-view.png]]

## Mechanism

### 上下游接口契约

`Com_SendSignal` 被定义为异步接口。其 `E_OK` 只表示 COM 接受了信号更新请求，并不表示报文已经在总线上发送完成。真正的发送完成由下层异步调用 `Com_TxConfirmation` 后确认。

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

### Com PduR Interaction

接收时 PduR 把 I-PDU 推给 Com；发送时 Com 先向 PduR 提交请求；CAN 路径直接携带数据，图中的 LIN/FlexRay 路径在调度时刻通过 `Com_TriggerTransmit` 再拉取数据；真正的发送结束统一由 `Com_TxConfirmation` 通知 Com。

![[com-pdur-interaction.png]]

- `PduR_ComTransmit`：提交发送 I-PDU
- `Com_RxIndication`：完整普通 I-PDU 接收通知
- `Com_TxConfirmation`：普通 I-PDU 发送完成
- `Com_TriggerTransmit`：下层向 Com 拉取发送数据
