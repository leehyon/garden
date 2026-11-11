---
title: AUTOSAR NvM Persistence
---

本文以一个需要保存后视镜模式（`MirrorMode`）的 Native Block 为例，介绍 AUTOSAR [[NvM]] 的读写流程、配置要点，以及如何根据数据特性选择持久化策略。

## 1. 先确定数据的持久化策略

并非所有数据都适合在每次变化时写入非易失存储。设计 NvM Block 前，应先明确数据的重要性、更新频率、允许丢失的数据窗口和掉电时限，再选择保存时机。

| 策略 | 适用数据 | 典型做法 |
| --- | --- | --- |
| 上电读取、下电保存 | 用户偏好、模式选择、可接受异常掉电丢失的状态 | `NvM_ReadAll()`；修改时标记 RAM Block；下电调用 `NvM_WriteAll()` |
| 稳定后保存、下电兜底 | 座椅位置、空调设定、音量、亮度 | 变化时更新 RAM；一段时间无新变化后调用 `NvM_WriteBlock()`；下电再兜底 |
| 周期 Checkpoint、下电兜底 | 运行时间、里程、动作计数、学习值 | RAM 中累计；达到时间或数值阈值后保存；下电时再保存 |
| 事务完成后提交 | 配对授权、软件更新状态、生产配置、功能激活 | 构造完整数据后调用 `NvM_WriteBlock()`；等待写成功，再确认业务事务完成 |

### 1.1 适合上电读、下电写的数据

这类数据通常变化不频繁，允许异常掉电时丢失本次运行期间的更新，而且正常下电时有足够时间完成 `NvM_WriteAll()`。例如：

- 用户偏好：后视镜模式、主题、语言、单位制；
- 舒适性设置：座椅默认位置、空调偏好、音量、灯光偏好；
- 非关键运行模式：上次显示页面、菜单状态；
- 可重新计算的缓存或派生状态；
- 普通提示标志和非安全相关的慢速学习值。

### 1.2 适合事件触发保存的数据

“立即写”指在业务事件发生后提交异步写请求，并不意味着应用调用时 Flash 同步写完。以下数据通常应在关键状态转换后提交，而不是等到 ECU Shutdown：

- 断电后不能丢失的关键状态，例如防盗配对或授权、软件更新阶段、回滚状态、关键生产配置；
- 更新频率低、每次变化都有业务意义的数据，例如 VIN、生产序列号、ECU Variant、功能激活状态；
- 要求事务一致性的数据。应先在 RAM 中构造完整的新 Block，再提交写请求并等待成功；只有写入成功后，才对外确认事务完成。

对掉电恢复要求高的事务，可以评估 `NVM_BLOCK_REDUNDANT`。冗余管理使用两个 NV 副本，但仍需根据项目的恢复需求和供应商实现进行验证。

### 1.3 不要逐次写入高频数据

里程、运行时间、能耗、传感器累计量、消息计数、故障次数和频繁变化的位置等数据，不宜每变化一次就调用 `NvM_WriteBlock()`。频繁请求可能填满 NvM 队列、增加 [[Fee]] 重组和 Flash 擦写，并挤占其他 NvM Job 的处理时间。

对这些数据采用 RAM 累计、周期 Checkpoint 或稳定后写入。即使某项数据很重要，也应避免不加区分地逐次写入。

## 2. NvM 的基本读写模型

一个 NvM Block 通常由 NV Block、RAM Block、可选 ROM 默认数据和管理状态组成。Native Block 通常对应一个 NV Block 和一个 RAM Block；Redundant Block 使用两个 NV 副本；Dataset Block 则管理多个数据集。

对于上电读、下电写的场景，典型流程如下：

1. 启动阶段调用 `NvM_ReadAll()`，将 NV 数据读入 Permanent RAM Block。
2. 等待 ReadAll 的 MultiBlock Job 完成，并检查结果。
3. 应用运行期间读写 RAM 副本，不在每次修改时直接写 Flash。
4. 数据发生变化时将 RAM Block 标记为 Changed。
5. Shutdown 阶段调用 `NvM_WriteAll()`，等待 MultiBlock Job 完成后再允许断电。

