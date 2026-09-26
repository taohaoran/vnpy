# 参数优化器（optimizer）

> 本文是 `trader-service` 域下的叶子子系统文档。域级总览见 `../trader-service.md`。
> 本文只展开 vnpy 的参数优化配置与两种优化算法（穷举 / 遗传算法）的并行执行框架，不重复展开各策略回测 `evaluate_func` 的内部实现（由调用方策略模板提供，不在本仓库 trader 包内）。
>
> 源码基准：vnpy 4.4.0，commit `fa5206fe`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 优化参数配置 | `OptimizationSetting`：`params` dict 记录每个参数的候选值列表，`target_name` 指定优化目标 | `vnpy/trader/optimize.py:26` |
| 添加参数 | `add_parameter(name, start, end, step)`：固定参数（end/step 缺省）或范围参数（按 step 展开为候选列表），带参数校验 | `optimize.py:36` |
| 目标设置 | `set_target(target_name)` | `optimize.py:65` |
| 全量组合生成 | `generate_settings()`：对所有参数候选做笛卡尔积，产出 `list[dict]` | `optimize.py:69` |
| 优化前校验 | `check_optimization_setting`：组合非空且目标已设置 | `optimize.py:83` |
| 穷举优化 | `run_bf_optimization`：`ProcessPoolExecutor` 并行评估全部组合，tqdm 进度条，按 key_func 排序 | `optimize.py:99` |
| 遗传算法优化 | `run_ga_optimization`：基于 deap 的 `eaMuPlusLambda`，多进程池 + Manager 共享结果缓存 | `optimize.py:132` |
| 个体评估回调 | `ga_evaluate(cache, evaluate_func, key_func, parameters)`：带缓存的单个体评估，避免重复计算 | `optimize.py:232` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `OptimizationSetting` | `optimize.py:26` | 参数空间配置器；`generate_settings()` 产出待评估组合 |
| `run_bf_optimization` | `optimize.py:99` | 穷举入口；签名 `(evaluate_func, optimization_setting, key_func, max_workers, output)` |
| `run_ga_optimization` | `optimize.py:132` | 遗传算法入口；额外参数 `pop_size=100/ngen=30/cxpb=0.95/mutpb/indpb=1.0` |
| `ga_evaluate` | `optimize.py:232` | 进程池 worker 调用的个体评估函数，带 `DictProxy` 缓存 |
| `evaluate_func`（回调） | `optimize.py:17` | `Callable[[dict], dict]`：传入参数 dict，返回回测结果 dict（由策略回测模板提供） |
| `key_func`（回调） | `optimize.py:18` | `Callable[[tuple], float]`：从结果 dict 提取优化目标值（如夏普比率） |

> deap 个体在模块 import 时创建：`creator.create("FitnessMax", base.Fitness, weights=(1.0,))` 与 `creator.create("Individual", list, fitness=...)`（`optimize.py:22`–`23`），即最大化单目标。

## 3. 关键调用链

**调用链一：穷举优化（`run_bf_optimization`）**
1. `optimize.py:107`：`generate_settings()` 产出全部参数组合。
2. `optimize.py:114`–`117`：建 `ProcessPoolExecutor(max_workers, mp_context=get_context("spawn"))`。
3. `optimize.py:118`–`122`：`executor.map(evaluate_func, settings)` 包进 tqdm，收集 `list(tuple)` 结果。
4. `optimize.py:123`：`results.sort(reverse=True, key=key_func)` 按目标值降序。

**调用链二：遗传算法主循环（`run_ga_optimization`）**
1. `optimize.py:148`–`149`：把全量组合转成 `parameter_tuples` 列表；`generate_parameter` 从中 `choice` 一个作为个体基因来源（`optimize.py:151`）。
2. `optimize.py:165`–`168`：`ctx.Manager()` + `ctx.Pool(max_workers)`，建共享结果缓存 `cache = manager.dict()`（`DictProxy`）。
3. `optimize.py:171`–`184`：注册 deap toolbox：`individual`/`population`/`mate=cxTwoPoint`/`mutate=mutate_individual`/`select=selNSGA2`/`map=pool.map`/`evaluate=ga_evaluate`。
4. `optimize.py:211`–`220`：`algorithms.eaMuPlusLambda(pop, toolbox, mu, lambda_, cxpb, mutpb, ngen, verbose=True)` 驱动整个进化迭代。
5. `optimize.py:227`–`228`：从 `cache.values()` 取全部评估结果，按 key_func 降序返回。

**调用链三：单个体评估与缓存（`ga_evaluate`）**
1. `optimize.py:241`：把参数列表转 tuple 作缓存键。
2. `optimize.py:242`–`247`：缓存命中直接复用；未命中则 `evaluate_func(setting)` 并写入共享缓存——避免 GA 迭代中重复评估同一组合。
3. `optimize.py:249`–`250`：返回 `(key_func(result),)` 适配 deap FitnessMax。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `pop_size` | 100，每代个体数 | `optimize.py:137` |
| `ngen` | 30，进化代数 | `optimize.py:138` |
| `mu` | None→`int(pop_size*0.8)`，每代选择数 | `optimize.py:139`、`optimize.py:188` |
| `lambda_` | None→`pop_size`，每代后代数 | `optimize.py:140`、`optimize.py:191` |
| `cxpb` | 0.95，交叉概率 | `optimize.py:141` |
| `mutpb` | None→`1-cxpb`，变异概率 | `optimize.py:142`、`optimize.py:194` |
| `indpb` | 1.0，每个基因独立变异概率 | `optimize.py:143` |
| `max_workers` | None（由 ProcessPool 默认 CPU 核数） | `optimize.py:103` |

