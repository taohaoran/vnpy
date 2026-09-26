# Alpha 因子研究（alpha）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`vnpy` 4.4.0（master），commit `fa5206fe`；纯 Python + polars。

## 1. 域职责

alpha 域是 VeighNa 的**量化因子研究与投研回测子系统**，覆盖从历史 K 线出发、构造因子特征、训练预测模型、生成交易信号并做组合回测的完整研究闭环。它面向"量化研究员"而非实盘交易员：所有数据落本地 parquet/pickle，离线批量运行，产出信号 DataFrame 后交给回测引擎验证策略收益。

核心定位是**编排层**：不直接做张量计算（polars/sklearn/lightgbm/torch 承担），而是用三套抽象基类（AlphaDataset / AlphaModel / AlphaStrategy）+ 仓储门面（AlphaLab）把"数据→因子→模型→策略→回测"串成可复用流水线。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序/数据流/生命周期图 | 职责一句话 |
|------|------|--------|------------------------|-----------|
| alpha-lab | [alpha-lab.md](alpha-lab/alpha-lab.md) | [架构图](alpha-lab/alpha-lab-architecture.html) | [数据流图](alpha-lab/alpha-lab-dataflow.html) | 研究目录仓储，统一管理 K线/数据集/模型/信号/成分/合约 |
| alpha-dataset | [alpha-dataset.md](alpha-dataset/alpha-dataset.md) | [架构图](alpha-dataset/alpha-dataset-architecture.html) | [数据流图](alpha-dataset/alpha-dataset-dataflow.html) | 因子特征工程：表达式 DSL、多进程并行算因子、预处理链、三时段划分 |
| alpha-model | [alpha-model.md](alpha-model/alpha-model.md) | [架构图](alpha-model/alpha-model-architecture.html) | [生命周期图](alpha-model/alpha-model-lifecycle.html) | 预测模型模板 + Lasso/LightGBM/MLP 三后端实现 |
| alpha-strategy | [alpha-strategy.md](alpha-strategy/alpha-strategy.md) | [架构图](alpha-strategy/alpha-strategy-architecture.html) | [生命周期图](alpha-strategy/alpha-strategy-lifecycle.html) | 策略模板 + 回测引擎（历史回放/限价单撮合/逐日盯市/绩效统计） |

## 3. 域级机制细节

### 3.1 研究全流程（数据→因子→模型→策略）

1. **数据准备**：`AlphaLab.save_bar_data` 把历史 K 线存 parquet；`load_bar_df` 批量读多标的并归一化、停牌置 NaN。
2. **特征工程**：构造 `AlphaDataset(df, train, valid, test)`，`add_feature` 注册因子表达式，`prepare_data` 用 `multiprocessing.Pool(spawn)` 并行算因子，`process_data` 跑 infer/learn 预处理链。
3. **模型训练**：`AlphaModel.fit(dataset)` 从 `fetch_learn(TRAIN/VALID)` 取矩阵训练，`predict(dataset, TEST)` 产出信号。
4. **信号落盘**：预测值拼成 `signal_df`（datetime/vt_symbol/signal）经 `save_signal` 存 parquet。
5. **策略回测**：`BacktestingEngine` 加载 K 线与合约配置，逐根 bar 回放，策略 `on_bars` 经 `get_signal` 取信号、`set_target` 目标仓位调仓，引擎撮合限价单并逐日盯市出绩效。

### 3.2 三套抽象基类能力缝

- **AlphaDataset**（`dataset/template.py:23`）：表达式注册 + 并行计算 + 预处理链 + 三时段 fetch；Alpha101/Alpha158 子类注册内置因子。
- **AlphaModel**（`model/template.py:9`）：`fit`/`predict` 抽象契约；Lasso/Lgb/Mlp 三实现矩阵封装 sklearn/lightgbm/torch。
- **AlphaStrategy**（`strategy/template.py:15`）：`on_init`/`on_bars`/`on_trade` 回调钩子 + `execute_trading` 目标仓位调仓；EquityDemo 是 top_k 选股示例。

### 3.3 表达式引擎与注册表

`calculate_by_expression`（`dataset/utility.py:203`）把列包成 `DataProxy`（重载算术运算符）、注入 ts/cs/ta/math 函数命名空间后 `eval`；`register_functions` 注册自定义算子到 `EXPRESSION_FUNCTIONS` 全局表。

### 3.4 持久化双轨

对象类（数据集/模型）用 pickle 整对象序列化（.pkl），表格类（信号/K线）用 parquet 列式存储；指数成分用 shelve。

## 4. 域级图

![alpha 域架构图](alpha-architecture.html)

![alpha 因子管线数据流图](alpha-dataflow.html)

![alpha 研究流程时序图](alpha-sequence.html)