`NvM_ReadAll()` 和 `NvM_WriteAll()` 都是异步请求。API 返回只表示请求已提交，不表示读写已经完成。[[EcuM]]/[[BswM]] 状态管理必须依据项目配置等待相应 Job 完成；依赖持久化数据的应用功能不能在 ReadAll 尚未完成时提前运行。

ReadAll 成功后，RAM Block 通常处于有效且未修改的状态（`VALID / UNCHANGED`）。NV 数据无效时，是否恢复 ROM 默认值以及报告何种请求结果，需结合 Block 配置、调用流程和供应商实现确认。

## 3. 上电读、下电写的推荐配置

以应用通过 Permanent RAM Block 访问数据为前提，常用配置如下：

| 配置 | 建议 | 作用 |
| --- | --- | --- |
| `NvMSelectBlockForReadAll` | `TRUE` | 将 Block 纳入 `NvM_ReadAll()` |
| `NvMSelectBlockForWriteAll` | `TRUE` | 将 Block 纳入 `NvM_WriteAll()` 的处理范围 |
| Permanent RAM Block / PIM | 配置 | 提供应用运行期间访问的 RAM 副本 |
| `NvMBlockUseSetRamBlockStatus` | `TRUE` | 允许应用标记 RAM Block 是否已修改 |
| `NvMBlockWriteProt` | `FALSE` | 不对普通下电写入设置初始写保护 |
| `NvMWriteBlockOnce` | `FALSE` | 允许 Block 在整个生命周期内多次写入 |
| CRC | 按项目完整性要求配置 | 检查 NV 数据完整性 |
| ROM 默认数据或 Init Callback | 按初始化策略选择 | 为首次使用或数据恢复提供初始化值 |

### 3.1 修改 RAM 后标记 Changed

启用 `NvMBlockUseSetRamBlockStatus` 后，应用修改 Permanent RAM Block 时应同步标记其状态：

```c
NVM_MirrorMode_Ram_Data = newMode;

NvM_SetRamBlockStatus(
    NvMConf_NvMBlockDescriptor_MirrorMode,
    TRUE
);
```

`NvM_SetRamBlockStatus(..., TRUE)` 不会立即写 NV；它表示 RAM Block 有效且已修改。之后 `NvM_WriteAll()` 才会根据 Block 状态决定是否写回。因此，漏掉这一步可能导致 `NvM_WriteAll()` 跳过已经被应用修改的数据。

该 API 的适用条件受 NvM 配置约束。应确认 Block 配置了适用的 RAM Block 或同步机制，并启用了 `NvMBlockUseSetRamBlockStatus`。实际项目还应检查 API 返回值，并按项目的错误处理方式处理失败。

### 3.2 不跟踪 Changed 状态

若将 `NvMBlockUseSetRamBlockStatus` 关闭，应用不再显式标记变化。按供应商的实现和 Block 状态管理方式，`NvM_WriteAll()` 可能在每次下电时都写入该 RAM Block，即使内容没有变化。

这种做法应用逻辑简单、不容易因漏标而丢失更新，但可能增加下电时间和 Flash/Fee 擦写。是否采用应结合供应商实现和数据写入频率确认。

### 3.3 两个容易混淆的写入配置

**`NvMWriteBlockOnce` 不是“每次下电只写一次”。** 它约束的是关联 NV Block 在其生命周期内只允许写入一次，通常用于 VIN 或制造数据等特殊场景，不适用于一般运行状态。

**`NvMBlockWriteProt` 表示初始写保护。** 配置为 `TRUE` 时，Block 默认受到写保护；只有在符合配置和 API 条件时才能解除。普通下电保存的 Block 通常配置为 `FALSE`。

## 4. MirrorMode 案例：配置意图与整体评价

本文案例中的 `MirrorMode` 配置为 Native Block，具有 Permanent RAM Block、一个 ROM 默认值和 CRC，并被纳入 ReadAll/WriteAll。目标行为是：

- 上电时从 NV 读取到 `NVM_MirrorMode_Ram_Data`；
- NV 数据无效时，按配置尝试使用 `NVM_MirrorMode_Rom_Data` 恢复；
- 运行期间应用修改 RAM，并标记 Block 为 Changed；
- 下电时通过 `NvM_WriteAll()` 保存到 NV。

