---
title: Memory Abstraction Interface
---

MemIf 位于 [[NvM]] 与 [[Ea]]/[[Fee]] 之间，通过统一 API 加 DeviceIndex 将 NvM 的存储请求路由到不同类型的非易失存储抽象模块，从而隔离 NvM 与具体 EEPROM/Flash 实现。它解决的主要问题不是存储算法，而是：
- 让 NvM 不必分别依赖 Ea API 和 Fee API
- 通过设备索引统一访问 EEPROM 与 Flash EEPROM Emulation
- 将设备差异限制在配置生成的函数接口表中
- 降低 NvM 对底层介质类型、驱动接口和工程变体的耦合

> MemIf 更接近一个“静态多态分发器”，而不是一个真正的存储管理器。它不维护 NV Block，不执行磨损均衡，不计算 CRC，也不负责请求排队。

## 工作机制

### 上下游接口契约

#### 上游：NvM

NvM 负责：
- 根据 `NvMBlockDescriptor` 确定目标 NV Block
- 根据 `NvMNvramDeviceId` 选择目标设备
- 提供 `BlockNumber`、偏移、长度和 RAM Buffer
- 发起读、写、失效、擦除和取消请求
- 在后续调度周期中查询设备状态与 JobResult
- 根据 `MEMIF_JOB_OK`、`MEMIF_JOB_FAILED`、`MEMIF_BLOCK_INVALID` 等结果推进 NvM 自己的 Block 状态机

MemIf 对 NvM 提供的统一接口包括：
- `MemIf_SetMode`
- `MemIf_Read`
- `MemIf_Write`
- `MemIf_Cancel`
- `MemIf_GetStatus`
- `MemIf_GetJobResult`
- `MemIf_InvalidateBlock`
- `MemIf_EraseImmediateBlock`
- 可选的 `MemIf_GetVersionInfo`

#### 下游：Ea/Fee

MemIf 根据 `DeviceIndex`：
1. 校验设备索引是否越界
2. 从配置生成的接口列表中找到对应设备
3. 调用该设备的 Ea/Fee 等价 API
4. 将下层返回值或状态原样返回给 NvM

下层 Ea/Fee 负责：
- Block 地址解释与介质映射
- 请求接收、排队和实际执行
- 维护 `IDLE / BUSY / BUSY_INTERNAL` 状态
- 维护 JobResult
- 最终访问 Eep/Fls 驱动
- 根据自身算法完成冗余、失效标记、擦除或 Flash 仿 EEPROM 管理

### 核心机制与状态机

MemIf API 中同时出现两个关键标识：
- `DeviceIndex`：决定请求发给哪个 Ea/Fee 实例
- `BlockNumber`：由目标 Ea/Fee 解释，用于定位下层存储 Block

### 请求提交与异步作业

`MemIf_Read()`、`MemIf_Write()` 等接口在调用期间完成设备分发并返回请求是否被接受；实际介质操作通常是异步 Job。请求被接受后，应结合 `MemIf_GetStatus()` 和 `MemIf_GetJobResult()` 判断设备忙闲及最终结果。`MEMIF_JOB_PENDING` 表示作业仍在处理，不能当作成功；立即返回 `E_NOT_OK` 则表示本次请求未被下层接受。

