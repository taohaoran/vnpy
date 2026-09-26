# 配置与工具（config-utility）

> 本文是 `trader-service` 域下的叶子子系统文档。域级总览见 `../trader-service.md`。
> 本文只展开 vnpy 平台的通用工具函数、全局配置加载与日志初始化，不重复展开 `database.py`/`datafeed.py`/`wechat.py`/`optimize.py` 的职责（分别见各自叶子）。
>
> 源码基准：vnpy 4.4.0，commit `fa5206fe`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| vt_symbol 编解码 | `extract_vt_symbol` 把 `symbol.exchange` 字符串拆回 `(symbol, Exchange)`；`generate_vt_symbol` 反向拼接 | `vnpy/trader/utility.py:23`、`utility.py:31` |
| 运行目录定位 | `_get_trader_dir` 优先用当前目录下 `.vntrader`，否则用用户 HOME 下 `.vntrader`；模块加载时即算出 `TRADER_DIR`/`TEMP_DIR` 并把 TRADER_DIR 加入 `sys.path` | `utility.py:38`、`utility.py:61` |
| 临时文件/目录/图标路径 | `get_file_path`、`get_folder_path`（不存在则自动建目录）、`get_icon_path` | `utility.py:65`、`utility.py:72`、`utility.py:82` |
| JSON 配置读写 | `load_json`（文件不存在则落盘空 dict）、`save_json`（UTF-8、缩进 4、保留非 ASCII） | `utility.py:91`、`utility.py:106` |
| 价格取整工具 | `round_to`/`floor_to`/`ceil_to` 按最小价格变动单位 tick 取整（基于 `Decimal` 避免浮点误差）；`get_digits` 解析小数位数 | `utility.py:120`、`utility.py:130`、`utility.py:140`、`utility.py:150` |
| BarGenerator K 线合成 | 由 Tick 合成 1 分钟 K 线，再由 1 分钟 K 线合成 x 分钟 / x 小时 / 日 K 线；通过回调 `on_bar`/`on_window_bar` 推送 | `utility.py:166` |
| ArrayManager 序列容器 | 定长滑动窗口（默认 100）维护 OHLCV/持仓量 numpy 数组，封装 40+ 个 TA-Lib 技术指标 | `utility.py:488` |
| virtual 装饰器 | 标记基类中可被子类覆写的方法（配合 ABC 体系的软扩展点约定） | `utility.py:1275` |
| 全局默认配置 | `SETTINGS` 字典：字体、日志、邮件、数据服务、数据库连接默认值 | `vnpy/trader/setting.py:11` |
| 配置文件加载 | 模块加载时从 `vt_setting.json` 读入并 `SETTINGS.update(...)` 覆盖默认值 | `setting.py:42` |
| loguru 日志初始化 | 按 `SETTINGS` 配置控制台/文件输出，格式含 gateway_name 标签，文件名按日期 `vt_YYYYMMDD.log` | `vnpy/trader/logger.py:23`、`logger.py:44`、`logger.py:49` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `BarGenerator` | `utility.py:166` | K 线合成状态机：内部维护 `self.bar`（当前 1 分钟 bar）、`window_bar`/`hour_bar`/`daily_bar`（各级窗口 bar），构造时注入 `on_bar` 与 `on_window_bar` 两个回调 |
| `BarGenerator.update_tick` | `utility.py:204` | Tick 入口：过滤零价、跨分钟时结束并推送旧分钟 bar、累计 OHLCV/成交量增量 |
| `BarGenerator.update_bar` | `utility.py:262` | 1 分钟 bar 入口：按 `interval` 分派到分钟窗/小时窗/日窗三条合成路径 |
| `BarGenerator.update_bar_minute_window` | `utility.py:273` | x 分钟窗合成，完成判定 `(minute+1) % window == 0` |
| `BarGenerator.update_bar_hour_window` / `on_hour_bar` | `utility.py:311`、`utility.py:390` | 小时窗合成；`on_hour_bar` 再把整点 bar 聚合成 x 小时窗 |
| `BarGenerator.update_bar_daily_window` | `utility.py:430` | 日 K 合成，按 `daily_end` 收盘时间触发 |
| `ArrayManager` | `utility.py:488` | 滑动窗口序列容器；`update_bar` 用 numpy 数组整体左移 `[:-1]=[1:]` 再写末位 |
| `ArrayManager.sma/ema/macd/boll...` | `utility.py:586` 起 | 指标族：统一委托 `talib.*`，`array=False` 返回末位标量，`array=True` 返回整条 ndarray |
| `SETTINGS` | `setting.py:11` | 全局单例配置 dict，被 database/datafeed/logger/UI 等几乎所有模块读取 |
| `logger`（loguru） | `logger.py:6` | 模块级单例 logger，`configure(extra={"gateway_name":"Logger"})` 注入默认网关名标签 |

