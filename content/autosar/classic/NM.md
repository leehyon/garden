---
title: Network Management Interface
---

Nm 是 [[ComM]] 与各类 BusNm 之间的统一网络管理接口层；在配置启用时，也负责多通道网络休眠协调。它解决两个核心问题：

- 屏蔽总线差异：ComM 不需要分别理解 CanNm、LinNm、UdpNm 或 GenericNm 的接口
- 协调多网络休眠：网关 ECU 同时挂接多条总线时，Nm 确保各通道满足条件后，以协调方式进入 Bus-Sleep Mode

这里的 Nm 接口层不等于总线上的 NM 协议。NM PDU 的发送周期、超时检测、Repeat Message、Ready Sleep、Prepare Bus Sleep 等总线相关行为仍由 CanNm、LinNm、UdpNm 等 BusNm 实现。Nm 接口层主要负责：

- 根据 Channel 找到对应 BusNm；
- 转发 ComM 请求；
- 将 BusNm 状态回调统一转发给 ComM；
- 在启用 Coordinator 时维护跨通道协调状态和定时器。

![[nm-stack.png]]

## 工作机制

### 上下游接口契约

ComM 使用统一的 `NetworkHandleType` 请求网络启动、保持或释放，Nm 再根据通道配置将请求分派给对应 BusNm：

- `NetworkHandle` 是 ComM/Nm 语义下的逻辑通道句柄，必须能够映射到一个 `NmChannelConfig`
- API 的可重入性、并发粒度和跨通道调用限制应以模块配置及具体实现为准，不应仅凭通道不同就假定可并行调用
- 请求 API 返回 `E_OK` 只表示请求已被接受，不代表网络已经完成唤醒或休眠
- 最终状态通过 `Nm_NetworkMode`、`Nm_PrepareBusSleepMode`、`Nm_BusSleepMode` 等回调反馈

Nm 不直接控制 CAN、LIN 或 Ethernet 控制器，而是调用 BusNm 的标准接口。

BusNm 通过 Nm 的统一回调上报状态。

### 请求、释放与网络状态

Nm 的 API 表达网络管理意图，不是控制器启停命令：

- `Nm_NetworkRequest()`：本地需要保持网络通信，转交给对应 BusNm 发起或维持网络请求。
- `Nm_NetworkRelease()`：本地不再需要保持网络；BusNm 仍需依据其他节点的请求、NM 定时器和协议状态决定何时休眠。
- `Nm_PassiveStartUp()`：通知 BusNm 本地检测到网络活动并参与启动，不等价于建立持续的本地网络请求。

因此，`Nm_NetworkRelease()` 返回后网络不一定立即休眠；应等待 BusNm 状态回调和 ComM 的模式反馈。BusNm 的周期函数负责协议状态机和 NM PDU 处理，Nm 统一层不替代这些协议定时逻辑。

```mermaid
sequenceDiagram
    participant ComM
    participant Nm
    participant BusNm
    participant Bus as 总线

    ComM->>Nm: Nm_NetworkRelease(NetworkHandle)
    Nm->>BusNm: BusNm_NetworkRelease(Channel)
    BusNm-->>Nm: E_OK（释放请求已接受）
    Nm-->>ComM: E_OK
    Note over ComM,BusNm: 其他节点或本地请求仍可能维持网络唤醒
    loop BusNm 周期处理与网络协调
        BusNm->>BusNm: 推进协议状态机和休眠计时
        BusNm->>Bus: 按协议发送/接收 NM PDU
    end
    BusNm->>Nm: Nm_PrepareBusSleepMode / Nm_BusSleepMode
    Nm->>ComM: ComM_Nm_PrepareBusSleepMode / ComM_Nm_BusSleepMode
    ComM->>ComM: 按通道状态继续协调关网
```

### 主动网络请求与异步状态反馈

当 ComM 需要通信时，调用 `Nm_NetworkRequest()`。Nm 根据通道的总线类型调用对应 `BusNm_NetworkRequest()`。返回 `E_OK` 后，BusNm 在后续周期中推动自身状态机，进入 Network Mode 后再回调 Nm。

```mermaid
sequenceDiagram
    autonumber
    participant User as ComM User
    participant ComM as ComM
    participant Nm as Nm
    participant BusNm as CanNm / LinNm / UdpNm
    participant BusIf as Bus Interface

    User->>ComM: Request FULL_COMMUNICATION
    ComM->>Nm: Nm_NetworkRequest(NetworkHandle)

    Nm->>Nm: Validate initialization and handle
    Nm->>Nm: Lookup channel and BusNm type
    Nm->>BusNm: BusNm_NetworkRequest(Channel)

    BusNm-->>Nm: E_OK
    Nm-->>ComM: E_OK

    Note over ComM,BusNm: E_OK means request accepted, not Network Mode reached

    loop BusNm periodic processing
        BusNm->>BusNm: Advance BusNm state machine
        BusNm->>BusIf: Transmit NM PDU
    end

    BusNm->>Nm: Nm_NetworkMode(Channel)
    Nm->>ComM: ComM_Nm_NetworkMode(Channel)
    ComM-->>User: Communication mode available
```

### 网络协调器

启用 Coordinator 时，Nm 跨关联通道汇总休眠条件，并依据各 BusNm 的协调配置推进关网定时与状态，减少网关 ECU 不同网络休眠时间不一致的问题。Coordinator 只负责跨网络协调，不替代各 BusNm 的协议状态机；目标时间和协调范围取决于配置。

![[nm-coordination.png]]