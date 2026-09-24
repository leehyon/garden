---
title: AUTOSAR OS
---

AUTOSAR OS 是运行在 MCU 硬件与 RTE/BSW/Application 之间的实时调度内核，核心解决以下问题：
- 将任务、ISR、Alarm、Event、ScheduleTable 统一映射到确定性的实时调度模型
- 在多核场景下提供跨核服务、IOC 通信和 Spinlock 同步
- 通过 MPU 内存保护和时间保护，限制非受信应用的空间访问与 CPU 时间占用
- 接管任务上下文、NVIC、SysTick、STM、MPU 等底层资源，向上屏蔽芯片级调度差异

OS 本质上不是一个周期轮询型 BSW 模块，也没有类似 `Os_MainFunction()` 的任务入口。它由硬件定时器中断、任务激活、事件设置、跨核中断及系统调用共同驱动。

## Mechanism

### 上下游接口契约

上游契约：
- **EcuM/Startup**：负责在正确阶段调用 `Os_Init()`、启动从核并调用 `StartOS(AppMode)`
- **RTE**：通常生成周期任务及其 Alarm 配置，将 Runnable 映射到 OS Task。示例中的 5 ms、10 ms 和 20 ms 任务均通过 Alarm 或 Event 驱动
- **BSW/Application**：通过 `ActivateTask`、`SetEvent`、`WaitEvent`、资源管理、多核服务或 IOC 接口使用 OS 能力，同时必须遵守调用上下文、对象访问权限和核归属
- **非受信 Application**：只能访问自身 MPU 区域及显式授权区域；需要访问受信资源时，应通过 Trusted Function 进入受控的特权执行路径

下游契约：
- **SysTick**：通常作为 PIT 类型系统 Counter，按固定周期产生系统 Tick。在 S32K3x 上其中断号配置为 `-1`，在 S32G3x 上配置为 `65535`
- **STM**：作为 HRT Counter，用于 ScheduleTable、时间保护或高精度调度；一旦被 OS 占用，应用不可再直接操作对应 STM 资源
- **NVIC**：OS 管理中断使能、优先级和 Pending 状态等寄存器，应用不得直接修改 OS 接管的 NVIC 配置
- **MPU**：OS 在任务或 Application 切换时装载对应的 MPU Region 配置；S32K3x/S32G3x 的 MPU 仅对当前核有效，因此每个核都需要独立配置
- **Linker/MemMap**：配置工具定义逻辑保护区域，链接脚本负责把代码、变量、栈真正放入对应物理地址。两者必须完全一致，最终应通过 `.map` 文件验证

### 运行模型

OS 的运行核心可以归纳为四层：
- **时间源层**：SysTick/STM 产生 Tick 或比较中断
- **触发层**：Counter 推进 Alarm 和 ScheduleTable，到期后激活 Task 或设置 Event
- **调度层**：根据任务状态、优先级和抢占属性选择当前核上的最高优先级 Ready Task
- **保护层**：任务运行前切换上下文和 MPU 配置，运行期间由时间保护监控执行预算、中断锁预算和最小激活间隔

调度决策模型：
- `OsTaskPriority` 越大，任务优先级越高；同一核上的任务优先级不能重复
- 初始化任务必须具有最高优先级，配置为不可抢占 `NON`
- Idle Task 优先级最低，通常为 0，并配置为可抢占 `FULL`
- Basic Task 通常执行结束后调用 `TerminateTask()`
- Extended Task 可以通过 `WaitEvent()` 进入 Waiting 状态，Event 到达后重新进入 Ready 状态。
- `OsTaskActivation` 限制任务尚未执行完成时允许积压的激活次数。若周期小于最坏执行时间，可能出现激活溢出

![[os-system-call.png]]

## Configuration

### OsSecondsPerTick

这是时间系统中最容易引发全局错误的配置之一：
- PIT 系统 Counter 通常配置为 `0.001 s`
- HRT Counter 必须根据 STM 实际频率计算
- 例如 STM 为 160 MHz，则：`OsSecondsPerTick = 1 / 160000000 = 0.00000000625 s (6.25 ns)

它直接影响：
- Alarm 周期的 Tick 到实际时间换算
- ScheduleTable Duration 和 ExpiryPoint Offset
- 时间保护预算
- 显式同步精度与调整幅度

若 STM 时钟树发生变化但该参数未同步更新，OS 配置在逻辑上仍可运行，但所有绝对时间都会发生比例偏差。
