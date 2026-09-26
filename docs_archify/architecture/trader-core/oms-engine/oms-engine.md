# OmsEngine 委托持仓账户管理（oms-engine）

> 本文是 `trader-core` 域下的叶子子系统文档。域级总览见 `../trader-core.md`。
> 本文只展开 OmsEngine 的内存缓存与事件订阅职责，不重复展开 MainEngine 容器（见 `../main-engine/main-engine.md`）、EventEngine 分发（见 `../event-engine/event-engine.md`）与 OffsetConverter 平仓换算细节（见 `../gateway-converter/gateway-converter.md`）。
>
> 源码基准：vnpy 4.4.0，commit `fa5206fe`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 行情缓存 | ticks 字典按 vt_symbol 存最新 TickData | `engine.py:369,394` process_tick_event |
| 委托缓存 | orders 字典按 vt_orderid 存 OrderData，维护 active_orders 活跃集合 | `engine.py:370,399` process_order_event |
| 成交缓存 | trades 字典按 vt_tradeid 存 TradeData | `engine.py:371,416` process_trade_event |
| 持仓缓存 | positions 字典按 vt_positionid 存 PositionData | `engine.py:372,426` process_position_event |
| 账户缓存 | accounts 字典按 vt_accountid 存 AccountData | `engine.py:373,436` process_account_event |
| 合约缓存 | contracts 字典按 vt_symbol 存 ContractData，首到即建 OffsetConverter | `engine.py:374,441` process_contract_event |
| 报价缓存 | quotes 字典 + active_quotes 活跃集合 | `engine.py:375,450` process_quote_event |
| 单条查询 | get_tick/get_order/get_trade/get_position/get_account/get_contract/get_quote | `engine.py:462-502` |
| 全量查询 | get_all_ticks/orders/trades/positions/accounts/contracts/quotes | `engine.py:504-544` |
| 活跃查询 | get_all_active_orders/quotes | `engine.py:546-556` |
| 平仓换算委托 | update_order_request / convert_order_request 转发到 OffsetConverter | `engine.py:558-581` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `OmsEngine(BaseEngine)` | `engine.py:360` | 委托持仓管理系统引擎，订阅 7 类事件并维护内存表 |
| `register_event` | `engine.py:384` | 向 EventEngine 注册 7 个 process_* 事件处理器 |
| `ACTIVE_STATUSES` | `object.py:14` | `{SUBMITTING, NOTTRADED, PARTTRADED}`，判定委托/报价是否活跃 |
| `process_order_event` | `engine.py:399` | 更新 orders/active_orders 并回写 converter |
| `convert_order_request` | `engine.py:566` | 按 lock/net 模式把原始请求交给 converter 拆单 |

## 3. 关键调用链

1. **事件入库链**：网关 `on_order(order)` → EventEngine put EVENT_ORDER → OmsEngine.process_order_event（`engine.py:399`）→ `self.orders[order.vt_orderid] = order` → `order.is_active()` 判定 active_orders 增删 → 按 `order.gateway_name` 取 OffsetConverter 调 `converter.update_order(order)`（`engine.py:412-414`）。
2. **合约到达建 converter**：`process_contract_event`（`engine.py:441`）→ `self.contracts[contract.vt_symbol] = contract` → 若该 gateway_name 尚无 converter 则 `self.offset_converters[gateway_name] = OffsetConverter(self)`（`engine.py:447-448`）。
3. **下单前换算链**：策略经 MainEngine.convert_order_request（`engine.py:566`）→ 按 gateway_name 取 converter → `converter.convert_order_request(req, lock, net)` 返回拆分后的 OrderRequest 列表（无 converter 时原样返回 `[req]`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| engine_name | 固定 `"oms"` | `engine.py:367` |
| 无外部配置文件 | OmsEngine 行为完全由事件流驱动，不读 SETTINGS | — |
| convert_order_request 的 net 参数 | 默认 False，由调用方按策略配置传 | `engine.py:571` |

## 5. 错误与重试语义

