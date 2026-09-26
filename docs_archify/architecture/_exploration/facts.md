# vnpy 项目探查事实（共享基线，分片直接读此文件，勿重扫仓库）

> 本文件由 MainAgent 在首次只读探查时产出，供组织者与各分片共享。
> 分片只做叶子级深读（grep 定位 + Read offset 精读），不要再做全仓库扫描。

## 1. 身份

- 项目：vnpy（VeighNa）——基于 Python 的开源量化交易系统开发框架（README 定位："A framework for developing quant trading systems."）
- 版本：4.4.0（`vnpy/__init__.py` 的 `__version__`；pyproject 动态版本）
- 提交：`fa5206fe`（master，"[Mod] update README.md"）
- 语言：纯 Python（主语言判定：Python，无 C++ 核心、无绑定层）
- 依赖：Python >=3.10；核心依赖 tzlocal / PySide6 / pyqtgraph / qdarkstyle / numpy / pandas / ta-lib / deap / pyzmq / plotly / tqdm / loguru / nbformat / requests / qrcode；可选 extras：alpha（polars / scipy / alphalens-reloaded / scikit-learn / lightgbm / torch / pyarrow）、dev
- 许可证：MIT；作者：Xiaoyou Chen
- 构建：hatchling；含 locale 编译 hook（`vnpy/trader/locale/build_hook.py`）

## 2. 规模

- vnpy/ 包内 59 个 py 文件，合计 12,840 行
- 子包分布：alpha 25 py / trader 22 py / chart 6 py / rpc 4 py / event 2 py
- 顶层文件行数关键：trader/utility.py 1281、trader/engine.py 838、alpha/lab.py 480、trader/object.py 427、trader/converter.py 402、trader/wechat.py 307、trader/gateway.py 272、trader/optimize.py 250、trader/constant.py 160、trader/database.py 159、trader/datafeed.py 68、trader/logger.py 55、trader/setting.py 43、trader/app.py 21、trader/event.py 14
- tests/：tests/alpha/test_dataproxy.py、tests/test_alpha101.py（2 个测试文件）
- examples/：alpha_research、candle_chart、client_server、cta_backtesting、data_recorder、download_bars、no_ui、notebook_trading、portfolio_backtesting、simple_rpc、spread_backtesting、veighna_trader（13 个示例目录，均为运行示例，不在 vnpy/ 包源码内）
- CI：.github/workflows/pythonapp.yml

## 3. 结构

```
vnpy/
├── alpha/          # AI 量化策略模块（4.0 新增，25 py）
│   ├── lab.py          # 480 行，Lab 统筹类（数据集/模型/策略一站式）
│   ├── logger.py
│   ├── dataset/        # 因子特征工程
│   │   ├── template.py     # 数据集抽象基类
│   │   ├── processor.py    # 数据预处理（缺失值/无穷值/标准化等）
│   │   ├── cs_function.py  # 横截面函数库
│   │   ├── ts_function.py  # 时序函数库
│   │   ├── math_function.py
│   │   ├── ta_function.py  # TA-Lib 封装
│   │   ├── utility.py
│   │   └── datasets/       # alpha_101.py、alpha_158.py（微软 Qlib 风格因子集）
│   ├── model/         # 预测模型
│   │   ├── template.py     # 模型抽象基类
│   │   └── models/         # lasso_model.py、lgb_model.py、mlp_model.py
│   └── strategy/      # 策略投研
│       ├── template.py     # 策略抽象基类
│       ├── backtesting.py  # 回测
│       └── strategies/     # equity_demo_strategy.py 示例策略
├── chart/           # K线图表控件（6 py）：widget.py、manager.py、item.py、axis.py、base.py
├── event/           # 事件驱动引擎（2 py）：engine.py（EventEngine/Event/EventDispatcher）
├── rpc/             # RPC 通信（4 py）：server.py、client.py、common.py
└── trader/          # 核心交易框架（22 py）
    ├── engine.py        # 838 行：MainEngine/BaseEngine/OmsEngine/AppLoader
    ├── object.py        # 427 行：TickData/BarData/OrderData/TradeData/PositionData 等
    ├── constant.py      # 160 行：Direction/Offset/Status/Exchange/Interval 等枚举
    ├── gateway.py       # 272 行：BaseGateway 抽象（交易网关接口）
    ├── converter.py     # 402 行：OffsetConverter 委托转换器
    ├── utility.py       # 1281 行：工具函数 + BarGenerator/ArrayManager（K线合成）
    ├── database.py      # 159 行：BaseDatabase 抽象（行情/账户数据落库接口）
    ├── datafeed.py      # 68 行：BaseDatafeed 抽象（历史数据服务接口）
    ├── optimizer.py     # 250 行：OptimizationSetting（deap 遗传算法参数优化）
    ├── wechat.py        # 307 行：微信推送通知
    ├── logger.py        # 55 行：Logger 封装
    ├── setting.py       # 43 行：SETTINGS 配置
    ├── app.py           # 21 行：BaseApp 应用基类
    ├── event.py         # 14 行：事件引擎封装（转 vnpy.event）
    ├── ui/              # PySide6 GUI：mainwindow.py、qt.py、widget.py + ico
    └── locale/          # 国际化（.po/.pot 编译为 .mo）
```

