# vnpy（VeighNa）系统级总览

> 源码基准：vnpy 4.4.0，commit fa5206fe，MIT 许可证，作者 Xiaoyou Chen。
> 主语言：纯 Python（>=3.10），无 C++ 核心与绑定层。vnpy/ 包内 59 个 py 文件，合计 12,840 行。

## 1. 项目概述

vnpy（VeighNa）是一款基于 Python 的开源量化交易系统开发框架，README 定位为 "A framework for developing quant trading systems"。它以**事件驱动**为核心范式，通过**引擎插件化**和**抽象接口**设计，为量化交易者提供从行情接入、策略开发、订单管理到风险控制的全栈解决方案。

4.0 版本新增 `vnpy.alpha` 模块，提供一站式多因子机器学习策略开发、投研与实盘交易能力，涵盖因子特征工程（Alpha 101 / Alpha 158）、标准化 ML 模型模板（Lasso / LightGBM / MLP）和策略回测。

### 代码规模

| 子包 | py 文件数 | 说明 |
|------|----------|------|
| vnpy/trader | 22 | 核心交易框架（引擎/网关/数据结构/UI/工具） |
| vnpy/alpha | 25 | AI 量化策略模块（因子/模型/策略/回测） |
| vnpy/chart | 6 | K线图表控件（pyqtgraph） |
| vnpy/rpc | 4 | 远程过程调用通信（pyzmq） |
| vnpy/event | 2 | 事件驱动引擎 |
| **合计** | **59** | **12,840 行** |

## 2. 功能总览

| 领域 | 功能模块 | 归属域 / 叶子 |
|------|---------|--------------|
| 核心引擎 | MainEngine 主引擎（引擎编排/应用加载） | trader-core / main-engine |
| 核心引擎 | OmsEngine 订单管理（委托/成交/持仓/账户） | trader-core / oms-engine |
| 核心引擎 | EventEngine 事件总线（发布订阅） | trader-core / event-engine |
| 核心引擎 | 数据结构与枚举（TickData/BarData/OrderData 等） | trader-core / objects-constants |
| 核心引擎 | BaseGateway 交易网关抽象 + OffsetConverter | trader-core / gateway-converter |
| 基础服务 | 工具函数/BarGenerator/ArrayManager/配置/日志 | trader-service / config-utility |
| 基础服务 | BaseDatabase 数据库接口抽象 | trader-service / database-interface |
| 基础服务 | BaseDatafeed 历史数据服务接口抽象 | trader-service / datafeed-interface |
| 基础服务 | 微信推送通知 | trader-service / notify-wechat |
| 基础服务 | 遗传算法参数优化（deap） | trader-service / optimizer |
| 桌面 GUI | MainWindow 主窗口 + Qt 组件 + 控件 | trader-ui / main-window |
| 远程通信 | RPC 服务端/客户端/通用模块（pyzmq） | rpc / rpc-communication |
| 图表组件 | K线图表 widget/manager/item/axis/base | chart / chart-widget |
| Alpha 投研 | AlphaLab 统筹引擎（数据集/模型/策略一站式） | alpha / alpha-lab |
| Alpha 投研 | 因子特征工程（函数库/Alpha101/Alpha158/预处理） | alpha / alpha-dataset |
| Alpha 投研 | 预测模型（模板 + Lasso/LightGBM/MLP 三实现） | alpha / alpha-model |
| Alpha 投研 | 策略模板 + 回测引擎 + 示例策略 | alpha / alpha-strategy |

## 3. 解决的问题

