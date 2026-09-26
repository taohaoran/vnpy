# 策略投研与回测（alpha-strategy）

> 本文是 `alpha` 域下的叶子子系统文档。域级总览见 `../alpha.md`。
>
> 源码基准：`vnpy` 4.4.0（master），commit `fa5206fe`；纯 Python + polars，主语言口径见本文第 9 节。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 策略抽象基类 | `AlphaStrategy`（ABCMeta）：定义 on_init/on_bars/on_trade 回调，提供 buy/sell/short/cover/send_order/cancel_order、目标仓位管理 | `vnpy/alpha/strategy/template.py:15` |
| 目标仓位调仓 | `execute_trading(bars, price_add)`：按 target-pos 差额拆 cover/buy 或 sell/short 限价单 | `template.py:133` |
| 回测引擎 | `BacktestingEngine`：加载历史、按时间回放、撮合限价单、逐日盯市盈亏 | `vnpy/alpha/strategy/backtesting.py:22` |
| 历史回放 | `run_backtesting` → 按排序 dt 逐根 `new_bars(dt)`，空数据填昨收 bar | `backtesting.py:150,579` |
| 限价单撮合 | `cross_order`：按 bar low/high 判断穿越，涨停跌停不成交，成交价取 min/max(限价,开盘) | `backtesting.py:619` |
| 信号取数 | `get_signal()` 按当前 dt 从 signal_df 过滤当日预测 | `backtesting.py:709` |
| 逐日盈亏 | `PortfolioDailyResult`/`ContractDailyResult` 计算 trading_pnl/holding_pnl/佣金 | `backtesting.py:799,875` |
| 绩效统计 | `calculate_statistics` 算年化/夏普/最大回撤等；`show_performance` 对接 plotly | `backtesting.py:228,440` |
| 示例策略 | EquityDemoStrategy：top_k 多头选股，按信号排序定期调仓 | `strategies/equity_demo_strategy.py:12` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `AlphaStrategy` | `template.py:15` | 策略模板：on_init/on_bars/on_trade 抽象回调；pos_data/target_data/orders/active_orderids 状态 |
| `AlphaStrategy.execute_trading` | `template.py:133` | 按目标仓位差额生成买卖限价单 |
| `BacktestingEngine` | `backtesting.py:22` | 回测引擎：set_parameters/add_strategy/load_data/run_backtesting/calculate_result |
| `BacktestingEngine.new_bars` | `backtesting.py:579` | 单根时间切片：更新 bars→cross_order→on_bars→update_daily_close |
| `BacktestingEngine.cross_order` | `backtesting.py:619` | 限价单撮合与成交生成 |
| `ContractDailyResult` | `backtesting.py:799` | 单合约单日盈亏 |
| `PortfolioDailyResult` | `backtesting.py:875` | 组合单日盈亏汇总 |
| `EquityDemoStrategy` | `equity_demo_strategy.py:12` | 多头 top_k 选股示例策略 |

## 3. 关键调用链

**调用链 1：回测主循环**

1. 用户 `set_parameters(vt_symbols, interval, start, end, capital)`（`backtesting.py:70`）从 lab 读合约费率/乘数（`backtesting.py:92-102`）。
2. `add_strategy(strategy_class, setting, signal_df)`（`backtesting.py:104`）实例化策略、存信号表。
3. `load_data()`（`backtesting.py:112`）逐标的 `lab.load_bar_data` 读历史，按 `(datetime, vt_symbol)` 存 history_data、收集 dts 集合（`backtesting.py:129-139`）。
4. `run_backtesting()`（`backtesting.py:150`）：先 `strategy.on_init()`（`backtesting.py:152`），dts 排序后逐 dt `new_bars(dt)`，异常则终止回测（`backtesting.py:160-166`）。
5. `new_bars(dt)`（`backtesting.py:579`）：更新 pre_closes、取该 dt 各标的 bar（缺数据用昨收填 fill_bar，`backtesting.py:599-612`）；先 `cross_order()` 撮合旧挂单（`backtesting.py:614`），再 `strategy.on_bars(bars)`（`backtesting.py:615`），最后 `update_daily_close`（`backtesting.py:617`）。