## 4. 功能清单（README Feature List 摘录）

- 4.0 重磅：`vnpy.alpha`——一站式多因子机器学习策略开发、投研与实盘交易解决方案：
  - dataset：因子特征工程（批量特征计算、表达式计算引擎、自定义函数注册、缺失值/无穷值/标准化/特征删除处理；Alpha 158 = 微软 Qlib 股票特征集，Alpha 101 亦内置）
  - model：标准化 ML 模型开发模板、统一 API、Lasso/LightGBM/MLP 三实现
  - strategy：策略投研开发（策略模板 + 回测）
- 核心框架：事件驱动引擎（vnpy.event）、主引擎与功能引擎（MainEngine/BaseEngine）、交易网关抽象（BaseGateway）、数据结构体系、数据库/数据服务接口抽象、RPC 通信（vnpy.rpc）、K线图表（vnpy.chart）、桌面 GUI（vnpy.trader.ui）、微信通知（wechat）、遗传算法参数优化（optimize）

## 5. 约束（硬性）

- 输出根：项目根已存在 `docs/`（用户自有 Sphinx 文档目录），按规则输出根改为 `<repo>/docs_archify/architecture/`
- 基线：`docs/architecture/` 与 `docs_archify/architecture/` 均不存在 → **首次分析，无基线**，不产出 improve-comparison.md
- 输出物一律简体中文（MD/README/HTML 图作者内容/JSON IR 作者字段）；代码标识符、路径、flag 名保持原文
- 文件名/目录名英文短横线（-），禁止中文短横线
- 外部系统/第三方组件（交易所、数据库产品、PySide6 等）一律标注"不在本仓库源码内"
- 主语言 = 纯 Python → 读 `references/language-python.md` 并按"能力缝 + 注册表 + 插件体系"口径分析
- 只新增/修改 `docs_archify/` 下文件，不触碰仓库源码

## 6. 域与叶子盘点（MainAgent 划分，供组织者参考）

| 域 | 叶子 | 范围（源码） |
|---|---|---|
| trader-core | main-engine | engine.py 中 MainEngine/BaseEngine/AppLoader |
| trader-core | oms-engine | engine.py 中 OmsEngine（委托/持仓/账户管理） |
| trader-core | event-engine | event/engine.py + trader/event.py |
| trader-core | objects-constants | object.py + constant.py |
| trader-core | gateway-converter | gateway.py + converter.py |
| trader-service | config-utility | utility.py + setting.py + logger.py |
| trader-service | database-interface | database.py |
| trader-service | datafeed-interface | datafeed.py |
| trader-service | notify-wechat | wechat.py |
| trader-service | optimizer | optimizer.py |
| trader-ui | main-window | ui/mainwindow.py + qt.py + widget.py |
| rpc | rpc-communication | rpc/server.py + client.py + common.py |
| chart | chart-widget | chart/（widget/manager/item/axis/base） |
| alpha | alpha-lab | alpha/lab.py |
| alpha | alpha-dataset | alpha/dataset/（template/processor/函数库/datasets） |
| alpha | alpha-model | alpha/model/（template + 三模型） |
| alpha | alpha-strategy | alpha/strategy/（template + backtesting + 示例策略） |
