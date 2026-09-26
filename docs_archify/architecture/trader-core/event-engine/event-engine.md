# EventEngine 事件驱动引擎（event-engine）

> 本文是 `trader-core` 域下的叶子子系统文档。域级总览见 `../trader-core.md`。
> 本文只展开事件队列、分发线程与定时器机制，不重复展开上层业务事件处理器（见 oms-engine 叶子）。
>
> 源码基准：vnpy 4.4.0，commit `fa5206fe`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 事件封装 | Event 由 type 字符串 + data 负载组成 | `vnpy/event/engine.py:16` Event |
| 事件入队 | put 把 Event 放入内部 Queue | `event/engine.py:105` put |
| 类型处理器注册 | register/unregister 按事件类型注册回调 | `event/engine.py:111,120` |
| 通用处理器注册 | register_general/unregister_general 广播给所有事件 | `event/engine.py:132,140` |
| 分发循环 | _run 从队列 get 事件并 _process | `event/engine.py:55` _run |
| 事件分发 | _process 先投类型处理器，再投通用处理器 | `event/engine.py:66` _process |
| 定时器事件 | _run_timer 每 interval 秒 put 一个 EVENT_TIMER | `event/engine.py:80` _run_timer |
| 启停 | start/stop 启停分发线程与定时器线程 | `event/engine.py:89,97` |
| 交易事件类型常量 | EVENT_TICK/TRADE/ORDER/POSITION/ACCOUNT/CONTRACT/QUOTE/LOG | `vnpy/trader/event.py:7-14` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Event` | `event/engine.py:16` | 事件载体（type + data） |
| `HandlerType` | `event/engine.py:30` | `Callable[[Event], None]` 类型别名 |
| `EventEngine` | `event/engine.py:33` | 事件分发核心，持有队列/双线程/处理器表 |
| `EVENT_TIMER` | `event/engine.py:13` | 定时器事件类型串 `"eTimer"` |
| `EVENT_TICK` 等 | `trader/event.py:7` | 交易事件类型串（带 `.` 后缀，支持按 vt_symbol 订阅） |

## 3. 关键调用链

1. **分发链**：`_run`（`event/engine.py:55`）循环 `self._queue.get(block=True, timeout=1)` → 拿到事件调 `_process`（`event/engine.py:66`）→ 若 `event.type in self._handlers` 用列表推导 `[handler(event) for handler in self._handlers[event.type]]` 依次调用 → 再对 `_general_handlers` 广播。`Empty` 异常被吞掉继续循环。
2. **定时器链**：`_run_timer`（`event/engine.py:80`）`while self._active: sleep(interval); put(Event(EVENT_TIMER))`——定时器事件本身也走同一队列，由分发线程串行处理。
3. **注册链**：`register(type, handler)`（`event/engine.py:111`）→ `handler_list = self._handlers[type]`（defaultdict 自动建表）→ 去重后 append。`unregister` 移除后若表空则 `self._handlers.pop(type)`。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `interval` | 定时器间隔，默认 1 秒 | `event/engine.py:42` |
| 队列 | 无界 `Queue()`（无 maxsize） | `event/engine.py:48` |
| `EVENT_TICK = "eTick."` | 带尾点，网关可追加 vt_symbol 形成按合约订阅 | `trader/event.py:7` |

## 5. 错误与重试语义

- `_run` 中 `get(timeout=1)` 超时抛 `Empty` 被 `pass` 吞掉——这是正常的空转退出条件检查，不是错误。
- **处理器异常不被捕获**：`[handler(event) for handler in ...]`（`event/engine.py:75,78`）中任一 handler 抛异常会中断后续 handler 并向上传播到 `_run` 的 try 块外——实际上会导致分发线程崩溃。框架依赖各业务 handler 自行 try/catch。
- 无重试/退避机制；事件入队后不丢失（Queue 内存队列），但进程崩溃即丢。

## 6. 并发细节

- **双线程模型**：`_thread`（分发线程，`event/engine.py:50`）+ `_timer`（定时器线程，`event/engine.py:51`）。
- **单消费者串行**：所有事件在分发线程单线程执行，业务 handler 之间天然串行，这是 vnpy 避免加锁的核心设计。
- **生产者-消费者**：任意线程（网关回调线程、策略线程）都可 `put`，由 `queue.Queue`（线程安全）缓冲；分发线程单线程消费。
- **关闭顺序**：`stop()`（`event/engine.py:97`）先置 `_active=False`，再 `_timer.join()` 再 `_thread.join()`。分发线程靠 `get(timeout=1)` 每秒醒来检查 active 标志退出，无需毒丸消息。
- **锁**：`_handlers` 与 `_general_handlers` 的注册/注销可能与分发并发发生（注册在主线程，分发在工作线程），但 CPython GIL 下列表 append/remove 原子，框架未加显式锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- Event/EventEngine 事件队列与分发
- 类型/通用处理器注册表
- 定时器事件生成
- trader 事件类型常量定义

**Out-of-Scope（不在本仓库源码内）**
- 具体业务处理器（OmsEngine/LogEngine/策略等，各自注册自己的 handler）
- 进程间事件传递（见 rpc 域的 RPC 通信）
- 持久化事件日志

## 8. 与相邻子系统交互

- 上游：BaseGateway 各 on_xxx 回调 → EventEngine.put（事件生产者）；MainEngine.write_log 也 put
- 下游：EventEngine → OmsEngine.process_*（交易数据入库）、LogEngine.process_log_event（日志输出）、各策略/应用引擎
- 横向：EventEngine 是 trader-core 域的中枢，所有引擎通过它解耦——生产者不直接调用消费者

## 9. 语言专项适配口径（纯 Python）

- **能力缝**：EventEngine 是经典的"发布-订阅"能力缝——`_handlers` defaultdict 是字符串类型→回调列表的注册表，`register` 是注册点，`_process` 是分发点。无字符串路由到类，但有"事件类型串→回调函数列表"的动态路由。
- **双轨处理器**：类型处理器（只听特定 type）+ 通用处理器（听所有 type）是同一机制的双轨——通用处理器用于日志/监控/全局拦截。
- **可选依赖/懒加载**：不涉及；EventEngine 零重依赖（仅标准库 queue/threading）。
- **隐藏状态机在循环里**：`_run` 分发循环本身是一个 `active=True/False` 的两态机（运行/停止），但过于简单（无分支迁移），不单独补 lifecycle 图；`_run_timer` 同理。
- **图型侧重**：architecture 表达组件拓扑；dataflow 表达事件从生产者到处理器的管道（这是事件引擎最自然的语义模型）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| EventEngine 组件架构图 | `event-engine-architecture.html` | architecture | standard |
| 事件分发数据流 | `event-engine-dataflow.html` | dataflow | standard |

- 架构图落 standard 原因：初版组件围绕 ee 上下分布（client 左、queue 右、线程下）触发 `clean-flow/endpoint-side-direction`；修复动作：改为左→右列布局（client→ee→右侧 4 个组件竖排），消除多扇出方向冲突。
- 数据流图一次通过 standard（5 stage 节点、标签简短、viewBox 1180 宽）。
- 未补 sequence/lifecycle/workflow：分发时序本质是数据管道（已用 dataflow 表达）；active 两态机过于简单；无审批流程语义。
- JSON IR 源文件位于 `json/` 目录。
