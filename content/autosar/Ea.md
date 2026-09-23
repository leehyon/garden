---
title: EEPROM Abstraction
---

Ea 是 EEPROM 抽象层，位于 [[NvM]]/[[MemIf]] 与 Eep 之间，向上提供基于逻辑 Block 的统一访问接口，向下调用 Eep 操作真实 EEPROM 硬件。Ea 适用于真正的 EEPROM。EEPROM 支持按字节写入，且写入前通常不需要显式擦除，因此 Ea 的实现比面向 DataFlash 的 [[Fee]] 更直接。

## 功能点

- Ea 的核心职责包括：Block 地址分配与计算、读写请求管理、Block 有效性管理、写入长度补齐、任务状态维护，以及向 NvM 返回任务结果
- Ea 不理解应用数据的业务含义，也不负责 RAM Block、ROM 默认值、CRC、冗余策略或 Dataset 策略管理；这些主要由 NvM 管理。Ea 只处理最终映射后的 Ea Block
- Ea 的 Block 配置通常由 NvM Block 配置自动生成，包括 `EaBlockNumber`、`EaBlockSize` 和 `EaImmediateData`
- NvM 通过一个 32 位虚拟地址访问 Ea，其中高 16 位表示 `BlockNumber`，低 16 位表示 `BlockOffset`

### 异步任务模型

- `Ea_Read`、`Ea_Write`、`Ea_InvalidateBlock` 和 `Ea_EraseImmediateBlock` 都是异步服务
- 这些 API 返回 `E_OK`，只表示 Ea 已经接受请求，不代表 EEPROM 操作已经完成
- 实际任务由周期调用的 `Ea_MainFunction()` 推进，包括读、写、擦除和失效操作
- `Ea_MainFunctionPeriod` 必须与实际 OS Task 周期匹配。例如配置为 `0.01 s`，就应将 `Ea_MainFunction()` 放入 10 ms 周期任务

### Ea Write

![[ea-write.png]]