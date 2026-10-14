---
title: Default Error Tracer
---

Det 解决的是“BSW 检测到软件使用错误或运行异常后，如何用统一四元组快速定位到具体模块、实例、接口和错误类型”的问题。Det 接收各 BSW 模块上报的错误，将错误交给配置的 Hook/Callout，必要时转发给 Dlt，并支持按核保存错误信息，从而避免错误仅表现为复位、死循环或数据破坏，却缺少可定位上下文。

四元组中各字段的职责是：
- `ModuleId`：错误来源模块
- `InstanceId`：模块实例，单实例模块通常为 `0`
- `ApiId`：检测到错误的那个服务接口的 Service ID
- `ErrorId/FaultId`：具体错误类型

主要入口：
- 开发错误：`Det_ReportError()`
- 运行时错误：`Det_ReportRuntimeError()`
- 瞬时故障：`Det_ReportTransientFault()`

## Mechanism

### 模块边界

Det 不负责什么：
- 不负责检测业务错误：错误检测发生在调用 Det 的 BSW 模块内部，例如空指针检查、初始化状态检查、参数范围检查和超时检查
- 不负责故障去抖和成熟化：没有类似 [[Dem]] 的 debounce、event status、aging 或 healing 机制
- 不负责 DTC 管理和诊断协议响应：这些属于 Dem、Dcm 等诊断模块
- 不负责恢复策略编排：Det 只提供 Hook/Callout 扩展点，真正的降级、复位、报警或日志策略由集成方实现
- 不依赖 RTE 服务接口：手册明确说明 Det 没有 RTE 接口处理

EcuM 负责在 ECU 上电或复位期间调用 `Det_Init()`。该调用只能执行一次，而且必须先于其他可能向 Det 上报错误的模块完成初始化。

![[det-initialization.png]]

### 内部任务模型

Det 与 NvM、Fee、Fls 等模块不同，它不是一个典型的“请求入队 + MainFunction 周期推进”的异步任务模块。手册将 `Det_ReportError` 和 `Det_LogError` 标记为异步，但同时又规定 `Det_ReportError` 固定返回 `E_OK`，且没有描述后续状态查询或完成通知机制。工程上更合理的理解是：调用者只负责提交错误，不通过返回值获知 Hook、日志或 Dlt 后处理的最终结果。该理解属于基于接口行为的架构推断，具体实现仍应以 `Det.c` 源码为准。

![[det-report-error-configured.png]]

## Summary

Det 是一个面向 BSW 的统一错误汇聚与分发模块。其架构重点不在复杂状态机或后台任务，而在于初始化顺序、错误四元组编码、每核错误上下文、Hook/Callout 执行边界以及与 Dlt 的联动。工程实现中最容易出错的地方，是混淆 Det 与 Dem 的责任、误解 `E_OK` 的含义、把 Callout 写得过重，以及没有处理好多核与启动早期的错误上报。