| 用户痛点 | vnpy 解法 |
|---------|----------|
| 各交易所 API 不统一，对接成本高 | BaseGateway 抽象基类统一交易网关接口，新增网关只需实现抽象方法 |
| 行情/订单/持仓数据结构各异 | object.py 定义统一数据模型（TickData/BarData/OrderData/TradeData/PositionData/AccountData） |
| 多组件间耦合严重，难以扩展 | 事件驱动架构（EventEngine）解耦生产者与消费者，引擎插件化（BaseEngine + AppLoader） |
| 历史数据/数据库选型绑定 | BaseDatabase / BaseDatafeed 抽象接口，支持任意数据库和数据供应商 |
| 策略参数优化耗时 | 内置 deap 遗传算法参数优化（OptimizationSetting） |
| K线图需从零开发 | vnpy.chart 提供基于 pyqtgraph 的专业 K线图表组件 |
| 多因子 ML 策略门槛高 | vnpy.alpha 提供数据集→模型→策略全流程模板，内置 Alpha 101/158 因子库 |
| 远程部署/分布式交易 | vnpy.rpc 提供基于 pyzmq 的 RPC 通信，支持客户端-服务端架构 |

## 4. 系统边界

### 上边界（用户 / 第三方接入）
- 策略开发者通过 MainWindow GUI 或脚本 API 与系统交互
- 第三方交易网关（如 CTP、IB、Binance 等）实现 BaseGateway 抽象接口接入——**具体网关实现不在本仓库源码内**（位于独立的 vnpy_* 网关包）
- 第三方数据库（MySQL/PostgreSQL/MongoDB 等）通过 BaseDatabase 接口适配——**数据库驱动与产品不在本仓库源码内**
- 第三方数据供应商（RQData/Tushare 等）通过 BaseDatafeed 接口适配——**数据供应商不在本仓库源码内**

### 下边界（基础设施）
- Python >=3.10 运行时
- 依赖库：PySide6（GUI）、pyqtgraph（图表）、pyzmq（RPC）、numpy/pandas（数据处理）、ta-lib（技术指标）、deap（遗传算法）、loguru（日志）、plotly（可视化）等——**均不在本仓库源码内**
- alpha extras：polars / lightgbm / torch / scikit-learn / scipy / alphalens-reloaded——**均不在本仓库源码内**

