# 主窗口与交易 UI（main-window）

> 本文是 `trader-ui` 域下唯一的叶子子系统文档。域级总览见 `../trader-ui.md`。
> 本文只展开 vnpy 桌面 GUI 的主窗口装配、监控表格基类与事件→UI 线程桥接，不重复展开 K 线图表控件（`vnpy/chart/`，归 chart 域）与各 App 自带业务界面。
>
> 源码基准：vnpy 4.4.0，commit `fa5206fe`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| QApplication 工厂 | `create_qapp`：建 QApplication、套 qdarkstyle 暗色主题、设字体/图标、装全局异常钩子 | `vnpy/trader/ui/qt.py:21` |
| 全局异常弹窗 | `ExceptionWidget`：跨线程异常经 `Signal(str)` 投到主线程弹窗，支持复制/求助 | `qt.py:74` |
| 主窗口装配 | `MainWindow`：菜单栏/工具栏/8 个停靠窗口（交易/行情/委托/活动/成交/日志/资金/持仓） | `vnpy/trader/ui/mainwindow.py:40` |
| 停靠窗口创建 | `create_dock`：实例化监控控件并包成 `QDockWidget`，浮动可拖动 | `mainwindow.py:229` |
| 功能菜单动态加载 | 遍历 `main_engine.get_all_apps()`，`import_module(app_module+".ui")` 取 widget 类注册到"功能"菜单 | `mainwindow.py:131` |
| 窗口状态持久化 | `save/load_window_setting` 用 `QSettings` 存窗口 state/geometry 与各 Monitor 列宽 | `mainwindow.py:297`、`widget.py:404` |
| 退出确认 | `closeEvent`：确认后关闭所有 widget、存 Monitor 列宽、`main_engine.close()` | `mainwindow.py:256` |
| 监控表格基类 | `BaseMonitor(QTableWidget)`：类属性声明 event_type/data_key/headers，自动注册事件并增/改行 | `widget.py:241` |
| 单元格类型族 | `BaseCell`/`EnumCell`/`BidCell`/`AskCell`/`PnlCell`/`TimeCell`/`DateCell`/`MsgCell`：按列定制显示与着色 | `widget.py:60` |
| 行情监控子类 | `TickMonitor`/`LogMonitor`/`TradeMonitor`/`OrderMonitor`/`PositionMonitor`/`AccountMonitor`/`QuoteMonitor` | `widget.py:419` 起 |
| 交易下单面板 | `TradingWidget`：5 档行情展示、下单/撤单按钮、`process_tick_event` 刷新 | `widget.py:704`、`widget.py:873` |
| 微信绑定后台线程 | `WechatWorker(QThread)`：两阶段扫码绑定（取码→等首条消息），信号回主线程 | `widget.py:1307` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `create_qapp` | `qt.py:21` | 进程入口创建唯一 QApplication，装异常钩子 |
| `ExceptionWidget` | `qt.py:74` | 跨线程异常统一展示；`signal = Signal(str)` 是跨线程投递点 |
| `MainWindow` | `mainwindow.py:40` | 主窗口；持有 `widgets`/`monitors` 两个 dict 管理浮动窗口与监控表 |
| `BaseMonitor` | `widget.py:241` | 监控表抽象基类；`signal = Signal(Event)`（`widget.py:251`）把事件引擎线程的事件桥接到 Qt 主线程 |
| `BaseMonitor.register_event` | `widget.py:298` | `event_engine.register(self.event_type, self.signal.emit)`——事件回调直接是 Qt 信号 emit |
| `BaseMonitor.process_event` | `widget.py:306` | 在主线程槽里按 data_key 命中情况 insert 新行 / update 旧行 |
| `BaseCell` 族 | `widget.py:60` | 表格单元格；`BidCell`/`AskCell` 红涨绿跌着色 |
| `WechatWorker` | `widget.py:1307` | `QThread` 子类，`run()` 内同步轮询，用 4 个 Signal 把结果投回主线程对话框 |