**调用链 2：限价单撮合**

1. `cross_order()`（`backtesting.py:619`）遍历 active_limit_orders：买单穿越条件 `order.price >= bar.low_price 且 low>0 且未全天涨停`（`backtesting.py:642-647`）；卖单 `order.price <= bar.high_price 且 high>0 且未全天跌停`（`backtesting.py:649-654`）。
2. 成交：order.status=ALLTRADED，从 active 移除（`backtesting.py:660-665`）；成交价买单 `min(order.price, open)`、卖单 `max(order.price, open)`（`backtesting.py:670-673`）。
3. 生成 TradeData（`backtesting.py:675-686`），按 long/short 费率算佣金、更新现金（`backtesting.py:689-703`），调 `strategy.update_trade(trade)` 更新持仓（`backtesting.py:706`）。

**调用链 3：策略调仓**

1. `EquityDemoStrategy.on_bars`（`equity_demo_strategy.py:38`）：`get_signal()` 取当日信号并按 signal 降序（`equity_demo_strategy.py:41-42`）。
2. 生成 sell_symbols（持仓中跌出 top_k 或不在成分股）与 buy_symbols（`equity_demo_strategy.py:51-65`）；对卖出股票 `set_target(0)`，对买入股票按可用现金等权 `set_target(volume)`（`equity_demo_strategy.py:70-98`）。
3. `execute_trading(bars, price_add)`（`template.py:133`）：先 `cancel_all()`，对每个标的算 target-pos 差额，正差先 cover 后 buy（买价 close*(1+price_add)），负差先 sell 后 short（卖价 close*(1-price_add)）（`template.py:145-185`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `capital` | 初始资金 1,000,000 | `backtesting.py:76,90` |
| `risk_free` | 无风险利率 0 | `backtesting.py:77` |
| `annual_days` | 年交易日 240 | `backtesting.py:78` |
| 合约费率 | long_rate/short_rate/size/pricetick，从 lab contract.json 读 | `backtesting.py:99-102` |
| `price_add` | 限价单相对收盘价滑点（EquityDemo 0.05） | `equity_demo_strategy.py:23` |
| `top_k` | 持仓股票数上限 50 | `equity_demo_strategy.py:15` |
| `n_drop` | 每次调仓卖出垫底股票数 5 | `equity_demo_strategy.py:16` |
| `min_days` | 最短持有天数 3 | `equity_demo_strategy.py:17` |
| `cash_ratio` | 资金使用率 0.95 | `equity_demo_strategy.py:18` |
| 涨跌停 | 涨停 = pre_close*1.1、跌停 = pre_close*0.9，按 pricetick round | `backtesting.py:638-639` |

## 5. 错误与重试语义

- **回测异常终止**：`run_backtesting` 中 `new_bars` 抛异常则记录 traceback 后 return，回测中止（`backtesting.py:161-166`），不重试。
- **无合约配置**：`set_parameters` 缺合约设置仅 warning（`backtesting.py:95-97`）。
- **无成交**：`calculate_result` 在 `self.trades` 为空时返回 None（`backtesting.py:174-176`）。
- **信号缺失**：`get_signal` 当日无信号仅 warning，返回空 df（`backtesting.py:718-719`）。
- **限价单不撤销重挂**：未成交限价单留在 active_limit_orders，后续每根 bar 继续撮合；`execute_trading` 开头 `cancel_all()` 撤旧单再下新单。

## 6. 并发细节