## 5. 错误与重试语义

- `add_parameter` 校验：`start >= end` 返回 `(False, "参数优化起始点必须小于终止点")`（`optimize.py:48`）；`step <= 0` 返回 `(False, "参数优化步进必须大于0")`（`optimize.py:51`）——返回元组而非抛异常。
- `check_optimization_setting`：组合为空或目标未设置时通过 `output` 打印并返回 False（`optimize.py:88`–`95`）。
- 本模块**不重试单个 `evaluate_func` 失败**：worker 内回测异常会随进程池向上传播；结果缓存只对成功评估生效。
- 无退避策略；并行失败即整体失败。

## 6. 并发细节

- **多进程而非多线程**：穷举用 `ProcessPoolExecutor(mp_context=get_context("spawn"))`（`optimize.py:114`）；GA 用 `ctx.Pool(max_workers)`（`optimize.py:166`）——均显式用 `spawn` 上下文，避免回测对象 fork 继承状态问题。
- **共享状态**：GA 用 `ctx.Manager().dict()`（`DictProxy`，`optimize.py:168`）做跨进程结果缓存，`pool.map` 把 `ga_evaluate` 分发到各 worker。
- **进程间通信**：deap `toolbox.register("map", pool.map)`（`optimize.py:177`）覆盖默认 map，使 `eaMuPlusLambda` 的评估并行化。
- **无锁**：缓存写由 Manager 代理串行化；主进程单线程驱动 `eaMuPlusLambda` 迭代。
- 进度条 tqdm 在穷举路径包裹 `executor.map`（`optimize.py:118`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `vnpy/trader/optimize.py`：`OptimizationSetting`、穷举/GA 两个优化入口、`ga_evaluate` 缓存回调。

**Out-of-Scope（不在本仓库源码内）**
- deap 进化算法库（`creator/base/tools/algorithms`）——不在本仓库源码内。
- tqdm 进度条库——不在本仓库源码内。
- `evaluate_func`（策略回测逻辑）——由调用方（cta_strategy 等 App 的回测引擎）提供，不在本文件。
- numpy/pandas 等回测依赖——不在本文件。

## 8. 与相邻子系统交互

- **上游调用方**：各策略 App（如 cta_strategy 的 `BacktestingEngine`）构造 `OptimizationSetting`、定义 `evaluate_func`/`key_func`，调 `run_bf_optimization` 或 `run_ga_optimization`。
- **下游依赖**：`optimize.py` 仅依赖 deap/tqdm/标准库（`concurrent.futures`/`multiprocessing`/`itertools.product`），不直接依赖 vnpy 其他模块。
- **交互方向**：调用方 → OptimizationSetting（定义搜索空间）→ run_*（多进程并行 evaluate_func）→ 排序后的结果列表返回调用方。

## 9. 语言专项适配口径（纯 Python：能力缝 + 注册表 + 插件体系）

- **能力缝（回调注入式并行评估）**：本叶子是典型的"并行执行引擎 + 回调注入"能力缝——`run_bf/run_ga` 不关心回测内部，只通过 `evaluate_func: Callable[[dict], dict]`（`optimize.py:17`）这一注入回调驱动，搜索空间由 `OptimizationSetting` 配置对象描述。
- **deap 注册表/工具箱模式**：`base.Toolbox()` 上 `register` 各算子（individual/mate/mutate/select/map/evaluate，`optimize.py:171`–`184`）——这是 deap 框架的标准注册表扩展点；vnpy 用 `pool.map` 覆盖默认 `map` 以接入多进程。
- **import 时副作用**：`creator.create(...)`（`optimize.py:22`）在模块加载时创建 `FitnessMax`/`Individual` 类，属 import 时注册。
- **多后端矩阵（双算法）**：同一优化问题提供穷举（bf）与遗传算法（ga）两套实现，共享 `OptimizationSetting` 与 `evaluate_func`/`key_func` 契约——配置驱动 + 双轨算法。
- **共享内存缓存**：`Manager.dict()` 跨进程缓存（`optimize.py:168`）是纯 Python 多进程工程的典型模式，避免 GA 进化中重复评估同一参数组合。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| optimizer 组件架构图 | `optimizer-architecture.html` | architecture | showcase |
| 遗传算法进化生命周期图 | `optimizer-lifecycle.html` | lifecycle | standard |

JSON IR 位于 `json/` 目录。
lifecycle 图落 **standard** 档：showcase 校验报 `label_route_clearance`（相邻状态列间距小，"评估完成"/"产生后代" 等迁移标签与相邻状态框横向重叠）；已采取修复动作——加宽 viewBox 至 1500、全部迁移标签 `labelDy=80` 下移到状态行下方；showcase 仍因状态横向排布固定间距无法完全消除标签与状态框的垂直投影重叠，按兜底策略降 standard 渲染（render 退出码 0、HTML 非空）。
本叶子未生成 sequence / dataflow / workflow 图：GA 进化是"单一过程对象多代迭代"的状态机（已由 lifecycle 表达），其内部 worker 评估调用链与穷举路径同构，文字已在第 3 节描述；无独立数据 ETL 管道。
