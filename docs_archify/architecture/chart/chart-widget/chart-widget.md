# K线图表组件（chart-widget）

> 本文是 `chart` 域下的叶子子系统文档。域级总览见 `../chart.md`。
>
> 源码基准：`vnpy` 4.4.0（master），commit `fa5206fe`；纯 Python + PySide6/pyqtgraph，主语言口径见本文第 9 节。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| K 线数据管理 | 以 `datetime` 为键存储 BarData，双向映射 datetime↔整数下标，按可视区间缓存价格/成交量范围 | `vnpy/chart/manager.py:9`（`BarManager`） |
| 蜡烛图绘制 | 逐 K 线绘制影线 + 实体矩形，涨/跌配色（close>=open 为涨色） | `vnpy/chart/item.py:168`（`CandleItem`） |
| 成交量副图 | 逐根绘制成交量柱，颜色随涨跌 | `vnpy/chart/item.py:268`（`VolumeItem`） |
| 数据驱动的图形项框架 | `ChartItem` 抽象基类：按可见区间增量重绘、单根 K 线 QPicture 缓存、脏标记失效 | `vnpy/chart/item.py:12`（`ChartItem`） |
| 多副图布局 | `add_plot` 堆叠多个纵格子图（主图/成交量/指标），X 轴联动 | `vnpy/chart/widget.py:62`（`add_plot`） |
| 时间轴刻度 | 把整数下标转回 datetime 字符串，日内显示时分秒、日级显示日期 | `vnpy/chart/axis.py:10`（`DatetimeAxis`） |
| 十字光标 | 鼠标移动跟踪十字线、坐标标签、每根 K 线 OHLC 信息浮窗 | `vnpy/chart/widget.py:324`（`ChartCursor`） |
| 缩放与平移 | 键盘左右平移、上下/滚轮缩放（可视 K 线数 bar_count 缩放 1.2 倍） | `vnpy/chart/widget.py:237-311` |
| 配置常量 | 涨跌色、光标色、线宽、K 线宽、字体统一常量 | `vnpy/chart/base.py:4-16` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `BarManager` | `vnpy/chart/manager.py:9` | 数据中心：持有 bars 字典与双向索引，提供 get_price_range/get_volume_range（带缓存） |
| `ChartItem`（抽象基类） | `vnpy/chart/item.py:12` | 继承 `pg.GraphicsObject`；定义绘制契约 `_draw_bar_picture`/`boundingRect`/`get_y_range`/`get_info_text`；实现 QPicture 缓存与可见区增量重绘 |
| `CandleItem` | `vnpy/chart/item.py:168` | 蜡烛图实现：影线 + 实体 |
| `VolumeItem` | `vnpy/chart/item.py:268` | 成交量柱实现 |
| `DatetimeAxis` | `vnpy/chart/axis.py:10` | 继承 `pg.AxisItem`；`tickStrings` 把下标转 datetime 文本 |
| `ChartWidget` | `vnpy/chart/widget.py:20` | 继承 `pg.PlotWidget`；顶层控件，组合 manager/plots/items/cursor，对外暴露 add_plot/add_item/update_history/update_bar |
| `ChartCursor` | `vnpy/chart/widget.py:324` | 十字光标：竖/横线、坐标标签、信息浮窗；响应 `sigMouseMoved` |
| `to_int(value)` | `vnpy/chart/base.py:19` | 浮点下标四舍五入为整数 |

## 3. 关键调用链

**调用链 1：历史数据初始化渲染**

1. 外部调用 `ChartWidget.update_history(history)`（`vnpy/chart/widget.py:155`）。
2. 委托 `self._manager.update_history(history)`（`widget.py:159`）：把 bars 写入字典、按 datetime 排序（`manager.py:26-30`）、重建双向索引 map（`manager.py:33-37`）、清空范围缓存（`manager.py:40`）。
3. 遍历所有 item 调用 `item.update_history(history)`（`widget.py:161`）：清空 `_bar_picutures` 缓存并按 bars 数量重置为 None（`item.py:78-85`），触发 `self.scene().update()`（`item.py:105`）。
4. `_update_plot_limits()` 为每个 plot 设置 x/y 范围边界（`widget.py:182-194`），最后 `move_to_right()` 把视图定位到最右侧最新 K 线（`widget.py:166,313`）。
5. Qt 场景触发 `ChartItem.paint(painter, opt, w)`（`item.py:107`）：从 `opt.exposedRect` 取可见区间 min_ix/max_ix（`item.py:118-122`）；若 `_to_update` 或可视区间变化，则 `_draw_item_picture(min_ix, max_ix)`（`item.py:125-132`），循环内对未缓存的 K 线调用 `_draw_bar_picture(ix, bar)` 生成 QPicture 并缓存（`item.py:144-155`），最后 `_item_picuture.play(painter)` 一次性贴到画布。

**调用链 2：实时单根 K 线更新**