- **单线程回放**：`run_backtesting` 是同步 for 循环逐 dt 回放，无工作线程。
- **无锁**：引擎/策略状态在单线程内访问。
- **事件顺序**：每根 dt 固定顺序 cross_order（先撮合旧单）→ on_bars（策略下单）→ update_daily_close，保证"旧单先成交、策略再决策"。
- **取消传播**：无；回测一旦启动跑到结束或异常。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `vnpy/alpha/strategy/template.py`：AlphaStrategy 策略模板与调仓执行
- `vnpy/alpha/strategy/backtesting.py`：BacktestingEngine 回测引擎 + 日盈亏结果
- `vnpy/alpha/strategy/strategies/equity_demo_strategy.py`：示例选股策略

**Out-of-Scope（不在本仓库源码内）**
- **BarData/OrderData/TradeData/Status/Offset/Direction**：交易数据结构与枚举在 `vnpy/trader/object.py`/`constant.py`（trader-core 域）。
- **round_to/extract_vt_symbol**：`vnpy.trader.utility`（trader-service 域）。
- **plotly/matplotlib**：绩效图表绘制（`backtesting.py:404 show_chart`，不在本仓库源码内）。
- **信号来源**：signal_df 由 alpha-model 的 predict 产出（见 alpha-model 叶子），本叶子只消费。
- **实盘交易**：本叶子只做回测；实盘网关见 trader-core 域 gateway-converter。

## 8. 与相邻子系统交互

- **上游 → 本叶子**：AlphaLab 提供历史 K 线（`load_bar_data`）与合约配置；AlphaModel.predict 产出 signal_df；用户传入 strategy_class 与 setting。
- **本叶子 → 下游**：
  - 策略经 `send_order` → 引擎 `send_order`（`backtesting.py:723`）生成限价单入 active_limit_orders。
  - 引擎 `cross_order` 撮合后经 `strategy.update_trade`/`update_order` 回推。
  - `calculate_result`/`calculate_statistics` 产出 daily_df 与绩效指标，`show_performance` 画净值曲线。
- **相邻叶子指引**：信号生成见 alpha-model；数据加载与持久化见 alpha-lab；因子计算见 alpha-dataset。

## 9. 语言专项适配口径（纯 Python：能力缝 + 注册表 + 插件体系）

- **能力缝**：核心能力缝是"**策略抽象基类 + 回调钩子**"。`AlphaStrategy`（ABCMeta，`template.py:15`）定义 on_init/on_bars/on_trade 三个抽象回调，内置 send_order/buy/sell/short/cover/set_target/execute_trading 等交易辅助方法；用户子类化实现 on_bars 即完成策略——这是典型的"模板方法模式 + 回调钩子"。
- **引擎-策略双向解耦**：策略持有 `strategy_engine` 引用（`template.py:26`），策略调引擎的 send_order/cancel_order/get_signal；引擎调策略的 on_bars/update_trade/update_order——双向依赖但通过接口方法通信。
- **配置注入**：策略参数经 `setting` dict 在构造时 `setattr` 注入（`template.py:39-41`），类属性（如 EquityDemo 的 top_k=50）即默认值。
- **目标仓位模式**：策略不直接下单量，而是 `set_target(vt_symbol, target)`，由 `execute_trading` 统一计算差额并拆成 cover/buy 或 sell/short 限价单（`template.py:133`）——这是"目标仓位→订单"的策略模式。
- **回测状态机**：run_backtesting 是"加载→初始化→逐根回放→撮合成交→逐日盯市→统计"的循环；每根 bar 内 cross_order→on_bars→update_close 是固定阶段，补 lifecycle 图表达回测引擎状态变迁。
- **图型侧重**：回测主循环是典型的"单实体（回测引擎）状态推进"，补 lifecycle 图；architecture 图表达策略-引擎-数据-结果组件关系。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 策略回测架构图 | [alpha-strategy-architecture.html](alpha-strategy-architecture.html) | architecture | standard |
| 回测引擎生命周期图 | [alpha-strategy-lifecycle.html](alpha-strategy-lifecycle.html) | lifecycle | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 本叶子未生成 sequence 图：撮合/回放是引擎单线程循环，无多方消息时序；未生成 dataflow 图：数据流与 alpha-lab 重叠；未生成 workflow 图：调仓顺序在 template.py 已是固定流程。
