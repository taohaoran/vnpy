# 数据结构与枚举体系（objects-constants）

> 本文是 `trader-core` 域下的叶子子系统文档。域级总览见 `../trader-core.md`。
> 本文只展开 vnpy 核心数据结构（dataclass）与枚举定义，不重复展开引擎对数据的消费（见 oms-engine 叶子）。
>
> 源码基准：vnpy 4.4.0，commit `fa5206fe`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 数据基类 | BaseData 持有 gateway_name 来源标记与 extra 扩展字段 | `vnpy/trader/object.py:18` |
| 行情数据 | TickData（五档盘口+涨跌停）、BarData（K线） | `object.py:30,88` |
| 交易数据 | OrderData/TradeData/PositionData/AccountData/QuoteData | `object.py:112,154,179,201,265` |
| 合约与日志 | ContractData（合约规格+期权字段）、LogData | `object.py:233,219` |
| 请求类 | SubscribeRequest/OrderRequest/CancelRequest/HistoryRequest/QuoteRequest | `object.py:307,321,359,374,391` |
| 派生主键 | `__post_init__` 生成 vt_symbol/vt_orderid/vt_tradeid/vt_positionid/vt_accountid | 各 dataclass |
| 活跃判定 | OrderData.is_active()/QuoteData.is_active() 基于 ACTIVE_STATUSES | `object.py:137,290` |
| 请求转数据 | OrderRequest.create_order_data / QuoteRequest.create_quote_data | `object.py:339,410` |
| 枚举体系 | Direction/Offset/Status/Product/OrderType/Exchange/OptionType/Currency/Interval | `vnpy/trader/constant.py` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `BaseData` | `object.py:18` | 所有带 gateway_name 数据对象的基类 |
| `ACTIVE_STATUSES` | `object.py:14` | `{SUBMITTING, NOTTRADED, PARTTRADED}` 活跃状态集合 |
| `OrderData` | `object.py:112` | 委托数据，is_active/create_cancel_request |
| `ContractData` | `object.py:233` | 合约规格，含 net_position 标记驱动 OffsetConverter |
| `Exchange(Enum)` | `constant.py:82` | 全球交易所枚举（含 LOCAL/GLOBAL 特殊值） |
| `Offset(Enum)` | `constant.py:19` | 开/平/平今/平昨，平仓换算核心 |
| `Status(Enum)` | `constant.py:30` | 委托六态状态机 |

## 3. 关键调用链

1. **主键派生链**：`TickData.__post_init__`（`object.py:82`）→ `self.vt_symbol = f"{symbol}.{exchange.value}"`。OrderData 额外派生 `vt_orderid = f"{gateway_name}.{orderid}"`（`object.py:135`）。PositionData 派生 `vt_positionid = f"{gateway_name}.{vt_symbol}.{direction.value}"`（`object.py:197`）。
2. **下单对象生成链**：策略构造 OrderRequest → `OrderRequest.create_order_data(orderid, gateway_name)`（`object.py:339`）→ 填充 OrderData（status 默认 SUBMITTING）→ 网关 on_order 推事件。
3. **活跃判定链**：`OrderData.is_active()`（`object.py:137`）→ `return self.status in ACTIVE_STATUSES`。OmsEngine.process_order_event 据此维护 active_orders 字典增删（engine.py:405-409）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `ContractData.net_position` | False；True 时该合约不做平仓换算 | `object.py:248` |
| `ContractData.min_volume` | 默认 1 | `object.py:245` |
| `OrderData.status` | 默认 Status.SUBMITTING | `object.py:128` |
| `AccountData.available` | `balance - frozen` 派生 | `object.py:214` |
| 枚举本地化 | 中文展示值经 `locale._()` 翻译 | `constant.py:7,14` |

## 5. 错误与重试语义

- 数据类是纯值对象，**无错误处理、无重试**——字段类型由 dataclass 校验，运行时不抛业务异常。
- `AccountData.available` 在 `__post_init__` 计算，若 balance/frozen 类型不符由 Python 报错。
- 无错误包装；非法状态转换（如 SUBMITTING→ALLTRADED 跳过 PARTTRADED）由交易所柜台推送决定，本层不做状态机校验。

## 6. 并发细节

- 数据对象在 `__post_init__` 后视为**不可变**（BaseGateway 文档要求 on_xxx 传前 copy.copy），因此可跨事件分发线程安全共享引用。
- 无锁、无后台线程；所有派生主键在构造时一次性算好。
- `extra: dict | None`（`object.py:26`）是可变扩展槽，框架不触碰，由网关/策略自由挂载。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 全部数据 dataclass 与枚举定义
- vt_* 复合主键派生逻辑
- 请求→数据对象转换方法

**Out-of-Scope（不在本仓库源码内）**
- 数据持久化序列化（见 database-interface 叶子）
- 网关如何填充这些字段（各网关实现）
- 交易所枚举对应的真实交易所系统（外部）

## 8. 与相邻子系统交互

- 上游：BaseGateway 子类构造这些数据对象并经 on_xxx 推送（见 gateway-converter 叶子）
- 下游：OmsEngine 按 vt_*id 缓存这些对象（见 oms-engine 叶子）；OffsetConverter 读取 Offset/Direction/Exchange 字段做平仓换算（见 gateway-converter 叶子）
- 横向：枚举被所有层引用，是全框架的类型安全契约

## 9. 语言专项适配口径（纯 Python）

- **能力缝**：本叶子是"数据契约层"——dataclass 定义跨层数据结构，枚举定义合法值域。无注册表/工厂，但 `__post_init__` 是隐式的派生计算钩子。
- **dataclass 配置面**：字段默认值即配置（status=SUBMITTING、offset=NONE、price=0），构造时即定型。
- **复合主键模式**：`vt_symbol = symbol.exchange`、`vt_orderid = gateway.orderid` 是框架级命名约定，OmsEngine 字典全靠此做 key——这是纯 Python 框架中用字符串拼接代替分布式 ID 的典型做法。
- **图型侧重**：architecture 表达继承与引用拓扑；lifecycle 表达委托状态机（纯 Python 框架中状态枚举是显式的，比隐藏在循环里的状态机更易建模）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 数据结构与枚举体系架构图 | `objects-constants-architecture.html` | architecture | standard |
| 委托活跃状态流转 | `objects-constants-lifecycle.html` | lifecycle | standard |

- 架构图落 standard 原因：初版 enums→trading 连线穿过 mkt 节点；修复动作：删除该参考连线（枚举引用在 MD 文字说明）。
- 生命周期图落 standard 原因：archify lifecycle 布局对多状态分支敏感，多状态/同列堆叠反复触发 `clean-flow/endpoint-side-direction` 与 `edge-through-node`；修复动作：简化为"活跃中→已完结"两态主干图，六态完整状态机在 MD 第 3 节文字描述。
- 完整委托六态（SUBMITTING/NOTTRADED/PARTTRADED/ALLTRADED/CANCELLED/REJECTED）见 `constant.py:30-39`，ACTIVE_STATUSES 见 `object.py:14`。
- JSON IR 源文件位于 `json/` 目录。