```mermaid
sequenceDiagram
    autonumber

    participant NvM
    participant MemIf
    participant EaFee as Ea/Fee
    participant Driver as Eep/Fls
    participant HW as EEPROM/Flash

    rect rgb(230, 242, 255)
        Note over NvM,EaFee: 同步阶段：提交请求并立即返回

        NvM->>MemIf: MemIf_Read(DeviceIndex, BlockNumber,<br/>BlockOffset, DataBufferPtr, Length)
        MemIf->>MemIf: 根据 DeviceIndex 查找设备接口
        MemIf->>EaFee: Ea_Read(...) / Fee_Read(...)

        alt Ea/Fee 接受请求
            EaFee-->>MemIf: E_OK
            MemIf-->>NvM: E_OK<br/>仅表示请求已接受
        else Ea/Fee 拒绝请求
            EaFee-->>MemIf: E_NOT_OK
            MemIf-->>NvM: E_NOT_OK<br/>请求未进入异步处理
        end
    end

    rect rgb(240, 255, 240)
        Note over NvM,HW: 异步阶段：周期任务推进实际读取

        loop 后续 MainFunction 调度周期
            NvM->>MemIf: MemIf_GetStatus(DeviceIndex)
            MemIf->>EaFee: Ea/Fee_GetStatus()
            EaFee-->>MemIf: MEMIF_BUSY / MEMIF_IDLE
            MemIf-->>NvM: 当前设备状态

            EaFee->>EaFee: Ea/Fee_MainFunction()
            EaFee->>Driver: Eep/Fls_Read(...)

            Driver->>Driver: Eep/Fls_MainFunction()
            Driver->>HW: 执行物理介质访问
            HW-->>Driver: 返回读取结果
            Driver-->>EaFee: JobEnd / JobError

            NvM->>MemIf: MemIf_GetJobResult(DeviceIndex)
            MemIf->>EaFee: Ea/Fee_GetJobResult()
            EaFee-->>MemIf: MEMIF_JOB_PENDING / 最终结果
            MemIf-->>NvM: JobResult
        end

        alt 读取成功
            EaFee-->>MemIf: MEMIF_JOB_OK
            MemIf-->>NvM: MEMIF_JOB_OK
            Note over NvM: 此时 DataBuffer 中的数据才可使用
        else 读取失败
            EaFee-->>MemIf: MEMIF_JOB_FAILED
            MemIf-->>NvM: MEMIF_JOB_FAILED
            Note over NvM: NvM 执行重试、恢复或错误处理
        else Block 无效或数据不一致
            EaFee-->>MemIf: MEMIF_BLOCK_INVALID /<br/>MEMIF_BLOCK_INCONSISTENT
            MemIf-->>NvM: MEMIF_BLOCK_INVALID /<br/>MEMIF_BLOCK_INCONSISTENT
            Note over NvM: NvM 使用默认值、冗余副本或上报故障
        end
    end
```

## 配置与排查

### MemIfNumberOfDevices

在当前生成配置中，该参数配置下层 Ea/Fee 设备数量，范围为 `1..2`，属于必配项且不支持 Post-build；这一范围和配置约束应以项目使用的 MemIf 实现及配置工具为准。

联动规则：
- 仅使用 Ea 或仅使用 Fee：配置为 `1`
- 同时使用 Ea 和 Fee：配置为 `2`
- 若所有 `NvMNvramDeviceId` 都是同一个值，设备数应为 `1`
- 若 `NvMNvramDeviceId` 同时包含 `0` 和 `1`，设备数应为 `2`

### NvMNvramDeviceId

它属于 NvM Block 配置，不是 MemIf 参数，但却是影响 MemIf 路由最关键的上游配置。

```text
NvMBlockDescriptor.NvMNvramDeviceId
  → MemIf API 的 DeviceIndex
  → MemIf_Lcfg.c 中的设备接口条目
  → Ea 或 Fee
```

典型错误：
- Block 应位于 Fee，却配置到 Ea 的 DeviceIndex
- MemIf 配置为单设备，但 NvM 中仍存在 DeviceId 1
- 多 ECU 变体中设备表顺序不一致
- 合并 ARXML 后 DeviceId 数值变化，但 MemIf 生成配置没有同步更新

此外，`BlockNumber` 由选中的 Ea/Fee 实例解释，不等同于 NvM 的 Block ID。排查某个 Block 无法访问时，应从 NvM Block Descriptor 的设备索引开始，继续核对 MemIf 设备表、下层 Block 编号及 JobResult。
