# 网关抽象与平仓换算（gateway-converter）

> 本文是 `trader-core` 域下的叶子子系统文档。域级总览见 `../trader-core.md`。
> 本文只展开 BaseGateway 抽象接口与 OffsetConverter 平仓换算逻辑，不重复展开 MainEngine 对网关的转发（见 `../main-engine/main-engine.md`）与 OmsEngine 对 converter 的持有（见 `../oms-engine/oms-engine.md`）。
>
> 源码基准：vnpy 4.4.0，commit `fa5206fe`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 网关抽象基类 | BaseGateway(ABC) 定义 connect/subscribe/send_order/cancel_order 等接口 | `vnpy/trader/gateway.py:33` |
| 事件回调推送 | on_tick/on_order/on_trade/on_position/on_account/on_quote/on_contract/on_log | `gateway.py:93-151` |
| 双发事件 | on_xxx 同时发基础事件类型 + 带 vt_symbol 的定向事件 | `gateway.py:98-99` 等 |
| 写日志 | write_log 构造 LogData 并 on_log | `gateway.py:153` |
| 报价接口 | send_quote/cancel_quote 有默认空实现，网关可选覆盖 | `gateway.py:223,240` |
| 默认配置 | default_setting 类属性声明连接所需字段 | `gateway.py:76` |
| 平仓换算器 | OffsetConverter 按 gateway_name 维护持仓换算 | `vnpy/trader/converter.py:310` |
| 持仓跟踪 | PositionHolding 维护多空/今昨仓/冻结量 | `converter.py:17` |
| 三种拆单模式 | convert_order_request_shfe/lock/net | `converter.py:168,202,242` |
| 换算开关 | is_convert_required 按 net_position 标记决定是否换算 | `converter.py:390` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `BaseGateway(ABC)` | `gateway.py:33` | 交易网关抽象，子类实现连接与交易接口 |
| `on_event` | `gateway.py:86` | 通用事件推送入口，封装 Event 并 put |
| `PositionHolding` | `converter.py:17` | 单合约多空今昨仓 + 冻结量跟踪 |
| `OffsetConverter` | `converter.py:310` | 按网关隔离的持仓换算器，持有 holdings 字典 |
| `convert_order_request` | `converter.py:367` | 按 lock/net/交易所分派三种拆单逻辑 |

## 3. 关键调用链

1. **网关行情推送链**：网关收到行情 → `on_tick(tick)`（`gateway.py:93`）→ `on_event(EVENT_TICK, tick)` + `on_event(EVENT_TICK + tick.vt_symbol, tick)` 双发 → EventEngine 入队。
2. **平仓换算链**：策略 `send_order` 前经 OmsEngine.convert_order_request（`engine.py:566`）→ OffsetConverter.convert_order_request（`converter.py:367`）→ `is_convert_required` 判定 → 取 PositionHolding → 按 `lock`/`net`/`req.exchange in {SHFE,INE}` 分派 shfe/lock/net 三种拆单（`converter.py:381-388`）→ 返回拆分后的 OrderRequest 列表。
3. **持仓更新链**：OmsEngine.process_trade_event → `converter.update_trade(trade)`（`converter.py:328`）→ PositionHolding.update_trade（`converter.py:71`）按 Direction×Offset 增减今昨仓 → `sum_pos_frozen` 封顶冻结量。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `default_name` | 网关缺省名，子类覆盖 | `gateway.py:73` |
| `default_setting` | 连接参数字段模板，子类覆盖 | `gateway.py:76` |
| `exchanges` | 网关支持的交易所列表，子类覆盖 | `gateway.py:79` |
| `ContractData.net_position` | True 时不做平仓换算 | `converter.py:399` |
| `lock` / `net` 参数 | 锁仓模式 / 净仓模式，由策略传入 | `converter.py:367` |

## 5. 错误与重试语义