整体设计符合“运行期间使用 RAM、下电统一保存”的思路。重点风险不是缺少额外配置，而是应用是否正确标记 Changed，以及 Shutdown 是否等待写入完成。

## 5. MirrorMode 配置项详解

以下取值和判断对应本案例中的工具配置。部分配置名称、默认值和行为属于供应商扩展，最终应以项目使用的 AUTOSAR 版本、NvM 软件包文档和生成代码为准。

### 5.1 Block 标识、管理类型和数据长度

| 配置                            | 案例值                | 含义与检查要点                                                                  |
| ----------------------------- | ------------------ | ------------------------------------------------------------------------ |
| `TreeNodeName`                | `MirrorMode`       | 配置节点名称，可能影响生成的宏名、符号名和配置对象名；不是运行时 NvM 参数。                                 |
| `NvMBlockManagementType`      | `NVM_BLOCK_NATIVE` | Native Block 通常对应一个 NV Block，适合一般配置和状态数据；与本例 `NvMNvBlockNum = 1` 一致。     |
| `NvMNvramBlockIdentifier`     | `10`               | 应用调用 NvM API 使用的逻辑 Block ID，必须在 NvM 配置中唯一。应使用生成的 Block ID 宏，不要在应用中硬编码。   |
| `NvMNvBlockBaseNumber`        | 空或由工具派生            | 用于映射底层 Fee/Ea Block Number。需检查最终生成配置，确认映射唯一且正确。                          |
| `NvMNvBlockNum`               | `1`                | 本例 Native Block 使用一个 NV Block。Redundant 和 Dataset 的数量规则需按对应管理类型及供应商实现确认。 |
| `NvMNvBlockLength`            | `1`                | NV 数据净载荷长度。本例只有在 RAM 对象确为 1 字节时才正确；通常不含 CRC、Fee 管理头或存储对齐填充。              |
| `NvMBlockLengthAutoCalculate` | 开启                 | 工具根据 RAM/ROM 对象类型或符号推导长度；应检查生成结果是否确为 1 字节。                               |
| `NvMBlockHeaderInclude`       | `UserDefine.h`     | 生成配置包含的头文件。确认 RAM/ROM 符号、类型及回调声明在该头文件中正确可见。                              |
| `NvMBlockJobPriority`         | `0`                | 仅在全局启用 Job 优先级时才参与调度；优先级数值的高低含义应查供应商文档。                                  |

`NvMNvBlockBaseNumber` 到 MemIf Block Number 的映射方式与 Dataset Selection Bits 等配置有关。常见表达形式为：

```text
MemIf Block Number
    = (NvMNvBlockBaseNumber << NvMDatasetSelectionBits) + DatasetIndex
```

Native Block 的 Dataset Index 通常为 0，但工具也可能自动分配 Base Number。不要只根据配置界面推断最终地址，应检查生成代码和 Fee/Ea 配置，排除多个 NvM Block 映射到同一底层 Block 的情况。

本例逻辑 Block ID 为 10。ID 0、1 是否保留给 MultiBlock 请求状态或配置 ID Block，需按所用实现确认；业务代码应使用工具生成的 Block ID 宏，例如：

```c
NvMConf_NvMBlockDescriptor_MirrorMode
```

### 5.2 RAM、ROM 数据和同步方式

| 配置 | 案例值 | 含义与判断 |
| --- | --- | --- |
| `NvMRamBlockDataAddress` | `NVM_MirrorMode_Ram_Data` | Permanent RAM Block 地址。应用运行期间访问的 RAM 副本。 |
| `NvMRomBlockDataAddress` | `NVM_MirrorMode_Rom_Data` | ROM 默认数据地址，用于按 NvM 流程恢复初始值。 |
| `NvMRomBlockNum` | `1` | 配置了一个 ROM 默认数据 Block；Native Block 通常允许 0 或 1 个。 |
| `NvMBlockUseSyncMechanism` | 关闭 | 使用隐式同步，NvM 直接访问配置的 RAM Block。 |
| `NvMReadRamBlockFromNvMCallback` | 空 | 显式同步时使用的回调；本例关闭显式同步，不配置合理。 |
| `NvMWriteRamBlockToNvMCallback` | `NULL_PTR` | 显式同步时用于复制应用数据的回调；本例不需要。 |
| `NvMInitBlockCallback` | `NULL_PTR` | 未配置初始化回调。本例选择 ROM 默认数据策略；需确认工具链对两种初始化方式的约束。 |

