# 数据库接口（database-interface）

> 本文是 `trader-service` 域下的叶子子系统文档。域级总览见 `../trader-service.md`。
> 本文只展开 vnpy 的行情/账户数据落库抽象层与驱动注册路由，不重复展开 `datafeed.py`（历史行情服务）的职责（见 `../datafeed-interface/datafeed-interface.md`）。
>
> 源码基准：vnpy 4.4.0，commit `fa5206fe`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 数据库时区归一 | `DB_TZ` 由 `SETTINGS["database.timezone"]` 构造；`convert_tz` 把任意 datetime 转到 DB_TZ 后剥离 tzinfo（落库为 naive datetime） | `vnpy/trader/database.py:14`、`database.py:17` |
| K 线数据概览结构 | `BarOverview` dataclass：symbol/exchange/interval/count/start/end | `database.py:25` |
| Tick 数据概览结构 | `TickOverview` dataclass：symbol/exchange/count/start/end | `database.py:39` |
| 数据库抽象基类 | `BaseDatabase(ABC)` 定义 8 个抽象方法：保存/加载/删除 bar 与 tick、查询概览 | `database.py:52` |
| 数据库单例工厂 | `get_database()` 懒加载单例：按 `SETTINGS["database.name"]` 动态 `import_module("vnpy_<name>")` 并实例化 `Database()` | `database.py:139` |
| 驱动缺失兜底 | 目标驱动 `ModuleNotFoundError` 时回退到 `vnpy_sqlite` 并打印中文提示 | `database.py:153` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `convert_tz(dt)` | `database.py:17` | 时区转换工具：`dt.astimezone(DB_TZ).replace(tzinfo=None)` |
| `BarOverview` | `database.py:25` | dataclass，描述库内某 (symbol, exchange, interval) bar 数据的数量与起止时间 |
| `TickOverview` | `database.py:39` | dataclass，描述库内某 (symbol, exchange) tick 数据的数量与起止时间 |
| `BaseDatabase(ABC)` | `database.py:52` | 数据库契约基类；8 个 `@abstractmethod`：`save_bar_data`/`save_tick_data`/`load_bar_data`/`load_tick_data`/`delete_bar_data`/`delete_tick_data`/`get_bar_overview`/`get_tick_overview` |
| `get_database()` | `database.py:139` | 模块级单例工厂；维护全局变量 `database`，首次调用才真正 import 驱动并构造 |

> 接口定义位置说明：抽象基类 `BaseDatabase` 定义在**消费方**（本仓库 `vnpy/trader/database.py`），而具体实现类 `Database` 位于**外部扩展包** `vnpy_<dbname>`（如 `vnpy_sqlite`/`vnpy_mysql`/`vnpy_postgresql`），不在本仓库源码内——这是典型的"接口在框架、实现在插件"能力缝。

## 3. 关键调用链

**调用链一：首次 `get_database()` 懒加载解析（`database.py:139`）**
1. `database.py:143`：若全局 `database` 已存在，直接返回（单例短路）。
2. `database.py:147`–`148`：读 `SETTINGS["database.name"]`（默认 `"sqlite"`），拼出模块名 `vnpy_sqlite`。
3. `database.py:151`–`155`：`import_module("vnpy_sqlite")`；若抛 `ModuleNotFoundError`，打印"找不到数据库驱动…使用默认的SQLite数据库"，再 `import_module("vnpy_sqlite")` 兜底（注意：兜底仍可能因未安装而抛错）。
4. `database.py:158`：`database = module.Database()` 实例化驱动对象并缓存。

**调用链二：保存 bar 数据到库（抽象契约）**
1. 调用方（如数据记录器、`MainEngine`）持有 `get_database()` 返回的 `BaseDatabase` 实例。
2. 调用 `save_bar_data(bars, stream=False)`（`database.py:58`）——具体 SQL/ORM 实现由 `vnpy_<dbname>` 插件完成。
3. 加载侧对应 `load_bar_data(symbol, exchange, interval, start, end)`（`database.py:72`）按区间返回 `list[BarData]`。

**调用链三：概览查询**
1. `get_bar_overview()`（`database.py:122`）返回 `list[BarOverview]`，供 UI/数据管理工具展示库内已有数据范围，决定补数区间。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `database.timezone` | `get_localzone_name()` 本机时区，构造 `DB_TZ` | `setting.py:31`、`database.py:14` |
| `database.name` | `"sqlite"`，决定 import `vnpy_<name>` | `setting.py:32` |
| `database.database` | `"database.db"`（SQLite 文件路径） | `setting.py:33` |
| `database.host/port/user/password` | 空 / 0 / 空 / 空，供 C/S 数据库使用 | `setting.py:34` |

