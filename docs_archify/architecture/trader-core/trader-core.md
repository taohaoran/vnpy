# trader-core 域总览

> 本域是 vnpy（VeighNa）量化交易框架的**核心交易框架层**，位于 `vnpy/trader/` 与 `vnpy/event/` 包内。
> 源码基准：vnpy 4.4.0，commit `fa5206fe`。主语言：纯 Python。

## 域职责

trader-core 域提供量化交易系统的**运行时骨架**：
1. **事件驱动引擎**（EventEngine）——单线程串行分发所有事件，是全框架解耦的中枢；
2. **主引擎容器**（MainEngine）——注册并管理功能引擎、交易网关、应用插件，转发交易请求；
3. **委托持仓管理**（OmsEngine）——维护七类内存数据缓存，是策略查询行情/委托/持仓/账户的统一入口；
4. **数据结构与枚举契约**——dataclass 定义跨层数据载体，枚举定义合法值域；
5. **网关抽象与平仓换算**——BaseGateway 定义接入契约，OffsetConverter 处理期货今昨仓拆单。

本域**不做**：策略逻辑（在 cta_strategy 等应用）、数据持久化（在 database 接口层）、GUI（在 ui 层）、具体网关协议实现（独立包）。

## 叶子索引

| 叶子 | 职责 | 文档 |
|---|---|---|
| main-engine | MainEngine/BaseEngine/AppLoader 容器与注册 | [main-engine/main-engine.md](main-engine/main-engine.md) |
| oms-engine | OmsEngine 委托/成交/持仓/账户/合约内存管理 | [oms-engine/oms-engine.md](oms-engine/oms-engine.md) |
| event-engine | EventEngine 事件队列/分发/定时器 | [event-engine/event-engine.md](event-engine/event-engine.md) |
| objects-constants | dataclass 数据结构与枚举体系 | [objects-constants/objects-constants.md](objects-constants/objects-constants.md) |
| gateway-converter | BaseGateway 抽象 + OffsetConverter 平仓换算 | [gateway-converter/gateway-converter.md](gateway-converter/gateway-converter.md) |

## 域级机制细节

### 事件驱动是全框架的总线
EventEngine 是本域（乃至全框架）的中枢：所有网关行情、委托回报、定时器事件都进同一个 `Queue`，由**单一分发线程**串行投递给注册的 handler。这是 vnpy 避免跨线程加锁的核心设计——业务 handler 之间天然串行。见 [event-engine](event-engine/event-engine.md)。

### MainEngine 是唯一的入口
策略通过 MainEngine 下单/查询；MainEngine 把 OmsEngine 的 16 个 get_* 方法绑定为自身属性，对外暴露统一查询面。引擎/网关/应用三类注册表都在 MainEngine。见 [main-engine](main-engine/main-engine.md)。

### vt_* 复合主键命名约定
跨层数据对象用 `vt_symbol = symbol.exchange`、`vt_orderid = gateway.orderid` 等字符串拼接做全局唯一 key，OmsEngine 字典全靠此。见 [objects-constants](objects-constants/objects-constants.md)。

### 期货平仓换算
OffsetConverter 按 gateway_name 隔离，跟踪每个合约的今仓/昨仓/冻结量，把策略发出的 Offset.CLOSE 请求拆成平今/平昨/开仓子单，SHFE/INE 交易所特殊处理。见 [gateway-converter](gateway-converter/gateway-converter.md)。

## 域级图表

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| trader-core 域架构总览 | `trader-core-architecture.html` | architecture | standard |
| 端到端事件数据流 | `trader-core-dataflow.html` | dataflow | standard |
| 端到端下单时序 | `trader-core-sequence.html` | sequence | standard |

- 架构图：初版 gw→eventengine 竖线穿过 oms/conv、oms→conv 标签重叠；修复动作：删除 gw→eventengine 连线（在本文字说明）、去掉 oms→conv 标签，落 standard。
- 数据流图一次通过 standard（5 stage 管道）。
- 时序图一次通过 standard（6 参与者、7 消息）。
- JSON IR 源文件位于 `json/` 目录。
