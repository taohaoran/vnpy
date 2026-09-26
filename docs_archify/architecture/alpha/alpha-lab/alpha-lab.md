# Alpha 研究实验室（alpha-lab）

> 本文是 `alpha` 域下的叶子子系统文档。域级总览见 `../alpha.md`。
>
> 源码基准：`vnpy` 4.4.0（master），commit `fa5206fe`；纯 Python + polars，主语言口径见本文第 9 节。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| K线数据落盘 | 把 BarData 列表按日/分钟存为 parquet，已存在则合并去重排序 | `vnpy/alpha/lab.py:51`（`save_bar_data`） |
| K线数据加载 | 按 vt_symbol+区间读 parquet 还原为 BarData 列表 | `vnpy/alpha/lab.py:96`（`load_bar_data`） |
| 多标的 DataFrame 加载 | 批量读多标的 parquet，价格归一化（除以首收盘）、停牌置 NaN、拼接长表 | `vnpy/alpha/lab.py:156`（`load_bar_df`） |
| 指数成分管理 | 用 shelve 存/读每日成分股列表，lru_cache 缓存 | `vnpy/alpha/lab.py:245,257` |
| 成分过滤区间推导 | 由每日成分表反推每只股票的连续在指数区间（供因子计算只取在指期） | `vnpy/alpha/lab.py:301`（`load_component_filters`） |
| 合约配置管理 | JSON 存 long_rate/short_rate/size/pricetick | `vnpy/alpha/lab.py:349,379` |
| 数据集持久化 | pickle 存/取/列/删 AlphaDataset 对象 | `vnpy/alpha/lab.py:389,396,407,417` |
| 模型持久化 | pickle 存/取/列/删 AlphaModel 对象 | `vnpy/alpha/lab.py:421,428,439,449` |
| 信号持久化 | parquet 存/取/列/删预测信号 DataFrame | `vnpy/alpha/lab.py:453,459,468,478` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `AlphaLab` | `vnpy/alpha/lab.py:20` | 研究统筹类：管理本地数据目录与数据集/模型/信号的全生命周期 |
| `save_bar_data` / `load_bar_data` / `load_bar_df` | `lab.py:51/96/156` | K线 parquet 读写与归一化 |
| `save_component_data` / `load_component_data` / `load_component_symbols` / `load_component_filters` | `lab.py:245/257/281/301` | 指数成分 shelve 存储与区间推导 |
| `add_contract_setting` / `load_contract_setttings` | `lab.py:349/379` | 合约交易参数 JSON |
| `save_dataset`/`load_dataset`/`remove_dataset`/`list_all_datasets` | `lab.py:389/396/407/417` | 数据集 pickle CRUD |
| `save_model`/`load_model`/`remove_model`/`list_all_models` | `lab.py:421/428/439/449` | 模型 pickle CRUD |
| `save_signal`/`load_signal`/`remove_signal`/`list_all_signals` | `lab.py:453/459/468/478` | 信号 parquet CRUD |

## 3. 关键调用链

**调用链 1：历史 K 线入库**

1. 外部调用 `save_bar_data(bars)`（`lab.py:51`），按 `bar.interval` 选 daily/minute 目录，文件名为 `{vt_symbol}.parquet`（`lab.py:59-62`）。
2. 把每根 BarData 转为 dict（tzinfo 置 None，OHLCV+turnover+open_interest）构造 `pl.DataFrame`（`lab.py:67-81`）。
3. 文件已存在则 `pl.read_parquet` 读出旧表，`pl.concat` 合并后按 datetime 去重排序（`lab.py:84-91`），最后 `write_parquet`（`lab.py:94`）。

**调用链 2：多标的研究 DataFrame 加载**

1. `load_bar_df(vt_symbols, interval, start, end, extended_days)`（`lab.py:156`）：start 向前扩 extended_days（用于因子预热），end 向后扩 extended_days//10（`lab.py:172-173`）。
2. 逐标的读 parquet、按区间过滤（`lab.py:187-198`），派生 vwap=turnover/volume（`lab.py:209`）。
3. 价格归一化：open/high/low/close 全部除以首日 close（`lab.py:217-224`）。
4. 停牌识别：数值列横向求和为 0 则置 NaN（`lab.py:227-233`），加 vt_symbol 列（`lab.py:236`），最后 `pl.concat` 长表返回（`lab.py:242`）。

**调用链 3：成分股在指区间推导**

1. `load_component_filters(index_symbol, start, end)`（`lab.py:301`）先取每日成分字典（`lab.py:308-312`）。
2. 遍历每只股票在所有交易日上的出现连续性，把连续在指段记录为 `(period_start, period_end)` 区间列表（`lab.py:326-345`），供因子计算只在该股票在指数期间取样。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `lab_path` | 构造入参，研究根目录；下建 daily/minute/component/dataset/model/signal 子目录 | `lab.py:26-49` |
| `extended_days` | load_bar_df 入参，向前扩展天数做因子预热 | `lab.py:162,172` |
| 支持的 interval | 仅 DAILY / MINUTE，其余 log.error 并返回 | `lab.py:59-65,112-118,176-182` |
| 合约配置 | long_rate/short_rate/size/pricetick，存 contract.json | `lab.py:364-369` |