1. 外部推送新 bar，调用 `ChartWidget.update_bar(bar)`（`widget.py:168`）。
2. `manager.update_bar(bar)`：若 datetime 是新 K 线则分配新下标（`manager.py:48-51`），更新 bar 并清缓存（`manager.py:53-55`）。
3. 遍历 item 调用 `item.update_bar(bar)`（`widget.py:174`）：通过 `manager.get_index(bar.datetime)` 找到 ix，把该根的 QPicture 置 None 使其下次重绘（`item.py:91-97`）。
4. `_update_plot_limits()` 刷新边界；若视图已贴右沿（`_right_ix >= count - bar_count/2`）则自动 `move_to_right()` 跟随最新行情（`widget.py:179-180`）。

**调用链 3：缩放/平移联动 Y 轴自适应**

1. 用户拖拽/滚轮改变 X 可视范围，`ViewBox.sigXRangeChanged` 触发 `ChartWidget._update_y_range`（`widget.py:94,206`）。
2. 从 `view.viewRange()` 取当前可视 ix 区间（`widget.py:213-217`），对每个 item 调用 `get_y_range(min_ix, max_ix)`（`widget.py:220-221`）。
3. `CandleItem.get_y_range` 委托 `manager.get_price_range(min_ix, max_ix)`（`item.py:226-233`），后者优先查 `_price_ranges` 缓存，未命中才遍历区间内 high/low 求极值并回写缓存（`manager.py:108-122`）。
4. `plot.setRange(yRange=...)` 让 Y 轴随可视 K 线区间自适应缩放。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `MIN_BAR_COUNT` | 100，缩放时可视 K 线数下限 | `vnpy/chart/widget.py:22` |
| `_bar_count` | 初始 = MIN_BAR_COUNT；每次缩放 ×1.2（缩小）或 ÷1.2（放大） | `widget.py:38,293,305` |
| `BAR_WIDTH` | 0.3，K 线实体半宽 | `vnpy/chart/base.py:13` |
| `PEN_WIDTH` / `AXIS_WIDTH` | 1 / 0.8，画笔与轴线宽 | `base.py:12,15` |
| `UP_COLOR` / `DOWN_COLOR` | 涨红 (255,75,75) / 跌青 (0,255,255) | `base.py:8-9` |
| `CURSOR_COLOR` | 光标标签底色 (255,245,162) | `base.py:10` |
| `NORMAL_FONT` | Arial 9 号 | `base.py:16` |
| `add_plot(minimum_height)` | 子图最小高默认 80px，可传 maximum_height/hide_x_axis | `widget.py:62-87` |
| `antialias` | 模块级 `pg.setConfigOptions(antialias=True)` | `widget.py:17` |
| `setDownsampling(mode="peak")` | 子图启用峰值降采样 | `widget.py:78` |

## 5. 错误与重试语义

- **数据为空**：`manager.get_price_range`/`get_volume_range` 在无 bars 时返回 `(0, 1)` 占位范围（`manager.py:97-98,128-129`），避免除零/空序列极值错误。
- **下标越界**：`get_bar(ix)`/`get_datetime(ix)` 查不到 datetime 时返回 None（`manager.py:74,81-83`）；`paint` 循环内 bar 为 None 直接 `continue`（`item.py:149-150`）。
- **未知 item 信息**：`get_info_text` 在 bar 为 None 时返回空串（`item.py:262-263,330-331`）。
- **无重试**：本叶子是纯渲染层，无网络/IO 重试语义；数据更新失败由上游数据加载层负责。
- **缓存一致性**：每次 `update_history`/`update_bar` 都调用 `_clear_cache()` 清空价格/成交量范围缓存（`manager.py:40,55,155-160`），避免脏数据。

## 6. 并发细节

- **UI 线程模型**：本叶子完全运行在 PySide6/Qt 主线程（GUI 事件循环），不另起工作线程；所有 `update_bar`/`update_history` 调用应在主线程触发（外部通常经事件引擎信号槽跨线程投递）。
- **无显式锁**：数据字典与缓存都在 GUI 线程访问，无多线程竞争；`_bar_picutures`/`_price_ranges` 的读写无锁。
- **增量重绘优化**：`setFlag(ItemUsesExtendedStyleOption)`（`item.py:39`）让 `paint` 只重绘 `opt.exposedRect` 可见区间；`_to_update` 脏标记 + `_rect_area` 区间比较避免拖动时全量重绘（`item.py:125-132`）；单根 K 线 QPicture 缓存避免重复绘制同一根。
- **信号槽连接**：`view.sigXRangeChanged → _update_y_range`（`widget.py:94`）、`scene.sigMouseMoved → ChartCursor._mouse_moved`（`widget.py:421`），均为 Qt 队列连接，在 GUI 线程串行执行。
- **取消传播**：无取消概念；`clear_all()` 显式清空全部数据与缓存。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `vnpy/chart/widget.py`：ChartWidget 顶层控件、多子图布局、缩放平移、ChartCursor 十字光标
- `vnpy/chart/item.py`：ChartItem 抽象基类、CandleItem、VolumeItem 的 QPicture 绘制
- `vnpy/chart/manager.py`：BarManager 数据存储与范围缓存
- `vnpy/chart/axis.py`：DatetimeAxis 下标→时间刻度
- `vnpy/chart/base.py`：颜色/尺寸/字体常量