- BaseGateway 文档要求：所有方法非阻塞、线程安全、断线自动重连——**重连责任在网关实现侧**，本抽象层不提供。
- send_order 失败时网关应把 OrderData.status 设为 REJECTED 并 on_order（`gateway.py:206`），框架不抛异常。
- OffsetConverter 无持仓对象时（`get_position_holding` 返回 None，`converter.py:379`）原样返回 `[req]` 降级不换算。
- 无可平仓位时 `convert_order_request_shfe` 返回空列表 `[]`（`converter.py:181`），策略收到空列表即知无法平仓。

## 6. 并发细节

- BaseGateway 文档明确要求线程安全、无可变共享状态；on_xxx 传入的数据对象不可变（传前 copy.copy）。
- OffsetConverter/PositionHolding 由 OmsEngine 的事件分发线程串行更新（见 event-engine 叶子），无锁。
- 网关实现通常自有行情线程与交易线程（各网关 SDK 内部），本抽象层不约束。
- 无超时/取消传播——撤单是独立的 cancel_order 请求。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- BaseGateway 抽象基类与 on_xxx 回调
- OffsetConverter/PositionHolding 平仓换算与拆单

**Out-of-Scope（不在本仓库源码内）**
- 具体网关实现（CTP/IB/TTS/TT 等，独立仓库包）
- 交易所柜台、行情源服务器
- 今昨仓规则的交易所差异已编码在 converter 内（SHFE/INE 特殊处理），但交易所本身不在本仓库

## 8. 与相邻子系统交互

- 上游：策略/上层 → MainEngine → BaseGateway.send_order（交易请求下行）
- 下游：BaseGateway.on_xxx → EventEngine（行情/委托/成交事件上行）
- 横向：OmsEngine 持有 OffsetConverter 实例字典（按 gateway_name），在 contract 事件到达时惰性创建；策略下单前经 OmsEngine.convert_order_request 调用 converter 拆单
- 数据依赖：OffsetConverter 经 `oms_engine.get_contract` 回调查合约（`converter.py:317`）

## 9. 语言专项适配口径（纯 Python）

- **能力缝**：BaseGateway 是典型的"抽象基类定义契约、第三方包注册实现"能力缝——本仓库只有 ABC，数十个网关包（ctp/ib/tts 等）继承 BaseGateway 即完成接入，无需修改框架代码。
- **双轨实现/配置驱动**：平仓换算是典型的"配置驱动静态分派"——同一 convert_order_request 入口，按 `lock`/`net`/`exchange` 三个维度分派到 shfe/lock/net 三套算法（`converter.py:381-388`），是策略模式的内联实现。
- **惰性创建**：OffsetConverter 在首条 contract 事件时建（engine.py:447），PositionHolding 在首条相关数据时经 `get_position_holding` 建（converter.py:355）——懒加载。
- **特殊交易所硬编码**：`{Exchange.SHFE, Exchange.INE}` 集合在三处硬编码（converter.py:81,97,211,254,385）——这是业务规则而非配置，体现了期货平今平昨规则的特殊性。
- **图型侧重**：architecture 表达抽象与换算组件拓扑；sequence 表达拆单调用链。本叶子无状态机（持仓数值是连续更新的派生量，非离散状态），故不补 lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 网关抽象与平仓换算架构图 | `gateway-converter-architecture.html` | architecture | standard |
| 平仓换算拆单调用时序 | `gateway-converter-sequence.html` | sequence | standard |

- 架构图一次通过 standard（左→右列布局，边界分组清晰）。
- 时序图落 standard 原因：初版参与者标签 "OffsetConverter"/"PositionHolding" 超 86px 框宽、末消息 y=520 超可读区；修复动作：缩短为"换算器"/"持仓对象"、y 间距压到 55px、末消息 y=490。
- 未补 dataflow/lifecycle：事件数据流见 event-engine 叶子，委托状态机见 objects-constants 叶子，重复信息省略。
- JSON IR 源文件位于 `json/` 目录。
