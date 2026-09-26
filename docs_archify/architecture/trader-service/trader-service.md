# trader-service 域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：vnpy 4.4.0，commit `fa5206fe`。

## 1. 域职责

trader-service 域是 vnpy 核心交易框架的**横切服务层**，为上层引擎与策略提供五类可复用能力：
- **配置与日志**（config-utility）：全局 `SETTINGS` 加载、loguru 日志装配、K 线合成与技术指标工具；
- **数据持久化抽象**（database-interface）：`BaseDatabase` 契约 + 按配置字符串路由到外部 `vnpy_*` 数据库驱动；
- **历史数据服务抽象**（datafeed-interface）：`BaseDatafeed` 软契约 + 路由到外部行情服务驱动；
- **微信通知**（notify-wechat）：iLink 机器人协议的扫码绑定与消息推送；
- **参数优化**（optimizer）：穷举 / 遗传算法两种并行参数寻优。

本域的共性架构范式是**纯 Python 的"能力缝 + 注册表 + 插件体系"**：接口契约定义在本仓库，具体后端实现分散在独立发行的 `vnpy_*` 扩展包，运行时由配置字符串动态 import 路由。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序/数据流/生命周期图 | 职责一句话 |
|------|------|--------|-------------------|-----------|
| config-utility | [config-utility.md](config-utility/config-utility.md) | [架构图](config-utility/config-utility-architecture.html) | [数据流](config-utility/config-utility-dataflow.html) | 全局配置/日志/K线合成/指标工具 |
| database-interface | [database-interface.md](database-interface/database-interface.md) | [架构图](database-interface/database-interface-architecture.html) | [时序](database-interface/database-interface-sequence.html) | 行情落库抽象基类与驱动工厂 |
| datafeed-interface | [datafeed-interface.md](datafeed-interface/datafeed-interface.md) | [架构图](datafeed-interface/datafeed-interface-architecture.html) | [时序](datafeed-interface/datafeed-interface-sequence.html) | 历史行情服务抽象基类与驱动工厂 |
| notify-wechat | [notify-wechat.md](notify-wechat/notify-wechat.md) | [架构图](notify-wechat/notify-wechat-architecture.html) | [时序](notify-wechat/notify-wechat-sequence.html) | 微信 iLink 扫码绑定与消息推送 |
| optimizer | [optimizer.md](optimizer/optimizer.md) | [架构图](optimizer/optimizer-architecture.html) | [生命周期](optimizer/optimizer-lifecycle.html) | 穷举/遗传算法并行参数寻优 |

## 3. 域级机制细节

- **统一的"配置字符串 → 动态 import → 实例化"工厂模式**：`get_database()`（`database.py:139`）与 `get_datafeed()`（`datafeed.py:39`）是同构的懒加载单例工厂——读 `SETTINGS["*.name"]` → `import_module("vnpy_<name>")` → `module.<Class>()`。区别仅在降级策略：database 兜底 SQLite，datafeed 回退空基类。
- **横切配置面**：`SETTINGS`（`setting.py:11`）在 import 时从 `vt_setting.json` 加载，被 logger/database/datafeed/UI 共同读取，是全框架配置中枢。
- **多进程并行**：optimizer 是本域唯一的多进程点（`ProcessPoolExecutor`/`multiprocessing.Pool` + `Manager.dict()` 共享缓存），其余模块均为同步单线程。
- **外部边界集中**：本域所有"真正干活"的后端（数据库驱动、行情服务、微信服务器、TA-Lib、deap）都不在本仓库源码内，本域只提供契约、路由与胶水。

## 4. 域级图

![trader-service 域架构图](trader-service-architecture.html)
![trader-service 数据服务管道](trader-service-dataflow.html)
![trader-service 启动装配时序](trader-service-sequence.html)