建议 RAM 与 ROM 对象使用相同、明确的固定长度类型，例如：

```c
uint8 NVM_MirrorMode_Ram_Data;
const uint8 NVM_MirrorMode_Rom_Data = MIRROR_MODE_DEFAULT;
```

不要直接假定普通 C `enum` 一定占 1 字节。其大小可能受编译器和编译选项影响。可以使用固定宽度类型，并通过编译期检查或生成代码检查确保对象大小与 `NvMNvBlockLength` 一致。

隐式同步配置简单，但读写期间应用应避免并发修改 RAM 数据。显式同步可通过 `NvMReadRamBlockFromNvMCallback` 和 `NvMWriteRamBlockToNvMCallback` 在应用数据与 NvM RAM Mirror 之间复制，但会增加配置和 RAM 管理成本。对本例的单字节状态，隐式同步通常更直接。

### 5.3 CRC、状态和数据验证

| 配置 | 案例值 | 含义与判断 |
| --- | --- | --- |
| `NvMBlockCrcType` | `NVM_CRC32` | 使用 CRC32 检测 NV 数据完整性。对于单字节数据，管理开销相对较大；是否改为 CRC8 应服从项目完整性要求。 |
| `NvMBlockUseCrc` | 启用或由工具推导 | 通常由 `NvMBlockCrcType` 决定；工具界面可能显示为只读。 |
| `NvMBlockUseSetRamBlockStatus` | 开启 | 应用通过 `NvM_SetRamBlockStatus(..., TRUE)` 显式标记 RAM Block 为 Changed。 |
| `NvMBlockUseCRCCompMechanism` | 关闭 | 不启用基于 CRC 比较的写前变化检测。本例已显式标记 Changed，通常无需再依赖 CRC 比较判断变化。 |
| `NvMCalcRamBlockCrc` | 开启 | 维护或计算 RAM Block CRC。应结合全局 CRC 配置及工具告警确认其必要性。 |
| `NvMBlockUseAutoValidation` | 关闭 | 是否自动将 Block 标记为有效、以及与验证 API 的关系，可能受供应商扩展影响；按本项目软件包说明确认。 |
| `NvMStaticBlockIDCheck` | 开启 | 检查静态 Block ID，辅助发现映射或配置不匹配；确认 Fee/Ea 映射与生成结果一致。 |

CRC 可用于发现 NV 数据损坏或校验不匹配。若配置了 ROM 默认数据，错误后的恢复行为和最终请求结果（例如 `NVM_REQ_RESTORED_FROM_ROM` 或 `NVM_REQ_INTEGRITY_FAILED`）取决于实际请求流程和供应商实现，不能仅凭 CRC 开关推断。

本例的 `NvMCalcRamBlockCrc` 旁有黄色告警时，应查看工具的 Problem/Validation 详情。可能涉及全局 RAM CRC 或 Background CRC 配置、Block 参数或供应商约束；仅凭告警颜色不能确定原因。对单字节 Block 使用 CRC32 是否值得，也应同时结合项目统一规则与存储开销评估。

### 5.4 读写控制、保护和重试

| 配置 | 案例值 | 含义与检查要点 |
| --- | --- | --- |
| `NvMSelectBlockForReadAll` | 开启 | `NvM_ReadAll()` 会处理该 Block，符合上电加载需求。 |
| `NvMSelectBlockForWriteAll` | 开启 | `NvM_WriteAll()` 会考虑该 Block；实际是否写入还取决于 Block 状态和配置。 |
| `NvMBlockWriteProt` | 关闭 | 没有初始写保护，符合普通下电保存需求。 |
| `NvMWriteBlockOnce` | 关闭 | 允许生命周期内多次写入。 |
| `NvMSingleBlockCallback` | 空 | 不使用单 Block Job 专用回调；若应用需要异步通知，应另行配置和验证。 |
| `NvMBswMBlockStatusInformation` | 开启 | Block 状态变化可通知 BswM；确认 BswM 是否配置了对应规则或动作。 |
| `NvMWriteVerification` | 关闭 | 写入后不执行读回校验；这是速度与额外验证之间的项目决策。 |
| `NvMWriteVerificationDataSize` | 空 | 未启用写回校验时通常不适用。 |
| `NvMMaxNumOfReadRetries` | `3` | 读取失败重试配置。确认供应商对重试次数的定义。 |
| `NvMMaxNumOfWriteRetries` | `3` | 写入失败重试配置。重试会延长写入和 Shutdown 时间。 |