## 5. 错误与重试语义

- 目标数据库驱动未安装时，`import_module` 抛 `ModuleNotFoundError`（`database.py:153`），框架**不抛异常终止**，而是打印中文提示后回退 `vnpy_sqlite`；若连 `vnpy_sqlite` 都未安装，则异常向上传播。
- `save_bar_data`/`load_bar_data` 的 `stream` 参数（`database.py:58`）为流式保存开关，具体失败重试语义由各驱动插件实现，本抽象层不规定。
- `convert_tz` 无失败路径；DB_TZ 构造失败会在 import 期暴露。

## 6. 并发细节

- `get_database()` 单例**非线程安全**：全局变量 `database` 的惰性初始化（`database.py:142`–`158`）无锁；vnpy 约定在主引擎启动主线程中首次调用，后续在事件回调里复用已缓存实例。
- 抽象方法本身不含并发控制；多线程写库的串行化/连接池由驱动插件负责。
- 无超时/取消传播定义在抽象层。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `vnpy/trader/database.py`：`BaseDatabase` ABC、`BarOverview`/`TickOverview`、`convert_tz`、`get_database()` 注册路由。

**Out-of-Scope（不在本仓库源码内）**
- 具体数据库驱动包 `vnpy_sqlite`/`vnpy_mysql`/`vnpy_postgresql`/`vnpy_mongodb` 等（含 `Database` 类的真正 SQL 实现）——不在本仓库源码内。
- 物理数据库产品（SQLite/MySQL/PostgreSQL/MongoDB 服务端）——不在本仓库源码内。
- 数据服务驱动（`vnpy_rqdata` 等）属 `datafeed-interface` 叶子，不在本叶子。

## 8. 与相邻子系统交互

- **上游调用方**：`MainEngine`/`OmsEngine`、数据记录示例（`examples/data_recorder`）、回测模块通过 `get_database()` 拿到 `BaseDatabase` 句柄做历史 bar/tick 读写。
- **下游依赖**：`database.py` 依赖 `object.py`（`BarData`/`TickData`）、`constant.py`（`Interval`/`Exchange`）、`setting.py`（`SETTINGS`）、`utility.py`（`ZoneInfo`）。
- **路由方向**：`SETTINGS["database.name"]` 字符串 → `import_module("vnpy_<name>")` → `module.Database()` → `BaseDatabase` 句柄。

## 9. 语言专项适配口径（纯 Python：能力缝 + 注册表 + 插件体系）

- **能力缝（抽象基类契约）**：`BaseDatabase(ABC)` 是本域最典型的能力缝——接口定义在框架消费方（`database.py:52`），8 个抽象方法定义"保存/加载/删除/概览"契约，具体实现分散在多个独立发行的 `vnpy_*` 扩展包中。
- **注册表/工厂（字符串路由）**：`get_database()`（`database.py:139`）是字符串→实现类的懒工厂：`SETTINGS["database.name"]`（如 `"sqlite"`）→ `import_module("vnpy_sqlite")` → `module.Database()`。注册时机是**首次调用时动态 import**（非 import 时副作用），单例缓存于模块全局变量 `database`。
- **可选依赖与降级**：驱动未安装时回退 SQLite 并打印提示，属"可选依赖优雅降级"——框架不强制用户安装所有数据库驱动。
- **配置驱动静态分派**：运行时用哪套数据库后端，完全由 `SETTINGS["database.name"]` 这一配置项决定，代码中无 `if db == "mysql"` 硬编码分支——这是纯 Python 框架配置驱动多后端矩阵的标准做法。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| database-interface 架构图 | `database-interface-architecture.html` | architecture | showcase |
| get_database 懒加载时序图 | `database-interface-sequence.html` | sequence | showcase |

JSON IR 位于 `json/` 目录。
本叶子未生成 dataflow / lifecycle / workflow 图：本叶子是"接口契约 + 工厂路由"，数据落库的真实管道在各 `vnpy_*` 驱动插件内部（不在本仓库），本仓库内无独立数据变换管道可画；`get_database` 是一次性初始化（无反复状态变迁），不构成状态机。
