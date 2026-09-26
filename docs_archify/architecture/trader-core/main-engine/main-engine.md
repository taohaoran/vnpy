# MainEngine 应用加载与引擎管理（main-engine）

> 本文是 `trader-core` 域下的叶子子系统文档。域级总览见 `../trader-core.md`。
> 本文只展开 MainEngine / BaseEngine / AppLoader 的容器与注册职责，不重复展开 OmsEngine 的持仓委托管理（见 `../oms-engine/oms-engine.md`）与 EventEngine 的事件分发机制（见 `../event-engine/event-engine.md`）。
>
> 源码基准：vnpy 4.4.0，commit `fa5206fe`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 核心容器 | MainEngine 作为交易平台核心，持有 EventEngine、网关表、引擎表、应用表 | `vnpy/trader/engine.py:81` MainEngine |
| 引擎注册 | add_engine 实例化功能引擎并按 engine_name 入表 | `vnpy/trader/engine.py:102` add_engine |
| 网关注册 | add_gateway 实例化网关、聚合其支持交易所列表 | `vnpy/trader/engine.py:110` add_gateway |
| 应用加载 | add_app 实例化 BaseApp 并调用其 engine_class 注册引擎 | `vnpy/trader/engine.py:128` add_app |
| 内置引擎初始化 | init_engines 自动注册 Log/Oms/Email/Wechat 引擎 | `vnpy/trader/engine.py:138` init_engines |
| 查询接口委托 | 把 OmsEngine 的 get_* 方法绑定为 MainEngine 自身属性 | `vnpy/trader/engine.py:145-163` |
| 交易接口转发 | connect/subscribe/send_order/cancel_order/send_quote/query_history 转发到网关 | `vnpy/trader/engine.py:234-308` |
| 通知下发 | send_notification 同时走 EmailEngine 与 WechatEngine | `vnpy/trader/engine.py:176` send_notification |
| 优雅关闭 | close 先停事件引擎，再依次关闭引擎与网关 | `vnpy/trader/engine.py:310` close |
| 日志事件投递 | write_log 构造 LogData 并 put 到事件引擎 | `vnpy/trader/engine.py:168` write_log |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `BaseEngine(ABC)` | `engine.py:59` | 功能引擎抽象基类，持有 main_engine/event_engine/engine_name 三引用，默认空 close |
| `MainEngine` | `engine.py:81` | 平台核心容器，管理引擎/网关/应用三类注册表，转发交易请求 |
| `EngineType` TypeVar | `engine.py:56` | 绑定 BaseEngine 子类，供 add_engine 返回类型标注 |
| `BaseApp(ABC)` | `vnpy/trader/app.py:10` | 应用插件抽象基类，声明 app_name/engine_class/widget_name 等元信息 |
| `LogEngine` | `engine.py:325` | 订阅 EVENT_LOG，按级别写 loguru |
| `EmailEngine` | `engine.py:590` | 独立线程 + Queue，异步 SMTP_SSL 发邮件 |
| `WechatEngine` | `engine.py:657` | 独立线程 + Queue + 限流合并，异步微信推送 |

## 3. 关键调用链

1. **启动链**：`MainEngine.__init__`（`engine.py:86`）→ 构造/复用 EventEngine 并 `start()` → `os.chdir(TRADER_DIR)` → `init_engines()`（`engine.py:138`）依次 `add_engine(LogEngine)`、`add_engine(OmsEngine)` 并把 OmsEngine 的 16 个 get_* 方法绑定到 MainEngine 属性，最后注册 EmailEngine/WechatEngine。
2. **下单转发链**：策略调 `MainEngine.send_order(req, gateway_name)`（`engine.py:254`）→ `get_gateway`（`engine.py:189`）查表，未命中写日志返回空串 → `write_log` 记录 → `gateway.send_order(req)` 返回 vt_orderid。网关回调经 `on_order` 推 EVENT_ORDER，由 EventEngine 异步分发给 OmsEngine。
3. **关闭链**：`MainEngine.close()`（`engine.py:310`）→ 先 `event_engine.stop()` 阻止新定时器事件 → 遍历 engines 调 close → 遍历 gateways 调 close。EmailEngine/WechatEngine 的 close 会置 active=False 并 join 工作线程。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `event_engine` | 不传则内部 `EventEngine()`（timer 间隔 1 秒） | `engine.py:86-91` |
| `SETTINGS["log.active"]` | LogEngine 是否输出日志 | `engine.py:342` |
| `SETTINGS["email.*"]` | server/port/username/password/receiver/sender | `engine.py:611-626` |
| `wechat_setting.json` | bot_id/token/base_url/user_id/send_interval（默认 60s 限流） | `engine.py:662,685-705` |
| `gateway_name` 缺省 | 未传则用 `gateway_class.default_name` | `engine.py:115-116` |

## 5. 错误与重试语义

