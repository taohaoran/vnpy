# K线图表（chart）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`vnpy` 4.0.0（master），commit `fa5206fe`。

## 1. 域职责

`vnpy.chart` 是 vnpy 框架的桌面 K 线图表组件层，基于 pyqtgraph（PySide6 绑定）实现：

- **K 线可视化**：蜡烛图 + 成交量副图，支持多子图纵向堆叠、X 轴联动。
- **数据驱动渲染**：以 BarManager 为数据中心，双向索引 datetime↔整数下标，按可视区间缓存价格/量范围。
- **高性能重绘**：每个 ChartItem 把单根 K 线绘制成 QPicture 缓存，paint 只重绘可见区间（`ItemUsesExtendedStyleOption`），拖动/缩放不全量重画。
- **交互**：键盘左右平移、上下/滚轮缩放（bar_count 按 1.2 倍缩放）、十字光标跟踪 OHLC 信息。

本域在整个 vnpy 架构中定位为"**GUI 展示层组件**"——它只消费 BarData、渲染图形，不生产行情、不计算指标、不做交易决策；技术指标可通过继承 `ChartItem` 插件式扩展。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 / 数据流图 | 职责一句话 |
|---|---|---|---|---|
| chart-widget | [chart-widget.md](chart-widget/chart-widget.md) | [架构图](chart-widget/chart-widget-architecture.html) | [渲染数据流图](chart-widget/chart-widget-dataflow.html) | pyqtgraph K 线图组件：数据管理+图元渲染+十字光标 |

## 3. 域级机制细节

- **三层结构**：`ChartWidget`（顶层装配）→ `BarManager`（数据/缓存）+ `ChartItem` 族（图元）+ `DatetimeAxis`（时间轴）+ `ChartCursor`（光标）。
- **抽象基类扩展缝**：`ChartItem` 用四个抽象方法（`_draw_bar_picture`/`boundingRect`/`get_y_range`/`get_info_text`）定义绘制契约，内置 CandleItem/VolumeItem，用户可子类化扩展指标子图。
- **增量重绘**：`_to_update` 脏标记 + `_rect_area` 区间比较 + 逐根 QPicture 缓存，是本域性能核心。
- **Y 轴自适应**：X 可视区间变化触发 `sigXRangeChanged → _update_y_range`，按可见 K 线 high/low/volume 极值重算 Y 范围，带 `_price_ranges`/`_volume_ranges` 区间缓存。
- **外部依赖**：pyqtgraph/PySide6 承担实际渲染（不在本仓库源码内）；BarData 来自 `vnpy.trader.object`。

## 4. 域级图

![图表域架构图](chart-architecture.html)
![K线更新渲染时序图](chart-sequence.html)
![图表域数据流图](chart-dataflow.html)

- 三张域级图均为 showcase 档；JSON IR 位于 `json/` 目录。
- 本域未生成 workflow 图：无带泳道的审批/分步流程；未生成 lifecycle 图：ChartWidget 无多阶段状态机。