**Out-of-Scope（不在本仓库源码内）**
- **pyqtgraph**：`pg.PlotWidget`/`PlotItem`/`ViewBox`/`GraphicsObject`/`AxisItem`/`InfiniteLine`/`TextItem`/`QPicture` 渲染框架、降采样、场景视图管理均由 pyqtgraph 提供（不在本仓库源码内）。
- **PySide6（Qt）**：QPainter/QPixmap/QGraphicsScene 渲染、事件循环、信号槽、QFont 由 PySide6 提供（不在本仓库源码内）。
- **BarData 数据结构**：K 线数据对象定义在 `vnpy/trader/object.py`（trader-core 域），本叶子只消费。
- **行情数据来源**：历史 K 线加载、实时 bar 推送由 MainEngine/数据服务/网关负责（trader-core 域），本叶子不生产数据。
- **技术指标计算**：MACD/KDJ/均线等指标不在本叶子；本叶子只提供 add_item 扩展点由外部传入自定义 ChartItem 子类。
- **交易交互**：不在本叶子，图表仅展示。

## 8. 与相邻子系统交互

- **上游 → 本叶子**：`vnpy.trader.ui`（主窗口/图表页面）创建 `ChartWidget`，调用 `add_plot` + `add_item(CandleItem/VolumeItem)` 组装图表，再 `update_history(bars)` 灌历史、`update_bar(bar)` 推实时。BarData 来自 `vnpy.trader` 的行情服务。
- **本叶子 → 下游**：
  - ChartItem/BarManager/DatetimeAxis 全部建立在 pyqtgraph 图元体系之上（继承 `pg.GraphicsObject`/`pg.AxisItem`）。
  - BarData 对象来自 `vnpy.trader.object`。
- **扩展方向**：用户可继承 `ChartItem` 实现自定义指标子图（如均线、MACD），经 `add_item(MyItem, "ma", "main")` 挂到指定 plot——这是本叶子的插件扩展缝。
- **相邻叶子指引**：事件分发与数据推送见 trader-core 域 event-engine；BarData/枚举定义见 trader-core 域 objects-constants。

## 9. 语言专项适配口径（纯 Python：能力缝 + 注册表 + 插件体系）

- **能力缝（capability seam）**：本叶子的核心能力缝是"**图形项抽象基类 + 组合式子图装配**"。`ChartItem`（`item.py:12`）用四个 `@abstractmethod`（`_draw_bar_picture`/`boundingRect`/`get_y_range`/`get_info_text`）定义"一根 K 线怎么画、Y 轴范围、光标信息"契约，`CandleItem`/`VolumeItem` 是两个内置实现；用户新增指标只需子类化 `ChartItem` 并实现四方法，经 `ChartWidget.add_item(item_class, name, plot_name)` 注入——这是典型的"基类定义契约 + 外部插件注册"。
- **注册表/组合表**：`_plots: dict[str, PlotItem]`（`widget.py:30`）、`_items: dict[str, ChartItem]`（`widget.py:31`）、`_item_plot_map: dict[ChartItem, PlotItem]`（`widget.py:32`）是三张内部组合表，按名字路由子图与图形项；`_v_lines`/`_h_lines`/`_y_labels`/`_infos`（`widget.py:359-415`）按 plot_name 分组管理光标组件。
- **性能模式（隐藏状态在循环里）**：`paint()`（`item.py:107`）是 Qt 每帧回调的渲染循环，内部用 `_to_update` 脏标记 + `_rect_area` 区间比较 + 逐根 QPicture 缓存构成"可视区增量重绘"状态机——这是纯 Python GUI 框架中典型的"循环内隐藏状态机"，故补 dataflow 图表达"数据→缓存→可见区→画面"的渲染管线。
- **可选依赖**：pyqtgraph/PySide6 是核心依赖（非 extras）；本叶子无懒加载降级，import 即要求 GUI 环境。
- **配置面**：颜色/尺寸/字体集中在 `base.py` 模块级常量，无 dataclass 配置对象；缩放倍数 1.2、最小 K 线数 100 为硬编码。
- **图型侧重**：本叶子是数据→渲染管线，故除 architecture 外补 dataflow 图表达"BarData→BarManager 索引/缓存→ChartItem QPicture→Qt 画布"的数据流；sequence 图因渲染由 Qt 事件循环驱动、无明确多方消息交互，按资源节省原则省略。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 图表组件架构图 | [chart-widget-architecture.html](chart-widget-architecture.html) | architecture | 待渲染 |
| K 线渲染数据流图 | [chart-widget-dataflow.html](chart-widget-dataflow.html) | dataflow | 待渲染 |

- JSON IR 源文件位于 `json/` 目录。
- 本叶子未生成 sequence 图：渲染由 Qt `paintEvent`/信号槽在单线程 GUI 事件循环内驱动，无多参与方按时间先后的消息交互；未生成 lifecycle 图：ChartWidget 无多阶段状态机（仅 active 数据更新）；未生成 workflow 图：无带泳道的审批/分步流程。
