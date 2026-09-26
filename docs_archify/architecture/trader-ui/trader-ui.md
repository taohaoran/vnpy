# trader-ui 域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：vnpy 4.4.0，commit `fa5206fe`。

## 1. 域职责

trader-ui 域是 vnpy 基于 PySide6 的**桌面 GUI 层**，负责：
- 创建并装配 QApplication（暗色主题、字体、全局异常钩子）；
- 主窗口的菜单/工具栏/停靠窗口布局与窗口状态持久化；
- 各类监控表格（行情/委托/成交/持仓/资金/日志）的声明式实现；
- 交易下单面板与网关连接对话框；
- 微信绑定对话框及其后台轮询线程。

本域的核心并发机制是 **Qt Signal 跨线程桥**：事件引擎在子线程分发事件，经 `Signal.emit` 自动排队到 GUI 主线程槽执行，保证表格刷新线程安全。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 | 职责一句话 |
|------|------|--------|--------|-----------|
| main-window | [main-window.md](main-window/main-window.md) | [架构图](main-window/main-window-architecture.html) | [时序](main-window/main-window-sequence.html) | 主窗口装配、监控表基类与事件→UI 桥接 |

> 本域仅 1 个叶子（main-window），该叶子已覆盖 `mainwindow.py`/`qt.py`/`widget.py` 全部职责；K 线图表控件 `vnpy/chart/` 归 chart 域，不在本域。

## 3. 域级机制细节

- **声明式监控表**：`BaseMonitor` 基类 + 子类类属性（`event_type`/`data_key`/`headers`）模式，新增监控只需声明列定义，基类自动完成事件订阅与增改行。
- **App UI 动态加载**：主窗口"功能"菜单遍历已注册 App，按 `app_module + ".ui"` 字符串动态 import 并挂菜单——外部 App 包即插即用。
- **跨线程异常处理**：`create_qapp` 同时覆盖 `sys.excepthook` 与 `threading.excepthook`，后台线程异常经信号弹窗展示。
- **后台工作线程**：`WechatWorker(QThread)` 在子线程跑微信扫码轮询，通过 4 个 Signal 回主线程，避免阻塞 Qt 事件循环。

## 4. 域级图

![trader-ui 域架构图](trader-ui-architecture.html)
![trader-ui 启动装配时序](trader-ui-sequence.html)
![trader-ui 事件流](trader-ui-dataflow.html)
