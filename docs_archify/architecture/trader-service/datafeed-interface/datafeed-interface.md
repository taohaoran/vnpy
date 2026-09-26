# 历史数据服务接口（datafeed-interface）

> 本文是 `trader-service` 域下的叶子子系统文档。域级总览见 `../trader-service.md`。
> 本文只展开 vnpy 的历史行情数据服务抽象层与驱动注册路由，不重复展开 `database.py`（本地数据落库）的职责（见 `../database-interface/database-interface.md`）。
>
> 源码基准：vnpy 4.4.0，commit `fa5206fe`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 数据服务抽象基类 | `BaseDatafeed`：定义 `init`/`query_bar_history`/`query_tick_history` 三个钩子，基类提供"未配置"默认实现 | `vnpy/trader/datafeed.py:10` |
| 初始化连接 | `init(output)` 默认返回 `False`，由各数据服务驱动覆写为真实登录/鉴权 | `datafeed.py:15` |
| 历史 K 线查询 | `query_bar_history(req, output)` 默认输出"未配置数据服务"并返回空列表 | `datafeed.py:21` |
| 历史 Tick 查询 | `query_tick_history(req, output)` 同上 | `datafeed.py:28` |
| 数据服务单例工厂 | `get_datafeed()` 懒加载单例：按 `SETTINGS["datafeed.name"]` 动态 import 并实例化 `Datafeed()` | `datafeed.py:39` |
| 未配置/驱动缺失降级 | 名为空→直接用 `BaseDatafeed()` 空实现；驱动 `ModuleNotFoundError`→回退 `BaseDatafeed()` 并提示 pip install | `datafeed.py:49`、`datafeed.py:63` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `BaseDatafeed` | `datafeed.py:10` | 数据服务契约基类（非 ABC，用默认实现做软契约）；三个方法均接收 `output: Callable=print` 用于向用户回显进度/错误 |
| `get_datafeed()` | `datafeed.py:39` | 模块级单例工厂，维护全局 `datafeed`；按配置字符串路由到外部 `vnpy_<name>` 包 |
| `HistoryRequest` | 引自 `object.py` | 历史查询请求（symbol/exchange/interval/start/end），由调用方构造后传入 `query_bar_history` |

> 与 `BaseDatabase`（ABC 硬抽象）不同，`BaseDatafeed` 是**软契约**：方法带默认实现而非 `@abstractmethod`，未配置数据服务时返回空结果而不是抛错——体现"数据服务是可选增强"的定位。

## 3. 关键调用链

**调用链一：`get_datafeed()` 单例解析（`datafeed.py:39`）**
1. `datafeed.py:43`：全局 `datafeed` 已存在则直接返回。
2. `datafeed.py:47`：读 `SETTINGS["datafeed.name"]`。
3. `datafeed.py:49`–`52`：若名称为空，直接 `datafeed = BaseDatafeed()`，打印"没有配置要使用的数据服务…"。
4. `datafeed.py:54`–`61`：否则拼 `vnpy_<name>`，`import_module` 后 `module.Datafeed()` 实例化。
5. `datafeed.py:63`–`66`：若 import 抛 `ModuleNotFoundError`，回退 `BaseDatafeed()` 并打印"无法加载数据服务模块，请运行 pip install vnpy_<name>"。

**调用链二：历史 K 线查询（默认空实现路径）**
1. 调用方拿到 `get_datafeed()` 句柄后调 `init(output)` 建立连接。
2. 构造 `HistoryRequest`，调 `query_bar_history(req, output)`。
3. 若未配置，走基类默认实现（`datafeed.py:25`）：`output("查询K线数据失败：没有正确配置数据服务")` 并返回 `[]`。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `datafeed.name` | 空字符串；为空时使用 `BaseDatafeed` 空实现 | `setting.py:27` |
| `datafeed.username` / `datafeed.password` | 空，供数据服务鉴权 | `setting.py:28` |

## 5. 错误与重试语义

- 未配置数据服务（`datafeed.name` 为空）：不报错，返回空 `BaseDatafeed` 实例，查询时回显提示并返回空列表（`datafeed.py:50`、`datafeed.py:25`）。
- 驱动未安装（`ModuleNotFoundError`）：不抛异常，回退空实现并提示用户 `pip install vnpy_<name>`（`datafeed.py:63`）。
- 真实数据服务的网络失败/重试由各 `vnpy_<name>` 驱动自行处理，本抽象层不规定。
- `output` 回调（默认 `print`）是统一的进度/错误回显通道。

## 6. 并发细节

- `get_datafeed()` 单例初始化无锁（`datafeed.py:42`–`61`），约定主线程首次调用后复用。
- `query_bar_history`/`query_tick_history` 在本抽象层为同步调用；实际驱动可能内部用 session/连接池。
- 无超时/取消传播定义在抽象层。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `vnpy/trader/datafeed.py`：`BaseDatafeed` 软契约基类、`get_datafeed()` 注册路由。

**Out-of-Scope（不在本仓库源码内）**
- 具体数据服务驱动包 `vnpy_rqdata`/`vnpy_tushare`/`vnpy_ricequant` 等（含 `Datafeed` 类的真实 HTTP/WebSocket 实现）——不在本仓库源码内。
- 远端行情服务商服务器（米筐/TuShare/RiceQuant 等）——不在本仓库源码内。
- 本地落库逻辑属 `database-interface` 叶子。

## 8. 与相邻子系统交互

- **上游调用方**：`MainEngine`、数据下载示例（`examples/download_bars`）、回测模块通过 `get_datafeed().query_bar_history(req)` 拉取历史数据。
- **下游依赖**：`datafeed.py` 依赖 `object.py`（`HistoryRequest`/`BarData`/`TickData`）、`setting.py`（`SETTINGS`）。
- **典型用法**：`get_datafeed()` → `init()` → `query_bar_history(HistoryRequest(...))` → 结果转存 `get_database()` 入库。

## 9. 语言专项适配口径（纯 Python：能力缝 + 注册表 + 插件体系）

- **能力缝（软契约基类）**：与 `BaseDatabase` 的 ABC 硬抽象不同，`BaseDatafeed`（`datafeed.py:10`）采用"默认实现 + 可选覆写"的软契约模式——未配置时返回空结果而非强制子类实现。这体现纯 Python 框架对"可选增强能力缝"的典型处理。
- **注册表/工厂（字符串路由 + 双重降级）**：`get_datafeed()`（`datafeed.py:39`）是字符串→实现懒工厂：`datafeed.name` → `import_module("vnpy_<name>")` → `module.Datafeed()`。相比 `database` 工厂只兜底 SQLite，`datafeed` 有**两级降级**：名为空→空基类；驱动缺失→空基类+安装提示。
- **可注入回调**：所有查询方法接收 `output: Callable`（默认 `print`），允许 UI 把进度输出重定向到监控面板——这是纯 Python 框架常见的"依赖注入回调"能力缝。
- **配置驱动分派**：用哪个数据服务完全由 `SETTINGS["datafeed.name"]` 决定，代码无硬编码分支。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| datafeed-interface 架构图 | `datafeed-interface-architecture.html` | architecture | showcase |
| get_datafeed 解析时序图 | `datafeed-interface-sequence.html` | sequence | showcase |

JSON IR 位于 `json/` 目录。
本叶子未生成 dataflow / lifecycle / workflow 图：本叶子是"软契约 + 工厂路由"，真实历史数据下载管道在各 `vnpy_*` 驱动内（不在本仓库）；`get_datafeed` 为一次性初始化，无反复状态机。