- OmsEngine 是纯内存缓存，**不发起网络请求、不做重试**；所有数据来自网关事件推送。
- 查不到键时 `dict.get(..., None)` 返回 None，由 MainEngine 层 `get_gateway` 写日志（OmsEngine 的 get_* 静默返回 None）。
- OffsetConverter 未初始化时（converter 为 None）：convert_order_request 原样返回 `[req]`（`engine.py:577-578`），降级为不换算。
- 无错误包装/异常抛出——事件处理器内不 try/catch，异常会沿 EventEngine 分发链向上抛（见 event-engine 叶子的 `[handler(event) for handler in ...]` 列表推导）。

## 6. 并发细节

- OmsEngine 的所有 process_* 处理器**运行在 EventEngine 的单一分发线程**中（见 event-engine 叶子），因此七类字典的读写天然串行，**无需加锁**。
- 但策略线程可同时调用 get_all_orders 等查询接口读取字典——Python GIL 下字典读与单线程写的竞态在实践中可接受（框架明确要求网关数据对象不可变，见 gateway 文档）。
- `active_orders` 与 `orders` 在同一处理器内同步增删，保持一致；无后台轮询线程。
- offset_converters 字典在 contract 事件到达时惰性创建，之后由事件处理器串行更新。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 七类数据的内存字典缓存与活跃集合维护
- 事件订阅注册与 process_* 处理器
- 查询接口委托与平仓换算转发

**Out-of-Scope（不在本仓库源码内）**
- 数据持久化落库（见 trader-service/database-interface 叶子）
- 历史行情查询（网关 query_history）
- 实际持仓冻结计算逻辑（见 gateway-converter 叶子的 PositionHolding）
- 交易所柜台返回的真实持仓/账户（外部系统）

## 8. 与相邻子系统交互

- 上游：EventEngine → OmsEngine（按 EVENT_TICK/ORDER/TRADE/POSITION/ACCOUNT/CONTRACT/QUOTE 分发）
- 下游：OmsEngine → OffsetConverter（按 gateway_name 取，回写 order/trade/position）
- 横向：MainEngine 把 OmsEngine 的 16 个 get_* 方法绑定为自身属性（`engine.py:145-163`），供策略调用
- 数据源：BaseGateway 子类经 on_xxx 回调推送数据（见 gateway-converter 叶子）

## 9. 语言专项适配口径（纯 Python）

- **能力缝**：OmsEngine 是典型的"事件订阅 + 内存表"能力缝——7 个 process_* 方法是 EventEngine 的回调钩子，字典是状态存储。无注册表路由，但有"按 gateway_name 维护 OffsetConverter 实例字典"的轻量注册表。
- **惰性创建**：OffsetConverter 在首条 contract 事件到达时才创建（`engine.py:447`），PositionHolding 在首条持仓/成交/委托事件到达时经 `get_position_holding` 惰性建（converter.py:355）——典型的懒加载工程。
- **数据不可变契约**：BaseGateway 文档要求 on_xxx 传入的数据对象不可变（传前 copy.copy），因此 OmsEngine 直接存引用即可安全跨线程读。
- **图型侧重**：architecture 表达七表拓扑；dataflow 表达事件从网关到内存表的管道。本叶子无状态机（活跃集合增删是事件驱动的派生状态，非独立实体状态机），故不补 lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| OmsEngine 数据存储架构图 | `oms-engine-architecture.html` | architecture | standard |
| 事件入库数据流 | `oms-engine-dataflow.html` | dataflow | standard |

- 架构图落 standard 原因：初版 9 组件扇出布局触发 `clean-flow/endpoint-side-direction` 与 `edge-through-node`（oms→conv 穿过 trade 节点）；修复动作：合并 7 表为 3 个分组盒（行情与合约/委托成交报价/持仓与账户）、改为左→右列布局、把 oms→conv 改为 trade→conv 避免穿线，校验通过后落 standard。
- 数据流图落 standard 原因：首版标签"Tick/Order/Trade/Position/Account/Contract/Quote"过长压节点；修复动作：缩短为"事件入队"、加 viewBox [1180,400]，落 standard。
- 未补 sequence/lifecycle：事件分发时序见 event-engine 叶子，委托状态机见 objects-constants 叶子，重复信息省略。
- JSON IR 源文件位于 `json/` 目录。
