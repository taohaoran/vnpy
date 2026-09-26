# vnpy（VeighNa）系统架构文档

> 基于 vnpy 4.4.0 源码（commit fa5206fe，纯 Python，59 个 py 文件 / 12,840 行）深度分析产出，
> 覆盖系统级、6 个域、17 个叶子子系统的功能、问题域、系统边界、架构图、时序图与数据流图。
> 所有图表由 archify 渲染为自包含交互式 HTML。

## 文档导航

### 系统级

| 文档 | 说明 | 图表 |
|------|------|------|
| [system-overview.md](system-overview.md) | 项目概述、功能总览、解决的问题、系统边界、架构/时序/数据流图说明 | [系统架构图](system-architecture.html) · [委托交易时序图](system-trade-sequence.html) · [行情数据流图](system-dataflow.html) |

---

### trader-core 域（核心交易引擎）

| 子系统 | 文档 | 架构图 | 第二图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [trader-core.md](trader-core/trader-core.md) | [架构图](trader-core/trader-core-architecture.html) | [数据流](trader-core/trader-core-dataflow.html) · [时序](trader-core/trader-core-sequence.html) | 引擎编排、订单管理、事件驱动、数据结构、网关抽象 |
| main-engine | [main-engine.md](trader-core/main-engine/main-engine.md) | [架构图](trader-core/main-engine/main-engine-architecture.html) | [时序图](trader-core/main-engine/main-engine-sequence.html) | MainEngine/BaseEngine/AppLoader 应用加载与引擎管理 |
| oms-engine | [oms-engine.md](trader-core/oms-engine/oms-engine.md) | [架构图](trader-core/oms-engine/oms-engine-architecture.html) | [数据流图](trader-core/oms-engine/oms-engine-dataflow.html) | OmsEngine 委托/成交/持仓/账户/请求管理 |
| event-engine | [event-engine.md](trader-core/event-engine/event-engine.md) | [架构图](trader-core/event-engine/event-engine-architecture.html) | [数据流图](trader-core/event-engine/event-engine-dataflow.html) | EventEngine 事件驱动发布订阅引擎 |
| objects-constants | [objects-constants.md](trader-core/objects-constants/objects-constants.md) | [架构图](trader-core/objects-constants/objects-constants-architecture.html) | [生命周期图](trader-core/objects-constants/objects-constants-lifecycle.html) | TickData/BarData/OrderData 等数据结构与 Direction/Offset 枚举 |
| gateway-converter | [gateway-converter.md](trader-core/gateway-converter/gateway-converter.md) | [架构图](trader-core/gateway-converter/gateway-converter-architecture.html) | [时序图](trader-core/gateway-converter/gateway-converter-sequence.html) | BaseGateway 交易网关抽象 + OffsetConverter 开平转换器 |

---

### trader-service 域（基础服务）

| 子系统 | 文档 | 架构图 | 第二图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [trader-service.md](trader-service/trader-service.md) | [架构图](trader-service/trader-service-architecture.html) | [数据流](trader-service/trader-service-dataflow.html) · [时序](trader-service/trader-service-sequence.html) | 工具函数、数据库/数据服务接口、通知、参数优化 |
| config-utility | [config-utility.md](trader-service/config-utility/config-utility.md) | [架构图](trader-service/config-utility/config-utility-architecture.html) | [数据流图](trader-service/config-utility/config-utility-dataflow.html) | BarGenerator/ArrayManager K线合成 + 工具函数 + 配置 + 日志 |
| database-interface | [database-interface.md](trader-service/database-interface/database-interface.md) | [架构图](trader-service/database-interface/database-interface-architecture.html) | [时序图](trader-service/database-interface/database-interface-sequence.html) | BaseDatabase 数据库持久化抽象基类 |
| datafeed-interface | [datafeed-interface.md](trader-service/datafeed-interface/datafeed-interface.md) | [架构图](trader-service/datafeed-interface/datafeed-interface-architecture.html) | [时序图](trader-service/datafeed-interface/datafeed-interface-sequence.html) | BaseDatafeed 历史数据服务抽象基类 |
| notify-wechat | [notify-wechat.md](trader-service/notify-wechat/notify-wechat.md) | [架构图](trader-service/notify-wechat/notify-wechat-architecture.html) | [时序图](trader-service/notify-wechat/notify-wechat-sequence.html) | 微信推送通知（扫码登录 + 消息发送） |
| optimizer | [optimizer.md](trader-service/optimizer/optimizer.md) | [架构图](trader-service/optimizer/optimizer-architecture.html) | [生命周期图](trader-service/optimizer/optimizer-lifecycle.html) | deap 遗传算法策略参数优化 |

---

### trader-ui 域（桌面 GUI）

| 子系统 | 文档 | 架构图 | 第二图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [trader-ui.md](trader-ui/trader-ui.md) | [架构图](trader-ui/trader-ui-architecture.html) | [时序](trader-ui/trader-ui-sequence.html) · [数据流](trader-ui/trader-ui-dataflow.html) | PySide6 桌面 GUI 整体架构 |
| main-window | [main-window.md](trader-ui/main-window/main-window.md) | [架构图](trader-ui/main-window/main-window-architecture.html) | [时序图](trader-ui/main-window/main-window-sequence.html) | MainWindow 主窗口 + Qt 组件 + 控件 + 事件→UI 更新 |

---

### rpc 域（远程通信）

| 子系统 | 文档 | 架构图 | 第二图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [rpc.md](rpc/rpc.md) | [架构图](rpc/rpc-architecture.html) | [数据流](rpc/rpc-dataflow.html) · [时序](rpc/rpc-sequence.html) | pyzmq 远程过程调用通信整体架构 |
| rpc-communication | [rpc-communication.md](rpc/rpc-communication/rpc-communication.md) | [架构图](rpc/rpc-communication/rpc-communication-architecture.html) | [时序图](rpc/rpc-communication/rpc-communication-sequence.html) | RPC 服务端/客户端/通用模块，请求-响应与发布订阅 |