## 3. 关键调用链

**调用链一：事件引擎事件 → Monitor 表格刷新（Qt 线程桥）**
1. `widget.py:304`：`BaseMonitor.register_event` 把 `self.signal.emit` 注册为某 event_type 的回调。
2. 事件引擎在其分发线程里对该事件类型调用 `signal.emit(event)`（`widget.py:304`）。
3. Qt 信号槽机制自动把槽 `process_event`（`widget.py:306`）调度到 GUI 主线程执行——**这是跨线程安全的关键**。
4. `widget.py:315`–`325`：取 `event.data`，按 `data_key`（如 `vt_symbol`）查 `self.cells`，命中则 `update_old_row`，否则 `insert_new_row`。
5. `widget.py:311`–`329`：更新期间临时关闭排序，完成后恢复。

**调用链二：主窗口初始化装配（`MainWindow.__init__` → `init_ui`）**
1. `mainwindow.py:57`：`init_ui` 依次 `init_dock`→`init_toolbar`→`init_menu`→`load_window_setting("custom")`。
2. `mainwindow.py:69`–`92`：`create_dock` 实例化 8 个监控/交易控件并放到左/右/下停靠区。
3. `mainwindow.py:131`–`138`：遍历已注册 App，动态 import 各 App 的 `.ui` 模块并把 widget 挂到"功能"菜单。
4. `mainwindow.py:65`：最后 `load_window_setting("custom")` 恢复用户上次窗口布局。

**调用链三：交易面板 tick 刷新（`TradingWidget.process_tick_event`）**
1. `widget.py:870`–`871`：`signal_tick.connect(process_tick_event)` 并 `event_engine.register(EVENT_TICK, self.signal_tick.emit)`。
2. `widget.py:876`：只处理 `tick.vt_symbol == self.vt_symbol` 的行情。
3. `widget.py:881`–`910`：按价格精度刷新最新价/买卖五档标签。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `font.family`/`font.size` | "微软雅黑"/12，`create_qapp` 设全局字体 | `setting.py:12`、`qt.py:30` |
| `QSettings` 组织/应用名 | 窗口标题/类名作为 `QSettings` 作用域，持久化布局与列宽 | `mainwindow.py:301`、`widget.py:406` |
| Monitor 列状态 | 类名 + "custom" 作用域存 `column_state` | `widget.py:406` |
| WechatWorker 超时 | 10 分钟（600s）扫码/等消息截止 | `widget.py:1331` |

## 5. 错误与重试语义

- **全局异常钩子**：`create_qapp` 覆盖 `sys.excepthook`（主线程，`qt.py:46`）与 `threading.excepthook`（后台线程，`qt.py:60`）——异常经 loguru 记录后通过 `ExceptionWidget.signal.emit` 弹窗展示，不静默崩溃。
- 退出确认（`mainwindow.py:260`）：用户选"否"则 `event.ignore()` 阻止关闭。
- Monitor 更新时临时关排序（`widget.py:311`）避免排序中行索引错乱异常。
- `WechatWorker.run`（`widget.py:1328`）捕获 `WeixinTimeout` 继续轮询，捕获 `WeixinError`/通用 `Exception` 经 `signal_failed` 投回主线程；10 分钟超时失败。
- 无自动重试 UI 操作；下单失败由 OmsEngine 事件回流显示。

## 6. 并发细节