### 内边界（本仓库 vs 扩展 / 外部）
- **本仓库内**：vnpy/ 包（trader/alpha/chart/rpc/event 五个子包），定义核心框架、抽象接口、数据结构、基础工具和示例策略
- **本仓库外（扩展包）**：各交易网关实现（vnpy_ctp、vnpy_ib、vnpy_binance 等）、数据库适配（vnpy_mysql、vnpy_postgresql 等）、数据服务适配（vnpy_rqdata 等）、应用模块（vnpy_ctastrategy、vnpy_portfoliostrategy 等）
- **examples/**：13 个运行示例目录，为使用示例，不在 vnpy/ 包源码内

### 侧边界
- 支持单机运行（默认模式）和通过 RPC 的客户端-服务端分布式部署
- 支持 GUI 模式和 no_ui 纯脚本模式

### 不做什么
- 不包含具体交易所的网关实现（仅定义 BaseGateway 抽象）
- 不包含具体数据库的驱动实现（仅定义 BaseDatabase 抽象）
- 不包含具体数据供应商的客户端（仅定义 BaseDatafeed 抽象）
- 不提供行情数据本身（需通过 BaseDatafeed 或 BaseGateway 从外部获取）
- 不包含 CTA/价差/期权等具体策略模块（这些在独立的 vnpy_* 应用包中；本仓库 alpha 模块提供 ML 策略模板）
- 不提供回测引擎的通用实现（alpha/strategy/backtesting.py 仅服务于 alpha ML 策略；CTA 回测在 vnpy_ctastrategy 包中）

## 5. 系统架构图说明

![系统架构图](system-architecture.html)

系统采用**四层分层架构**：

1. **应用与展示层**：MainWindow 桌面端（PySide6 GUI，用户交互入口）、AlphaLab 研究端（因子投研环境）、K线图表组件（pyqtgraph 专业图表）
2. **核心引擎层**：MainEngine 主引擎（全局编排，管理所有功能引擎和网关）、OmsEngine 订单管理系统（维护委托/成交/持仓/账户的内存状态）、EventEngine 事件总线（发布-订阅模式，解耦各组件）
3. **服务接口层**：BaseGateway 交易网关抽象（统一报单/查仓/订阅接口）、BaseDatabase 数据库接口抽象（行情/持仓持久化）、BaseDatafeed 历史数据服务抽象（K线/Tick下载）、RPC 远程通信（pyzmq 客户端-服务端）
4. **外部系统**：交易所（网络通信对接）、数据库产品（持久化存储）、数据供应商（历史数据源）、微信推送（通知通道）

**主交易路径**（图中 emphasis 粗线）：策略开发者 → MainWindow → MainEngine → OmsEngine → BaseGateway → 交易所。

**未连线的关键关系**（图中卡片标注）：MainEngine 向 EventEngine 注册事件处理器；EventEngine 回调更新 MainWindow UI；OmsEngine 持仓经 BaseDatabase 落库；RPC 远程调用 MainEngine；AlphaLab 加载策略至 MainEngine。这些关系因跨层路由复杂未在图中连线，详见各叶子文档。

## 6. 核心时序图说明

![委托交易全流程时序图](system-trade-sequence.html)

时序图展示一笔委托从提交到成交的完整生命周期，分为两个阶段：

**阶段一：委托提交与回报（m1–m9）**
1. 策略开发者在 MainWindow 提交委托（m1）
2. MainWindow 调用 MainEngine.send_order()（m2）
3. MainEngine 转发至 OmsEngine.send_order()（m3）——OmsEngine 生成内部委托号并维护委托状态
4. OmsEngine 调用 BaseGateway.send_order()（m4）——网关将委托转换为交易所协议格式
5. BaseGateway 向交易所发送报单请求（m5）
6. 交易所返回委托回报（m6）——委托被接受/拒绝
7. BaseGateway 回调 OmsEngine.on_order()（m7）——更新委托状态
8. OmsEngine 向 EventEngine put(EVENT_ORDER)（m8）——发布订单事件
9. EventEngine 回调 MainWindow 更新委托列表（m9）

**阶段二：成交回报与持仓更新（m10–m13）**
10. 交易所返回成交回报（m10）
11. BaseGateway 回调 OmsEngine.on_trade()（m11）——更新成交记录和持仓
12. OmsEngine 向 EventEngine put(EVENT_TRADE)（m12）——发布成交事件
13. EventEngine 回调 MainWindow 更新持仓与成交（m13）

关键设计点：OmsEngine 是唯一的状态管理者，所有委托/成交/持仓/账户数据集中维护；EventEngine 实现了状态变更与 UI 更新的解耦；BaseGateway 屏蔽了不同交易所的协议差异。

## 7. 系统数据流图说明

![行情与交易数据流图](system-dataflow.html)

数据流图展示行情数据与交易事件在系统中的流动路径，分为五个阶段：

1. **数据源**：交易所（实时行情/委托/成交回报）、数据供应商（历史K线数据）
2. **接入层**：BaseGateway（接收交易所实时数据，转换为统一数据结构）、BaseDatafeed（从数据供应商下载历史数据）
3. **事件总线**：EventEngine（所有数据以事件形式发布，消费者按需订阅）
4. **消费层**：OmsEngine（处理订单/成交事件，更新持仓账户）、MainWindow（UI事件，更新界面）、K线图表（行情推送，实时绘图）、BaseDatabase（落库事件，持久化存储）
5. **存储**：数据库产品（行情/持仓/成交数据的最终持久化）

**主数据路径**（emphasis 粗线）：交易所 → BaseGateway → EventEngine → OmsEngine。这是实时交易数据的核心流动路径。

关键设计点：所有数据通过 EventEngine 统一分发，实现了数据源与数据消费者的完全解耦；BaseGateway 和 BaseDatafeed 作为接入层，将外部异构数据转换为 vnpy 统一数据模型后再进入事件总线；BaseDatabase 作为事件消费者，将需要持久化的数据写入外部数据库。