## 3. 关键调用链

**调用链一：Tick → 1 分钟 K 线合成（`BarGenerator.update_tick`）**
1. `utility.py:211` 先过滤 `tick.last_price` 为 0 的无效行情，直接返回。
2. `utility.py:214`–`225`：若 `self.bar` 为空，或新 tick 与当前 bar 的分钟/小时不同，则把旧 bar 的 datetime 截断到秒级 0 并调用 `self.on_bar(self.bar)` 推送，置 `new_minute=True`。
3. `utility.py:227`–`239`：新分钟到来时新建 `BarData`，开高低收均取 `tick.last_price`。
4. `utility.py:240`–`251`：同一分钟内更新 high/low/close/open_interest（high/low 还会参考 tick 快照价极值）。
5. `utility.py:253`–`258`：用相邻两 tick 的 volume/turnover 差值（`max(增量,0)`）累加，避免重计量。

**调用链二：1 分钟 bar → x 分钟窗口 bar（`update_bar` → `update_bar_minute_window`）**
1. `utility.py:266`–`271`：`update_bar` 按 `self.interval`（MINUTE/HOUR/DAILY）分派到三条合成路径。
2. `utility.py:276`–`302`：未初始化时以当前分钟对齐新建 `window_bar`，否则累计 high/low/close/volume/turnover。
3. `utility.py:305`–`309`：当 `(bar.datetime.minute + 1) % self.window == 0` 时认为窗口结束，回调 `on_window_bar` 并清空 `window_bar`（要求 window 能整除 60）。

**调用链三：ArrayManager 指标计算**
1. `utility.py:513`–`531`：每来一根 bar，count++，count≥size 时置 `inited=True`；7 个数组整体左移一位后把新值写入末位。
2. `utility.py:586`–`595`：以 `sma` 为例，直接 `talib.SMA(self.close, n)` 算出整条结果数组；`array=False` 时返回 `result_array[-1]` 标量，否则返回整条 ndarray。其余 40+ 指标同构（委托 TA-Lib）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `font.family` / `font.size` | "微软雅黑" / 12，供 UI 字体初始化 | `setting.py:12` |
| `log.active` / `log.level` / `log.console` / `log.file` | True / INFO / True / True | `setting.py:15` |
| `email.*` | smtp.qq.com:465，账号密码为空 | `setting.py:20` |
| `datafeed.name/username/password` | 空（未配置数据服务） | `setting.py:27` |
| `database.timezone` | `tzlocal.get_localzone_name()` 本机时区 | `setting.py:31` |
| `database.name/database` | "sqlite" / "database.db" | `setting.py:32` |
| `SETTING_FILENAME` | `"vt_setting.json"`，模块加载即 `SETTINGS.update(load_json(...))` | `setting.py:42` |
| `BarGenerator(window, daily_end)` | window 默认 0；合成日 K 时 `daily_end` 必传，否则 `RuntimeError` | `utility.py:176`、`utility.py:201` |
| `ArrayManager(size)` | 默认 100 | `utility.py:495` |

## 5. 错误与重试语义

- `BarGenerator` 对零价 tick 直接静默丢弃（`utility.py:211`），不抛错。
- 合成日 K 时若未传 `daily_end`，构造期即抛 `RuntimeError("合成日K线必须传入每日收盘时间")`（`utility.py:201`），属于配置期硬校验。
- `load_json` 在文件不存在时不报错，而是落盘空 dict 后返回 `{}`（`utility.py:102`），保证首次运行可启动。
- 价格取整用 `Decimal(str(value))`（`utility.py:124`）规避二进制浮点误差，无重试逻辑。
- 日志模块无失败重试；loguru sink 写入失败由 loguru 内部处理。

## 6. 并发细节