---

### chart 域（K线图表）

| 子系统 | 文档 | 架构图 | 第二图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [chart.md](chart/chart.md) | [架构图](chart/chart-architecture.html) | [数据流](chart/chart-dataflow.html) · [时序](chart/chart-sequence.html) | pyqtgraph K线图表组件整体架构 |
| chart-widget | [chart-widget.md](chart/chart-widget/chart-widget.md) | [架构图](chart/chart-widget/chart-widget-architecture.html) | [数据流图](chart/chart-widget/chart-widget-dataflow.html) | ChartWidget/ChartManager/CandleItem 等 K线图渲染全组件 |

---

### alpha 域（AI 量化投研）

| 子系统 | 文档 | 架构图 | 第二图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [alpha.md](alpha/alpha.md) | [架构图](alpha/alpha-architecture.html) | [数据流](alpha/alpha-dataflow.html) · [时序](alpha/alpha-sequence.html) | 数据集→模型→策略全流程 ML 量化投研框架 |
| alpha-lab | [alpha-lab.md](alpha/alpha-lab/alpha-lab.md) | [架构图](alpha/alpha-lab/alpha-lab-architecture.html) | [数据流图](alpha/alpha-lab/alpha-lab-dataflow.html) | AlphaLab 统筹引擎（数据集/模型/策略一站式管理） |
| alpha-dataset | [alpha-dataset.md](alpha/alpha-dataset/alpha-dataset.md) | [架构图](alpha/alpha-dataset/alpha-dataset-architecture.html) | [数据流图](alpha/alpha-dataset/alpha-dataset-dataflow.html) | 因子特征工程（函数库/Alpha101/Alpha158/数据预处理） |
| alpha-model | [alpha-model.md](alpha/alpha-model/alpha-model.md) | [架构图](alpha/alpha-model/alpha-model-architecture.html) | [生命周期图](alpha/alpha-model/alpha-model-lifecycle.html) | 预测模型模板 + Lasso/LightGBM/MLP 三实现 |
| alpha-strategy | [alpha-strategy.md](alpha/alpha-strategy/alpha-strategy.md) | [架构图](alpha/alpha-strategy/alpha-strategy-architecture.html) | [生命周期图](alpha/alpha-strategy/alpha-strategy-lifecycle.html) | 策略模板 + 回测引擎 + 示例策略 |

---

## 产出统计

| 层级 | MD 文档 | HTML 图表 | JSON IR | 说明 |
|------|--------|----------|---------|------|
| 系统级 | 1 | 3 | 3 | system-overview + architecture/sequence/dataflow |
| trader-core 域 | 6 | 13 | 13 | 1 域总览 + 5 叶子（每叶 2 图）+ 3 域图 |
| trader-service 域 | 6 | 13 | 13 | 1 域总览 + 5 叶子 + 3 域图 |
| trader-ui 域 | 2 | 5 | 5 | 1 域总览 + 1 叶子 + 3 域图 |
| rpc 域 | 2 | 5 | 5 | 1 域总览 + 1 叶子 + 3 域图 |
| chart 域 | 2 | 5 | 5 | 1 域总览 + 1 叶子 + 3 域图 |
| alpha 域 | 5 | 11 | 11 | 1 域总览 + 4 叶子 + 3 域图 |
| **合计** | **25** | **55** | **55** | 17 叶子 / 6 域 / 系统级 |

## 图质量档位说明

| 档位 | 数量 | 说明 |
|------|------|------|
| showcase | 38 | 0 错误通过校验（系统级 sequence/dataflow、大部分域级与叶子级图） |
| standard | 17 | 系统级 architecture（组件多跨层复杂为预期内）、trader-core 域多数图（多扇出架构布局严格）、alpha 域部分图（已在对应叶子 MD 第 10 节披露失败检查名与修复动作） |

所有 55 张图 render 退出码均为 0，HTML 自包含非空（780KB+）。

## 覆盖范围与说明

- **叶子清单（17 个）**：main-engine、oms-engine、event-engine、objects-constants、gateway-converter、config-utility、database-interface、datafeed-interface、notify-wechat、optimizer、main-window、rpc-communication、chart-widget、alpha-lab、alpha-dataset、alpha-model、alpha-strategy
- **语言适配口径**：主语言为纯 Python，按"能力缝 + 注册表 + 插件体系"分析——识别出 BaseEngine/BaseApp/BaseGateway/BaseDatabase/BaseDatafeed/AlphaDataset/AlphaModel/AlphaStrategy 等多处抽象基类契约缝、MainEngine 字典注册表、EXPRESSION_FUNCTIONS 函数注册表、OffsetConverter 配置驱动三模式分派、Qt Signal 跨线程桥等纯 Python 框架特征
- **外部组件标注**：交易所、数据库产品、数据供应商、PySide6、pyqtgraph、pyzmq、ta-lib、deap、numpy、pandas、lightgbm、torch、scikit-learn、polars、微信 API 等均标注"不在本仓库源码内"
- **未覆盖项**：examples/ 目录下 13 个运行示例（不在 vnpy/ 包源码内）、tests/ 目录（仅 2 个测试文件）、各独立网关/数据库/应用扩展包（vnpy_ctp、vnpy_mysql、vnpy_ctastrategy 等，不在本仓库内）
- **源码事实校正**：所有叶子 MD 关键调用链均含至少 3 处真实 `文件:函数:行号`，经 grep 定位 + Read offset 精读核实