## 5. 错误与重试语义

- **文件不存在**：load 类方法在文件缺失时 `logger.error` 后返回空列表/None（`lab.py:122-124,190-192,399-401,431-433,462-464`），不抛异常。
- **不支持的周期**：error 日志后返回空（`lab.py:63-65,117-118,181-182`）。
- **空 bars 直接返回**：`save_bar_data` 空列表 no-op（`lab.py:53-54`）。
- **无重试**：本叶子是本地文件 IO，无网络/重试语义；parquet/pickle 写失败会直接抛异常。

## 6. 并发细节

- **单线程**：本叶子无工作线程，全部同步文件读写；`load_component_data` 用 `@lru_cache` 缓存成分查询结果（`lab.py:256`）。
- **多进程边界**：`AlphaDataset.prepare_data` 内部用 `multiprocessing.Pool` 并行算因子（见 alpha-dataset 叶子），但 AlphaLab 本身不参与并行。
- **无锁**：本地文件操作假设单用户/单进程研究，无并发写锁。
- **取消传播**：无取消概念；文件 IO 为同步阻塞。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `vnpy/alpha/lab.py`：研究目录管理、K线 parquet 读写、成分 shelve、合约 JSON、数据集/模型/信号 pickle 与 parquet CRUD

**Out-of-Scope（不在本仓库源码内）**
- **polars**：DataFrame/parquet 读写、concat/join/filter/表达式计算由 polars 提供（不在本仓库源码内）。
- **shelve / pickle / json / pathlib**：Python 标准库持久化机制（不在本仓库源码内）。
- **BarData**：K线数据结构定义在 `vnpy/trader/object.py`（trader-core 域）。
- **因子计算 / 模型训练 / 回测**：分别见 alpha-dataset / alpha-model / alpha-strategy 叶子，本叶子只负责存储编排，不计算因子、不训练、不回测。
- **行情数据源**：历史 K 线从哪来（数据服务/网关）由 trader 域负责，本叶子只消费已下载到本地的 parquet。

## 8. 与相邻子系统交互

- **上游 → 本叶子**：用户/脚本先 `save_bar_data` 把行情落盘；`BacktestingEngine.set_parameters` 经 `lab.load_contract_setttings()` 读合约配置（`lab.py:92`）；`BacktestingEngine.load_data` 经 `lab.load_bar_data()` 读历史（`backtesting.py:130`）。
- **本叶子 → 下游**：
  - `AlphaDataset` 经 `save_dataset`/`load_dataset` 存取（lab.py:389/396）。
  - `AlphaModel` 经 `save_model`/`load_model` 存取（lab.py:421/428）。
  - 预测信号经 `save_signal`/`load_signal` 存 parquet（lab.py:453/459），回测引擎读 `signal_df`。
- **定位**：AlphaLab 是 alpha 域的"仓库/门面"，把数据集、模型、信号、K线、成分、合约配置统一管理在一个目录树下，供 dataset/model/strategy 三个叶子读写。

## 9. 语言专项适配口径（纯 Python：能力缝 + 注册表 + 插件体系）

- **能力缝**：AlphaLab 本身不是抽象基类体系，而是"**存储门面/仓储模式**"——把数据集（AlphaDataset）、模型（AlphaModel）、信号（DataFrame）三类研究产物的持久化统一为 `save_X/load_X/remove_X/list_all_X` 四件套 CRUD，按文件后缀（.pkl/.parquet）区分对象类型。
- **序列化双轨**：对象类（AlphaDataset/AlphaModel）用 pickle 整对象序列化（`.pkl`），表格类（信号）用 parquet 列式存储（`.parquet`）——这是典型的"对象 vs 表格"双轨持久化取舍：pickle 保留 Python 对象状态（含已训练模型权重、已配置的因子表达式），parquet 便于列式分析复用。
- **注册表**：本叶子无字符串→类路由；`@lru_cache`（`lab.py:256`）是成分查询的进程内缓存工厂。
- **配置面**：研究产物路径全部由构造入参 `lab_path` 派生，无 dataclass；合约配置走 JSON 文件。
- **可选依赖**：polars 是 alpha extras 依赖（不在本仓库源码内）；未装 polars 则整个 alpha 域不可用。
- **图型侧重**：本叶子是"数据→因子→模型→信号→回测"研究流水线的总仓储，补 dataflow 图表达研究产物在目录树中的流转；architecture 图表达与三个子域的仓储关系。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| AlphaLab 架构图 | [alpha-lab-architecture.html](alpha-lab-architecture.html) | architecture | standard |
| Alpha 研究产物数据流图 | [alpha-lab-dataflow.html](alpha-lab-dataflow.html) | dataflow | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 本叶子未生成 sequence 图：本叶子是同步本地文件读写，无多参与方消息交互；未生成 lifecycle 图：无多阶段状态机；未生成 workflow 图：无带泳道的审批流程。
