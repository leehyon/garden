---
title: NVRAM Manager
---

NvM 负责管理 RAM 与非易失存储之间的数据同步。以 **NV Block** 为管理单位，不直接管理单个变量。

```mermaid
flowchart LR

    ROM["ROM Block<br/>默认值"]

    RAM["RAM Block<br/>运行时数据"]

    NV["NV Block<br/>Flash/EEPROM"]

    NV -->|"Read"| RAM

    RAM -->|"Write"| NV

    ROM -.->|"Restore Default"| RAM
```

## 功能点

- `NvM_ReadAll()` 通常在启动阶段调用，用于初始化配置的 RAM Block。校验失败、Block 无效或首次使用时，NvM 按配置使用 ROM 默认值或初始化回调
- `NvM_WriteBlock()` 返回成功只表示请求已被 NvM 接受，不表示物理存储已经完成。请求进入 NvM 队列后，由周期性调用的 `NvM_MainFunction()` 调度，并经 [[MemIf]] 通过 [[Fee]]/[[Ea]] 链路写入 EEPROM 或 DataFlash。完成状态应通过回调或 `NvM_GetErrorStatus()` 查询
- `NvM_GetErrorStatus()` 查询 Block 的最新结果，例如 `NVM_REQ_PENDING`、`NVM_REQ_OK`、`NVM_REQ_NOT_OK`
- 同一个 Block 存在未完成请求（`PENDING`）时，新的请求可能被拒绝；调用方应检查返回值，并避免重复提交。队列调度可通过 `NvMJobPrioritization` 配置优先级策略

### Block 类型

| 类型            | NV 副本 | 适用场景       |
| --------------- | ---- | ---------------- |
| Native Block    | 1 | 一般数据             |
| Redundant Block | 2 | 需要提高掉电/介质故障容错的数据 |
| Dataset Block   | 多个 | 多套数据、版本或轮换存储     |

Dataset Block 通过 Data Index 选择当前使用的 NV 副本；Redundant Block 在读取时可在一个副本损坏时尝试另一个副本。

### 应用层访问

应用 SWC 通常不直接调用 `NvM_WriteBlock()`。一般是通过 AUTOSAR 配置的 NvM Service Port，经 RTE 调用生成的接口，例如：

```c
Std_ReturnType ret =
    Rte_Call_<Port>_<Operation>(/* 参数由配置决定 */);
```

- 实际名称由生成器决定，常见形式是 `Rte_Call_<Port>_<Operation>()`，也可能生成封装的 macro 或 inline function
- SWC 通过该接口访问服务，RTE 再将调用映射到 NvM
- Block ID、RAM Block 和结果状态通常由配置绑定，不应由应用硬编码

`NvM_WriteBlock()`、`NvM_ReadBlock()` 等标准 NvM API 主要供 BSW 模块或已明确配置为直接使用 NvM 的 CDD 调用。SWC 直接调用它们会绕过 RTE 抽象，降低可移植性，也可能不符合项目的分层和接口约束。

> NvM 只负责 NVRAM Block 的抽象、校验和调度；底层介质访问由 MemIf 及对应的 Fee/Ea 模块完成。

### NvM Asychronous Call with Callback

![[nvm-asychronous-call-with-callback.png]]

### NvM Asychronous Call with Polling

![[nvm-asychronous-call-with-polling.png]]

## Memory Stack

![[memory-stack.png]]