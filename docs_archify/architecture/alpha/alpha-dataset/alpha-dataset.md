# 因子特征工程（alpha-dataset）

> 本文是 `alpha` 域下的叶子子系统文档。域级总览见 `../alpha.md`。
>
> 源码基准：`vnpy` 4.4.0（master），commit `fa5206fe`；纯 Python + polars，主语言口径见本文第 9 节。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 数据集抽象基类 | `AlphaDataset`：持有原始 df、feature_expressions/feature_results、label_expression、infer/learn processors，三时段划分 | `vnpy/alpha/dataset/template.py:23` |
| 特征表达式注册 | `add_feature(name, expression)` 注册字符串表达式或 polars Expr；`set_label` 注册标签 | `template.py:58,75` |
| 多进程因子计算 | `prepare_data` 用 `multiprocessing.Pool(spawn)` 并行计算所有表达式因子，结果拼回 df | `template.py:90,108-118` |
| 表达式计算引擎 | `calculate_by_expression`：把列名包成 DataProxy、注入 ts/cs/ta/math 函数命名空间，`eval` 求值 | `vnpy/alpha/dataset/utility.py:203` |
| 函数库 | ts_*（时序）/cs_*（横截面）/ta_*（TA-Lib）/math_*（数学）四类算子 | `ts_function.py`/`cs_function.py`/`ta_function.py`/`math_function.py` |
| DataProxy 运算符重载 | 重载 +/-/*/比较等魔法方法，让表达式字符串里的列可直接做运算 | `utility.py:56-200` |
| 自定义函数注册 | `register_functions([func])` 把用户函数注入表达式命名空间 | `utility.py:10-16` |
| 数据预处理 | 缺失值/无穷值/横截面标准化/时序标准化/去特征等 9 个 processor | `vnpy/alpha/dataset/processor.py` |
| 三时段数据获取 | `fetch_raw/fetch_infer/fetch_learn(segment)` 按 TRAIN/VALID/TEST 切片 | `template.py:174-193` |
| 因子绩效分析 | 接入 alphalens-reloaded 画因子/信号 tear sheet | `template.py:195,242` |
| 内置因子集 | Alpha101（WorldQuant 101 因子）、Alpha158（Qlib 158 因子） | `datasets/alpha_101.py:4`、`datasets/alpha_158.py:4` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `AlphaDataset` | `template.py:23` | 数据集模板类：表达式注册、prepare_data 并行计算、process_data 预处理、三时段 fetch |
| `Segment`（Enum） | `utility.py:280` | TRAIN=1/VALID=2/TEST=3 数据段枚举 |
| `DataProxy` | `utility.py:19` | 单列数据代理，重载所有算术/比较运算符，使表达式可写 `(close-open)/open` |
| `calculate_by_expression` | `utility.py:203` | 字符串表达式求值：注入列名 DataProxy + 函数库，`eval` 执行 |
| `calculate_by_polars` | `utility.py:258` | polars Expr 表达式直接 select 求值 |
| `calculate_feature` | `template.py:289` | 多进程 worker：按表达式类型分派到 by_expression/by_polars |
| `register_functions` | `utility.py:13` | 把自定义函数登记进 `EXPRESSION_FUNCTIONS` 全局表 |
| processor 族 | `processor.py` | process_drop_na/fill_na/cs_norm/replace_inf/ts_norm/drop_feature/cs_fill_na/robust_zscore/cs_rank_norm |
| `Alpha101` / `Alpha158` | `datasets/alpha_101.py:4` / `alpha_158.py:4` | 继承 AlphaDataset，构造时 add_feature 注册整套内置因子 |

## 3. 关键调用链

**调用链 1：因子表达式计算（多进程）**

1. 用户构造 `AlphaDataset(df, train_period, valid_period, test_period)`，链式 `add_feature("alpha1", "...表达式...")` + `set_label("...")`（`template.py:58,75`）。
2. 调用 `prepare_data(filters, max_workers)`（`template.py:90`）：把所有 feature_expressions + label 汇总为 args 列表（`template.py:98-101`）。
3. `get_context("spawn").Pool(processes=max_workers)` 建进程池，`pool.imap(calculate_feature, args)` 并行算每个因子（`template.py:108-116`）。
4. 单个 worker `calculate_feature((df, name, expr))`（`template.py:289`）：若是 polars Expr 走 `calculate_by_polars`，否则走 `calculate_by_expression`。
5. `calculate_by_expression`（`utility.py:203`）：局部 import ts/cs/ta/math 函数（`utility.py:206-236`），把 `EXPRESSION_FUNCTIONS` 合并进 locals（`utility.py:240`），逐列把列包成 DataProxy 注入命名空间（`utility.py:242-249`），最后 `eval(expression, {}, d)` 求值（`utility.py:252`）。
6. 主进程把结果列 `self.df.with_columns(results)` 拼回（`template.py:118`），再 join feature_results 预计算表（`template.py:124-126`），label 挪到最后一列并排序（`template.py:128-131`）。

**调用链 2：预处理管线**

1. `prepare_data` 产出 `raw_df`（`template.py:154`），`infer_df = learn_df = raw_df`（`template.py:156-157`）。
2. `process_data()`（`template.py:159`）：先顺序跑 `infer_processors` 生成 infer_df（`template.py:164-165`）；若 process_type=="append" 则 learn_df=infer_df（`template.py:168-169`），再顺序跑 `learn_processors`（`template.py:171-172`）。
3. 典型 processor 如 `process_cs_norm(df, "robust")`（`processor.py:34`）：按 datetime 横截面做 median/MAD 标准化并 clip(-3,3)；`process_replace_inf`（`processor.py:80`）把无穷替换为个股均值。

**调用链 3：三时段取数供模型训练/推理**

1. 模型 `fit(dataset)` 调 `dataset.fetch_learn(Segment.TRAIN)` / `VALID`（见 alpha-model 叶子）。
2. `fetch_learn(segment)`（`template.py:188`）按 `data_periods[segment]` 起止调 `query_by_time(self.learn_df, start, end)`（`template.py:274`）过滤并排序。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `train_period/valid_period/test_period` | 构造入参，三时段起止字符串/日期 | `template.py:29-32,44-48` |
| `process_type` | 默认 "append"：learn_df 先等于 infer_df 再跑 learn processors | `template.py:32,54,168` |
| `max_workers` | prepare_data 入参，进程池大小，None=CPU 默认 | `template.py:90,110` |
| `filters` | prepare_data 入参，成分股在指区间 `{vt_symbol: [(start,end)]}` | `template.py:90,136-150` |
| `process_cs_norm(method)` | "robust"（median/MAD）或 "zscore" | `processor.py:36,46,64` |
| `clip(-3,3)` | robust 标准化后截断 | `processor.py:61,181` |
| `quantiles=10` | alphalens 分组数 | `template.py:237,267` |

## 5. 错误与重试语义

- **表达式同时给 expression 和 result**：`add_feature` 显式 `raise ValueError("Only one of...")`（`template.py:67-68`）。
- **多进程异常**：`pool.imap` 中单个因子计算抛错会传播到主进程中断 prepare_data；无 per-factor 重试。
- **空数据**：`load_bar_df` 某标的文件缺失/空表则跳过（`lab.py:190-192,213-214`，本叶子消费侧）；alphalens 分析前 `drop_nulls`（`template.py:224`）。
- **除零保护**：Alpha158 特征里显式 `+1e-12` 防 high-low 为 0（`alpha_158.py`）。
- **无网络/IO 重试**：本叶子纯 CPU 计算，无重试语义。

## 6. 并发细节

- **多进程因子计算**：`prepare_data` 用 `get_context("spawn")` 而非 fork（`template.py:108-110`），规避 polars/numpy 在 fork 下的线程问题；`pool.imap` 保留计算顺序，外层 `tqdm` 进度条。
- **spawn 开销**：每个 worker 重新 import 模块（calculate_by_expression 内局部 import 函数库，`utility.py:206-236`），避免顶层 import 污染。
- **共享状态**：进程间无共享内存，df 通过参数序列化传给 worker；主进程收集结果列拼回。
- **GIL**：polars/numpy 计算释放 GIL，但多进程主要用于 CPU 密集因子计算并行。
- **无线程/锁**：单进程内无锁；processor 链式顺序执行。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `vnpy/alpha/dataset/template.py`：AlphaDataset 抽象基类、prepare_data/process_data/fetch_*、alphalens 接入
- `vnpy/alpha/dataset/utility.py`：DataProxy、表达式引擎、Segment、register_functions
- `vnpy/alpha/dataset/processor.py`：9 个预处理函数
- `vnpy/alpha/dataset/{cs,ts,ta,math}_function.py`：四类因子算子库
- `vnpy/alpha/dataset/datasets/alpha_{101,158}.py`：两套内置因子集

**Out-of-Scope（不在本仓库源码内）**
- **polars**：DataFrame/表达式/窗口函数/parquet 读写（不在本仓库源码内）。
- **TA-Lib**：ta_rsi/ta_atr 经 `talib.RSI/ATR` 计算（`ta_function.py:28,40`，不在本仓库源码内）。
- **alphalens-reloaded**：因子绩效 tear sheet（`template.py:11-12`，不在本仓库源码内）。
- **pandas**：alphalens 对接时 to_pandas 转换（不在本仓库源码内）。
- **BarData/行情来源**：见 trader-core 域；本叶子只消费已加载的 polars df。
- **模型训练**：见 alpha-model 叶子；本叶子产出特征矩阵 X 与标签 y。

## 8. 与相邻子系统交互

- **上游 → 本叶子**：AlphaLab.load_bar_df 产出长表 polars df（datetime/vt_symbol/OHLCV/vwap），用户构造 `AlphaDataset(df, ...)`；Alpha101/Alpha158 是预置好因子表达式的子类。
- **本叶子 → 下游**：
  - `prepare_data` + `process_data` 后，模型经 `fetch_learn(Segment.TRAIN/VALID)` 取训练矩阵、`fetch_infer(Segment.TEST)` 取推理矩阵（见 alpha-model 叶子）。
  - `show_feature_performance`/`show_signal_performance` 调 alphalens 出分析图。
- **扩展方向**：用户经 `register_functions([my_func])` 注册自定义算子到 `EXPRESSION_FUNCTIONS`，即可在表达式字符串里调用——这是本叶子的插件扩展缝。

## 9. 语言专项适配口径（纯 Python：能力缝 + 注册表 + 插件体系）

- **能力缝**：核心能力缝是"**表达式 DSL + 函数算子库 + 预处理链**"。`AlphaDataset` 用 `add_feature(name, expression)` 注册字符串/Expr，运行时 `calculate_by_expression` 把列包成 DataProxy 注入命名空间后 `eval`——这是 Qlib 风格的表达式因子引擎。
- **注册表**：
  - `EXPRESSION_FUNCTIONS: dict[str, Callable]`（`utility.py:10`）是显式函数注册表，`register_functions()`（`utility.py:13`）按 `func.__name__` 登记；计算时 `d.update(EXPRESSION_FUNCTIONS)`（`utility.py:240`）注入。
  - ts/cs/ta/math 四套算子在 `calculate_by_expression` 内局部 import 后注入 locals（`utility.py:206-236`）——import 时副作用即为注册。
- **运算符重载（DSL 核心）**：DataProxy 重载 `__add__/__sub__/.../__gt__/__eq__` 等（`utility.py:56-200`），让表达式字符串 `(close-open)/open` 能在 eval 里逐元素运算并保持 datetime/vt_symbol 主键。
- **插件链**：`infer_processors`/`learn_processors` 是两个可调用对象列表（`template.py:55-56`），`add_processor(task, fn)` 追加（`template.py:81`），`process_data` 顺序链式调用——典型的"可插拔钩子链"。
- **多进程工厂**：`prepare_data` 用 `multiprocessing.Pool(spawn)` 并行算因子（`template.py:108-118`），worker 函数 `calculate_feature` 按表达式类型分派到 by_expression/by_polars。
- **双轨实现**：同一因子既可传字符串表达式（eval 引擎）也可传 polars Expr（直接 select）——双轨 DSL。
- **图型侧重**：本叶子是"原始 df → 表达式并行计算 → 预处理链 → 三时段矩阵"的特征管线，补 dataflow 图；architecture 图表达 DataProxy/函数库/processor 插件体系。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 因子特征工程架构图 | [alpha-dataset-architecture.html](alpha-dataset-architecture.html) | architecture | showcase |
| 因子计算数据流图 | [alpha-dataset-dataflow.html](alpha-dataset-dataflow.html) | dataflow | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 本叶子未生成 sequence 图：因子计算是 map 式并行（每因子独立），无明确多方消息时序；未生成 lifecycle 图：无单实体状态机；未生成 workflow 图：预处理链是固定顺序而非带泳道流程。