应依据 Fee/Fls 最坏时序评估 `NvM_WriteAll()` 的完成时间，包括写入、重试和 Fee 内部重组。若电源保持时间有限，配置重试次数并不能保证所有请求都能在断电前完成。

### 5.5 软件变更兼容性

`NvMResistantToChangedSw` 在本例中开启，意图是在软件或 NvM 配置变化后尽量保留已有 NV 数据。但它不会自动迁移数据结构。

若旧版本与新版本的长度、字段含义、枚举编码或端序不同，需要显式设计兼容策略，例如数据版本字段、兼容读取与转换、Migration Callback 或新旧 Block 并存。对单字节 `MirrorMode`，保留稳定的枚举编码有助于兼容，但仍须验证实际升级行为。

## 6. 端到端运行示例

### 6.1 运行期间更新

```c
void MirrorMode_Set(uint8 mode)
{
    if (NVM_MirrorMode_Ram_Data != mode)
    {
        NVM_MirrorMode_Ram_Data = mode;
        (void)NvM_SetRamBlockStatus(
            NvMConf_NvMBlockDescriptor_MirrorMode,
            TRUE
        );
    }
}
```

示例展示了“值变化时标记 Dirty”的流程。正式项目中应检查 `NvM_SetRamBlockStatus()` 的返回值，并使用项目约定的错误处理。若 Block 配置为 `NvMBlockUseSetRamBlockStatus = TRUE`，漏标会使后续 WriteAll 可能跳过该数据。

### 6.2 启动与关闭

```text
启动：
  调用 NvM_ReadAll()
       ↓
  等待 MultiBlock Job 完成并检查结果
       ↓
  确认数据已读取或按配置完成默认值恢复
       ↓
  开放依赖 MirrorMode 的应用功能

运行：
  应用读写 NVM_MirrorMode_Ram_Data
       ↓
  值变化时调用 NvM_SetRamBlockStatus(BlockId, TRUE)

关闭：
  EcuM/BswM 状态机触发 NvM_WriteAll()
       ↓
  NvM 处理 Changed Block，并计算所需完整性数据
       ↓
  写入 Fee/Ea
       ↓
  等待 MultiBlock Job 完成并检查结果
       ↓
  再允许真正断电
```

## 7. Block 划分与数据清单

建议按相近的写入时机、更新频率、重要性、安全等级、生命周期、默认恢复策略、升级兼容策略及数据所有权划分 Block。

例如：

| Block 示例 | 数据特性 | 可能的保存策略 |
| --- | --- | --- |
| `MirrorModeNvBlock` | 单字节用户偏好 | 标记 Dirty，下电通过 WriteAll 保存 |
| `MirrorPositionNvBlock` | 位置参数，变化较频繁 | 动作结束并稳定一段时间后保存，下电兜底 |
| `LifetimeCounterNvBlock` | 累计量 | RAM 累计，达到阈值后 Checkpoint |
| `SecurityStateNvBlock` | 关键安全状态 | 事务完成后提交；评估冗余与 CRC 要求 |

不要把每个 `uint8` 都拆成单独 Block。每个 Block 都可能带来 CRC、Header、管理信息、对齐和 Fee 管理开销；应在写入策略一致的前提下适度合并。

项目可为每个持久化数据建立一份 NvM 数据分类表，明确其 Owner、Block 类型、长度、默认值、更新频率、保存时机、允许丢失窗口、CRC、冗余、升级保留和迁移策略。

