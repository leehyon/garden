---
title: SystemDesk Explained
---

## 通过 Operation 的 Arguments 配置参数方向和数据类型

```c
Std_ReturnType Xxx_ReadData(
    Dcm_OpStatusType OpStatus,
    uint8* Data
);
// 返回值 E_OK：读取成功；DCM_E_PENDING：读取未完成
```

可以先把它拆成三个部分：

| C 接口内容 | AUTOSAR 建模位置 | 需要确认 |
|---|---|---|
| `OpStatus` | Operation 的 Argument，方向 `IN` | 类型引用 `Dcm_OpStatusType` |
| `Data` | Operation 的 Argument，方向 `OUT` | 数据长度、类型及最终 C 参数形式 |
| `Std_ReturnType` | Operation 返回状态及 Possible Errors | 生成的返回类型和允许的返回值 |

对于 Dcm 服务，最好由目标 BSW/Dcm 配置生成或提供 **对应 DID 的 ARXML 接口**，再导入 SystemDesk 引用。这样可以避免应用侧手工创建的接口与 Dcm 侧在类型、数组长度或错误码上不一致。

### `Std_ReturnType` 和 `DCM_E_PENDING` 怎么建？

不要在 Arguments 页面额外创建一个 `OUT Std_ReturnType` 参数，因为截图中的 `Std_ReturnType` 是函数返回值，不是输出参数。

在 C/S 建模中，SystemDesk 提供 **Possible Errors** 机制。

- 打开 **Client Server Interface → Properties → Possible Errors**
- 创建或引用项目要求的 Application Error，并设置其错误码
- 打开 **ReadData Operation → Properties → Possible Error Refs**
- 选择此 Operation 允许使用的错误

![[systemdesk-operation-args.png]]

还要注意：**配置了 Possible Errors，也不自动保证生成的函数返回类型一定叫 `Std_ReturnType`**。返回类型、错误码宏及它们的值，需要检查实际生成的 RTE/Dcm 头文件和供应商集成规则。

```c
#define Rte_Call_Diagnostic_PP_DataService_PcbaSerialNumber_ReadData UDS_PcbaSerialNumber_ReadData

FUNC(Std_ReturnType, Diagnostic_CODE) UDS_PcbaSerialNumber_ReadData(VAR(Dcm_OpStatusType, RTE_Diagnostic_APPL_DATA) OpStatus, P2VAR(uint8, AUTOMATIC, RTE_Diagnostic_APPL_VAR) Data);
```
### P2VAR 由谁保证？

```text
SystemDesk / ARXML
    定义参数类型、方向、类型属性和接口契约
                  ↓
RTE / BSW 生成器
    根据模型和目标配置生成实际 C 声明
                  ↓
项目头文件及编译器抽象
    定义 P2VAR 等宏及内存映射
```

## Port API Options

![[systemdesk-ib-portapi.png]]

**Port API Options 不是“启用 Port”的开关**，而是给 RTE 生成器提供针对某个 Port 的**额外 API 生成选项**。普通的 Dcm `ReadData` 服务实现或 NvM `ReadBlock` 调用，通常不需要因为使用了 Port 就特意添加一条；是否必须配置、未配置时采用什么默认值，仍以项目 RTE 生成器及集成规则为准。

### Enable To Take Address：什么时候勾选？

当应用代码或某个服务的配置**确实需要把该 Port 的 RTE API 当作函数指针使用**时，才需要考虑勾选。

- ```c
  /* 示例：具体名称和函数指针类型以生成的头文件为准 */
callback = &Rte_Call_RP_Os_Service_SomeOperation;
  ```
对于调用者而言，RTE API 可能由生成器实现为 C 函数，也可能是函数式宏等形式；但如果代码要**取函数地址**，就需要能够取地址的真实函数。

## Synchronous Server Call Point

推荐配置：

```text
Runnable：UDS_CameraPartNumber_ReadData
    │
    ├─ Triggered by
    │    └─ Dcm ReadData 对应的 Operation Invoked Event
    │
    └─ Operations
         ├─ RP_NvMBlock_CameraPartNumber.NvM_ReadBlock
         │      Invocation = Synchronous
         │
         └─ RP_NvMBlock_CameraPartNumber.NvM_GetErrorStatus
                Invocation = Synchronous
```

## Runnable 中 Triggered by 还是 Operations

![[systemdesk-runnable-cs.png]]

对于 `ReadData`，通信语义是：
- Dcm：我需要读取 DID 数据     → Client
- Diagnostic_Swc：我实现读取   → Server

根据 SystemDesk Manual 的 C/S 模型：
- Client 使用 R-Port
- Server 使用 P-Port

SystemDesk Manual 明确说明：Operation Invoked Event 因 Client 调用而触发 Runnable；被触发的 Runnable 是提供 Operation 的 Server。

| 页面               | 回答的问题                               | 你这个例子                                          |
| ---------------- | ----------------------------------- | ---------------------------------------------- |
| **Triggered by** | “什么事件会启动这个 Runnable？”               | Dcm 调用 `ReadData` → Operation Invoked Event    |
| **Operations**   | “这个 Runnable 主动调用哪些 C/S Operation？” | Runnable 调用 NvM 的 `ReadBlock`、`GetErrorStatus` |
| **Data Access**  | “这个 Runnable 读写哪些 S/R 数据元素或内部变量？”   | 如果读取 `VINHandler` 发布的 VIN，则在此选择                |
| **Arguments**    | Runnable 特定参数配置                     | 不要把它简单理解为“把所有 Operation 参数再手动创建一遍”             |

### 一个完整的 Runnable 配置示例

```text
Diagnostic_Swc
│
├─ P-Port: P_DataServices_F190
│    └─ Interface: DcmDspDid_ASync
│
├─ R-Port: R_NvM_F190
│    └─ Interface: 项目提供的 NvM Service Interface
│
└─ InternalBehavior
     │
     ├─ Runnable: DID_F190_ReadData
     │    ├─ Triggered by
     │    │    └─ OIT_F190_ReadData
     │    │         └─ P_DataServices_F190.ReadData
     │    │
     │    ├─ Operations
     │    │    ├─ R_NvM_F190.ReadBlock
     │    │    └─ R_NvM_F190.GetErrorStatus
     │    │
     │    └─ Data Access
     │         └─ 仅在需要访问其他 S/R 数据时配置
     │
     └─ Runnable: DID_F190_ConditionCheckRead
          └─ Triggered by
               └─ OIT_F190_ConditionCheckRead
```