- 本叶子为**纯同步、单线程模型**：`BarGenerator`/`ArrayManager` 都在事件引擎分发到策略/监控的回调线程里被顺序调用，内部无锁、无线程。
- `BarGenerator` 内部状态（`self.bar`/`window_bar`/`last_tick`）依赖调用方保证时序不并发写入；vnpy 的事件引擎在单线程队列里分发事件，天然串行。
- `logger` 模块在 import 时副作用执行 `logger.remove()` + `logger.add(...)`（`logger.py:40`），属于模块级一次性初始化，非运行期并发点。
- 无超时/取消传播；`virtual` 装饰器（`utility.py:1275`）是无操作装饰器，仅作文档标记。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `vnpy/trader/utility.py`：工具函数、`BarGenerator`、`ArrayManager`、`virtual`
- `vnpy/trader/setting.py`：全局 `SETTINGS` 默认值与 `vt_setting.json` 加载
- `vnpy/trader/logger.py`：loguru 控制台/文件 sink 装配

**Out-of-Scope（不在本仓库源码内）**
- TA-Lib（`talib`）：实际技术指标计算库，仅被 `ArrayManager` 调用，不在本仓库源码内。
- loguru：日志库，不在本仓库源码内。
- tzlocal：本机时区探测库，不在本仓库源码内。
- `numpy`：数组底层实现，不在本仓库源码内。
- 各数据库驱动（`vnpy_sqlite` 等）、数据服务驱动：由 `database.py`/`datafeed.py` 叶子的注册表动态 import，不在本叶子。

## 8. 与相邻子系统交互

- **被调用方（上游）**：策略模板（cta_strategy 等外部 App）、`OmsEngine`、各 Monitor 通过 `update_tick`/`update_bar` 喂数据给 `BarGenerator`；策略通过 `ArrayManager.update_bar` 喂 bar 后调用 `.sma()` 等指标。
- **依赖（下游）**：`BarGenerator`/`ArrayManager` 消费 `object.py` 的 `TickData`/`BarData`、`constant.py` 的 `Exchange`/`Interval`；`setting.py` 依赖 `utility.load_json`；`logger.py` 依赖 `setting.SETTINGS` 与 `utility.get_folder_path`。
- **数据流方向**：Gateway 推送 Tick →（BarGenerator）→ 1min/多周期 Bar →（ArrayManager）→ 指标标量/数组 → 策略信号。

## 9. 语言专项适配口径（纯 Python：能力缝 + 注册表 + 插件体系）

- **能力缝归组**：本叶子是典型的"基础设施/横切关注点"能力缝集合，而非某一业务域。包含两类可复用组件：
  1. **回调注入式钩子链**：`BarGenerator` 构造时接收 `on_bar`/`on_window_bar` 两个 `Callable`（`utility.py:176`），由使用方注入后续处理——这是纯 Python 框架常见的可插拔回调模式，框架本身不决定合成后数据去向。
  2. **委托外部计算库的薄封装**：`ArrayManager` 的 40+ 指标方法同构地委托 `talib.*`（`utility.py:590`），本仓库只负责"滑动窗口维护 + array/标量双态返回"的胶水层，真正的指标计算在 TA-Lib。
- **配置驱动**：`SETTINGS` 全局 dict 是配置面工程的核心，被 logger/database/datafeed/UI 多处读取；`setting.py:43` 在 import 时用 `vt_setting.json` 覆盖默认值，属于"import 时副作用加载配置"。
- **懒加载/可选依赖**：本叶子无 `is_*_available()` 检测，TA-Lib/numpy 为硬依赖（pyproject 已声明）。
- **扩展点**：`virtual` 装饰器（`utility.py:1275`）是 vnpy 自定义的软扩展标记——基类方法不强制 `@abstractmethod`，但约定子类可覆写，与 ABC 硬抽象互补。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| config-utility 架构图 | `config-utility-architecture.html` | architecture | showcase |
| K 线合成数据流图 | `config-utility-dataflow.html` | dataflow | showcase |

JSON IR 位于 `json/` 目录。
本叶子未生成 sequence / lifecycle / workflow 图：`BarGenerator` 的合成逻辑本质是"数据从 Tick 经多级窗口变换为各周期 bar"的管道，已由 dataflow 图表达；其内部窗口状态变迁（空→累计→完成）与数据流主线重合，且无多参与方消息交互，按资源节省原则省略 sequence/lifecycle/workflow。