- 网关未注册时（`get_gateway` 返回 None）：`connect/subscribe/send_order/cancel_order` 仅写日志并静默跳过；`send_order/send_quote` 返回空串 `""`，`query_history` 返回空列表——**不抛异常，由调用方判空**。
- `EmailEngine.run`（`engine.py:621`）：单条邮件发送失败捕获 `Exception` 后写日志，不中断工作线程；队列 `get` 超时 1s 触发 `Empty` 后继续循环。
- `WechatEngine.run`（`engine.py:783`）：`SessionExpired` 时把消息放回 pending_msgs 队首并置 active=False 停止；`WeixinError` 同样回退消息并写日志，不丢消息。
- 本层**无自动重试/退避**，重连责任在网关实现侧（BaseGateway 文档要求自动重连）。

## 6. 并发细节

- EventEngine 由 MainEngine 持有并 start（见 event-engine 叶子）；MainEngine 自身**不创建业务工作线程**。
- EmailEngine 与 WechatEngine 各有一个 `Thread` + `Queue` 生产者-消费者模型：`send_email/send_wechat` 仅 put 入队，工作线程异步发送，避免阻塞策略线程。
- WechatEngine 用 `time.monotonic()` 做发送间隔限流（`engine.py:804-810`），并在一次循环内 `get_nowait` 排空队列合并成一条消息发送。
- 关闭顺序：先停 EventEngine（定时器线程），再 join 引擎/网关工作线程，避免关闭过程中新事件投递到已关闭组件。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- MainEngine/BaseEngine/LogEngine/EmailEngine/WechatEngine 容器与注册逻辑
- add_engine/add_gateway/add_app 三类注册表
- OmsEngine 方法绑定与交易接口转发

**Out-of-Scope（不在本仓库源码内）**
- 具体网关实现（CTP/IB/TTS 等，均为独立仓库包）——本仓库只有抽象 BaseGateway
- 策略应用（cta_strategy 等）——本仓库只有 BaseApp 抽象
- 交易所柜台、SMTP 邮件服务器、微信 iLink 服务端——外部系统
- PySide6 GUI 层（见 trader-ui 域）

## 8. 与相邻子系统交互

- 上游：策略/上层应用 → MainEngine（调 send_order/subscribe/query_history 等）
- 下游：MainEngine → BaseGateway 子类（转发交易请求）；MainEngine → EventEngine（持有、投递事件）
- 横向：MainEngine 注册 OmsEngine 后把其 get_* 方法委托出去；OmsEngine 经 EventEngine 接收网关行情/委托事件（见 oms-engine 叶子）
- 通知：MainEngine.send_notification → EmailEngine + WechatEngine（后者依赖 vnpy/trader/wechat.py）

## 9. 语言专项适配口径（纯 Python）

- **能力缝（capability seam）**：本叶子是典型的"注册表 + 插件宿主"能力缝——`BaseEngine(ABC)` 定义功能引擎契约，`MainEngine.engines` 字典是字符串名→实例的注册表；`BaseApp(ABC)` 定义应用插件契约，`MainEngine.apps` 注册表 + `add_app` 触发 `engine_class` 实例化即"字符串→类路由"的工厂模式（此处由显式类引用而非字符串路由）。
- **注册表注册时机**：引擎注册在 `init_engines()` 启动时显式调用；网关/应用注册由用户在启动脚本中显式 `add_gateway/add_app`，非 import 副作用。
- **双轨/配置驱动**：不涉及。引擎行为由 `SETTINGS` 与各网关 `default_setting` 驱动，属配置注入而非运行时后端切换。
- **可选依赖**：EmailEngine/WechatEngine 懒启动——首次 send 才 start 线程（`engine.py:606-607`），未配置凭据时 WechatEngine 不激活。
- **图型侧重**：architecture 表达注册表拓扑；sequence 表达单次下单调用链。本叶子无隐藏循环状态机（Email/Wechat 工作线程是标准消费者循环，非业务状态机），故不补 lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| MainEngine 组件架构图 | `main-engine-architecture.html` | architecture | standard |
| 下单调用链时序 | `main-engine-sequence.html` | sequence | showcase |

- 架构图落 standard 原因：首次 showcase 校验报 `clean-flow/endpoint-side-direction`（多扇出连线端点方向不一致）与 `edge-through-node`（mainengine→apps 穿过 notify 节点）；采取的修复动作：精简组件（合并 Email/Wechat 为一个通知引擎盒）、显式设置 `fromSide`（left/bottom/right）、删除跨层次要连线（gateways→eventengine 改在 MD 第 8 节文字描述），仍因多扇出布局复杂度落 standard。
- 未补 dataflow/lifecycle/workflow：本叶子是容器注册与转发层，数据流动见 event-engine 叶子、委托状态机见 objects-constants 叶子，重复信息按资源节省原则省略。
- JSON IR 源文件位于 `json/` 目录。
