---
title: Flash EEPROM Emulation
---

Fee 本质上是在 Flash 上实现的一套基于双虚拟扇区、追加写、有效位提交、上电扫描和垃圾回收的数据管理机制。
## 功能点

- Fee 作用是在 DataFlash 上模拟 EEPROM，为上层提供基于逻辑 Block 的非易失性数据读写能力。
- Fee 向上屏蔽 Flash 的物理限制，向下通过 Fls 完成实际的读、写和擦除操作。
- Fee 不把每个 Block 固定映射到某个 Flash 地址，而是采用类似**追加写、日志式存储**的方式，每次更新都写到新的空闲位置。
- 旧数据不会在每次更新时立即擦除，而是在扇区空间不足时，通过扇区交换统一回收。这种机制减少了 Flash 擦除次数，实现写平衡并延长 DataFlash 使用寿命。

### 异步任务模型

- Fee 的主要读写操作是异步的：
    - `Fee_Read`
    - `Fee_Write`
    - `Fee_InvalidateBlock`
    - `Fee_EraseImmediateBlock`
- 接口返回 `E_OK` 表示请求被接受，而不是任务已完成
- 实际任务由 `Fee_MainFunction()` 周期推进
- 上层可以通过以下方式获得结果：
    - 轮询 `Fee_GetStatus()` 和 `Fee_GetJobResult()`
    - 使用任务完成及失败回调
- Fee 在处理任务时通常不接受另一个任务，否则可能返回 `FEE_E_BUSY`
- `Fee_MainFunctionPeriod` 必须与实际 OS Task 周期匹配

### NvM 配置

- Fee Block 通常不是在 Fee 中手工独立配置
- `FeeBlockNumber`、`FeeBlockSize` 和 `FeeImmediateData` 等信息由 NvM 配置及引用自动生成
- 因此新增 NvM Block 时，核心不是只修改 Fee，而是：
    - 在 NvM 中创建或配置 NvM Block
    - 配置相应 RAM Block、ROM Block、Block 长度和管理策略
    - 将底层存储设备映射到 Fee
    - 由配置工具生成对应 Fee Block 信息

### Fee Write

![[fee-write.png]]