- **Qt 主线程模型**：所有控件创建与槽执行在 GUI 主线程；事件引擎在独立线程分发事件。
- **线程桥（核心）**：`BaseMonitor.signal = Signal(Event)` + `event_engine.register(event_type, signal.emit)`（`widget.py:251`、`widget.py:304`）——PySide6 队列信号自动跨线程排队到主线程，是本域并发安全的基石。
- **后台工作线程**：`WechatWorker(QThread)`（`widget.py:1307`）在子线程跑同步轮询，通过 `signal_qr_ready`/`signal_bound`/`signal_failed` 等信号回主线程；`stop()` 设 `_stop` 标志在下个循环点退出。
- **无显式锁**：Monitor 表格只在主线程槽里读写，天然串行；跨线程只传递不可变 Event 数据。
- `closeEvent` 顺序：先关浮动 widget → 存 Monitor 列宽 → 存窗口布局 → `main_engine.close()`（`mainwindow.py:269`–`277`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `vnpy/trader/ui/mainwindow.py`：主窗口装配与菜单/工具栏/停靠管理。
- `vnpy/trader/ui/qt.py`：QApplication 工厂与异常弹窗。
- `vnpy/trader/ui/widget.py`：监控表基类、单元格族、交易面板、连接对话框、微信绑定对话框与后台线程。

**Out-of-Scope（不在本仓库源码内）**
- PySide6（Qt 绑定）、qdarkstyle、pyqtgraph、qrcode——GUI 框架与样式库，不在本仓库源码内。
- `vnpy.event.EventEngine`（事件分发线程本体）——在 `vnpy/event/engine.py`，归 event-engine 叶子。
- `vnpy.trader.engine.MainEngine`/`OmsEngine`（引擎本体）——归 trader-core 域。
- K 线图表控件 `vnpy/chart/`——归 chart 域。
- 各 App 自带业务界面（`vnpy_ctastrategy.ui` 等）——外部 App 包，不在本仓库。

## 8. 与相邻子系统交互

- **上游**：`examples/veighna_trader` 入口先 `create_qapp()` → 建 `MainEngine`/`EventEngine` → `MainWindow(main_engine, event_engine).show()`。
- **依赖（下游）**：UI 通过 `event_engine.register` 订阅事件（EVENT_TICK/ORDER/TRADE/POSITION/ACCOUNT/LOG），通过 `main_engine.send_order/cancel_order/connect` 下单与连接网关。
- **方向**：Gateway → EventEngine（子线程发事件）→ Qt Signal → Monitor 槽（主线程刷新表）；用户点击 → TradingWidget → main_engine.send_order → Gateway。

## 9. 语言专项适配口径（纯 Python：能力缝 + 注册表 + 插件体系）

- **能力缝（类属性声明式监控）**：`BaseMonitor` 是典型的"声明式基类"能力缝——子类只需覆盖类属性 `event_type`/`data_key`/`headers`（如 `TickMonitor`，`widget.py:419`），基类 `register_event`/`process_event`/`insert_new_row` 自动完成事件订阅与表格增改。新增一种监控只需写十几行类属性，无需碰事件分发代码。
- **注册表/工厂（动态 App UI 加载）**：`MainWindow.init_menu`（`mainwindow.py:131`–`138`）遍历 `get_all_apps()`，按 `app.app_module + ".ui"` 字符串动态 import 并用 `getattr(widget_class)` 取控件类——这是纯 Python 框架"App 插件注册 UI"的扩展点：外部 App 包只需约定 `.ui` 模块导出 `widget_name` 类即被自动挂到菜单。
- **跨线程钩子链（Qt Signal 桥）**：`signal = Signal(Event)` + `event_engine.register(..., signal.emit)` 是事件线程→GUI 主线程的标准可插拔桥接，每个 Monitor 自带一个信号实例。
- **可选依赖/懒加载**：App UI 模块在用户点击菜单时才 `import_module`（`mainwindow.py:133`），主窗口启动不加载所有 App 界面。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| main-window 组件架构图 | `main-window-architecture.html` | architecture | showcase |
| 事件→UI 刷新时序图 | `main-window-sequence.html` | sequence | showcase |

JSON IR 位于 `json/` 目录。
本叶子未生成 dataflow / lifecycle / workflow 图：UI 是"事件驱动的表格刷新 + 菜单装配"，其核心跨线程交互已由 sequence 表达；窗口生命周期（创建→布局→关闭）与微信绑定状态机已在 notify-wechat 叶子覆盖扫码流程，UI 侧只是信号转发，不另画 lifecycle。