| 字段 | MirrorMode 示例 |
| --- | --- |
| 数据名称 / Owner | `MirrorMode` / `MirrorMgr` |
| Block 类型 / 长度 | Native / 1 byte |
| 默认值 | `Normal` |
| 更新频率 | 每驾驶循环 0～数次 |
| 正常保存时机 | Shutdown |
| 异常掉电可否丢失 | 可以；最大丢失窗口为当前驾驶循环 |
| 写入策略 | 标记 Dirty + WriteAll |
| CRC / Redundant / Write Verification | 按项目完整性规则；本例需确认 CRC32 是否必要 |
| 软件升级保留 / Migration | 保留；维持枚举编码兼容或设计迁移 |

## 8. Fee 映射与本案例的派生值

案例草稿中的 Fee 参数关系如下。这里的数值建立在工具配置确实采用相应位数和管理开销的前提上，应以生成配置为准：

```text
FeeBlockNumber
    = 10 << 2
    = 40

FeeBlockSize
    = NvM 数据 1 字节 + CRC32 4 字节 + Static Block ID 2 字节
    = 7 字节
```

Fee 对数据的实际编码、管理开销、对齐和最终 Block Number 可能由供应商工具派生，不应仅凭公式认定运行时布局。应检查生成的 NvM、MemIf、Fee/Ea 配置及符号映射。

若 `FeeImmediateData = TRUE`，需进一步确认该标志的供应商定义，以及它是否符合仅在 Shutdown 保存的需求。若该字段为派生项，应从 NvM Job Priority 或供应商提供的 NvM Immediate 属性等上游配置入手，不要直接修改派生结果。

## 9. 项目落地检查清单

对本例，建议按以下顺序验证：

1. **确认启动顺序。** `NvM_ReadAll()` 是异步请求；确认应用只在 MultiBlock Job 完成且结果满足要求后使用恢复数据。
2. **确认 Dirty 标记。** 应用修改 `NVM_MirrorMode_Ram_Data` 后，确实调用了 `NvM_SetRamBlockStatus(BlockId, TRUE)`，并处理 API 失败。
3. **确认下电等待。** `NvM_WriteAll()` 完成之前，不允许电源管理流程切断 NvM/Fee/Fls 所需电源。
4. **确认底层设备映射。** 本例 `NvMNvramDeviceId = 1`，需检查 MemIf、Fee/Ea 配置和生成代码，确认最终调用的是 `Fee_Read/Fee_Write` 还是 `Ea_Read/Ea_Write`。不要根据 Device ID 数字猜测映射。
5. **确认长度与类型。** 检查 `NVM_MirrorMode_Ram_Data` 和 ROM 默认对象的实际大小，确保与 `NvMNvBlockLength = 1` 一致；不要假定 C `enum` 为 1 字节。
6. **确认 Block Number 唯一。** 检查最终 NvM 到 Fee/Ea 的 Block Number 映射，避免不同逻辑 Block 指向同一底层 Block。
7. **检查 CRC 告警。** 阅读 `NvMCalcRamBlockCrc` 相关 Problem/Validation 详情，核实是否缺少全局 CRC 配置或存在供应商约束。
8. **评估完整性与时限。** CRC32 对单字节数据的开销是否合理、写回验证是否需要、重试次数是否满足最坏 Shutdown 时间，应按项目要求决定。
9. **验证恢复和升级。** 覆盖首次启动、NV 数据无效或 CRC 错误、ROM 默认恢复、软件升级后 Block 布局兼容等场景。

## 10. 总结

`MirrorMode` 的总体配置方向符合“上电读一次、运行期使用 RAM、下电统一保存”的目标：

- `NvMSelectBlockForReadAll = TRUE`：纳入启动读取；
- Permanent RAM Block：供应用运行期间访问；
- `NvMBlockUseSetRamBlockStatus = TRUE`：应用修改后显式标记 Changed；
- `NvMSelectBlockForWriteAll = TRUE`：纳入下电写入；
- ROM 默认数据：为配置的恢复流程提供初始值；
- CRC 与 Static Block ID：按完整性要求检查数据。

最关键的工程保障是：启动时等待 ReadAll 完成，修改 RAM 后不漏标 Changed，并在断电前等待 WriteAll 完成。其余 CRC、冗余、重试和 Block 划分决策，应基于数据风险、存储开销、供应商实现及 ECU 电源时序共同确定。
