# 预测模型（alpha-model）

> 本文是 `alpha` 域下的叶子子系统文档。域级总览见 `../alpha.md`。
>
> 源码基准：`vnpy` 4.4.0（master），commit `fa5206fe`；纯 Python + scikit-learn/LightGBM/PyTorch，主语言口径见本文第 9 节。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 模型抽象基类 | `AlphaModel`（ABCMeta）：定义 `fit(dataset)` / `predict(dataset, segment)` 契约，`detail()` 可选 | `vnpy/alpha/model/template.py:9` |
| Lasso 线性回归 | sklearn Lasso 实现，TRAIN+VALID 合并训练，非零系数特征排序 | `vnpy/alpha/model/models/lasso_model.py:13` |
| LightGBM 集成树 | lgb.Booster，TRAIN/VALID 早停，mse 目标 | `vnpy/alpha/model/models/lgb_model.py:12` |
| MLP 神经网络 | PyTorch 多层感知机，Adam/SGD、batch 训练、early stopping | `vnpy/alpha/model/models/mlp_model.py:22` |
| 训练数据适配 | 从 dataset.fetch_learn(TRAIN/VALID) 取 X/y，列 2:-1 为特征、末列为 label | `lasso_model.py:50-63`、`lgb_model.py:70-80` |
| 预测 | 从 dataset.fetch_infer(segment) 取特征矩阵 predict，返回 np.ndarray | `lasso_model.py:101-108` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `AlphaModel`（ABCMeta） | `template.py:9` | 模型契约：`fit`/`predict` 抽象方法，`detail` 默认空 |
| `LassoModel` | `lasso_model.py:13` | sklearn Lasso 封装；alpha=0.0005、max_iter=1000、fit_intercept=False |
| `LgbModel` | `lgb_model.py:12` | LightGBM 封装；num_leaves=31、num_boost_round=1000、early_stopping=50 |
| `MlpModel` | `mlp_model.py:22` | PyTorch MLP；hidden=(256,)、lr=0.001、epochs=300、batch=2000、early_stop=50 |
| `MlpNetwork` | `mlp_model.py`（nn.Module） | MLP 网络结构（Linear+激活堆叠） |

## 3. 关键调用链

**调用链 1：Lasso 训练**

1. `LassoModel(alpha=0.0005).fit(dataset)`（`lasso_model.py:40`）：`fetch_learn(TRAIN)` 与 `fetch_learn(VALID)` 取出后 `pl.concat` 合并、按 (datetime, vt_symbol) 去重排序（`lasso_model.py:50-56`）。
2. 特征列 = `df.columns[2:-1]`（去掉 datetime/vt_symbol 主键，去掉末列 label）（`lasso_model.py:59`）；X=特征 to_numpy，y=label（`lasso_model.py:62-63`）。
3. `Lasso(alpha, max_iter, random_state, fit_intercept=False, copy_X=False).fit(X, y)`（`lasso_model.py:66-73`）。
4. `detail()` 打印非零系数按绝对值排序（`lasso_model.py:112-139`）。

**调用链 2：LightGBM 训练**

1. `LgbModel(...).fit(dataset)`（`lgb_model.py:84`）：`_prepare_data` 把 TRAIN/VALID 分别包成 `lgb.Dataset`（`lgb_model.py:53-82`）。
2. 调 `lgb.train(params, dtrain, num_boost_round, valid_sets=[dvalid], early_stopping_rounds, callbacks)` 训练（`lgb_model.py:84` 后续）。
3. `predict` 时 `fetch_infer(segment)` 取特征 to_pandas，`self.model.predict(data)` 返回（`lasso_model.py:101-108` 同构）。

**调用链 3：MLP 训练循环**

1. `MlpModel(input_size, hidden_sizes, lr, n_epochs, batch_size, early_stop_rounds, optimizer, device).fit(dataset)`（`mlp_model.py:35-120`）：建 MlpNetwork、Adam/SGD 优化器、MSE loss。
2. 按 batch 遍历训练集，每 eval_steps 步在验证集评估，验证 loss 连续 early_stop_rounds 不改善则早停（`mlp_model.py` fit 循环）。
3. `predict` 时模型 eval 模式下前向推理返回 np.ndarray。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| Lasso `alpha` | 0.0005 正则强度；max_iter=1000；fit_intercept=False | `lasso_model.py:18-20,66-71` |
| LGB `learning_rate` | 0.1；num_leaves=31；num_boost_round=1000；early_stopping_rounds=50 | `lgb_model.py:17-22,40-49` |
| LGB `objective` | "mse" 回归 | `lgb_model.py:41` |
| MLP `hidden_sizes` | (256,)；lr=0.001；n_epochs=300；batch_size=2000 | `mlp_model.py:38-42` |
| MLP `early_stop_rounds` | 50；eval_steps=20；optimizer="adam"；device="cpu" | `mlp_model.py:43-47` |
| 特征列约定 | `columns[2:-1]`（datetime/vt_symbol 之后、label 之前） | `lasso_model.py:59,105` |

## 5. 错误与重试语义

- **未训练即预测**：`LassoModel.predict` 检查 `self.model is None` 抛 `ValueError("model is not fitted yet!")`（`lasso_model.py:97-98`）。
- **LightGBM 早停**：验证集指标不改善时自动停止 boost，无异常。
- **MLP 早停**：验证 loss 连续 early_stop_rounds 轮不改善则停止训练，保留 best_step 权重。
- **无重试**：训练/预测为同步 CPU/GPU 计算，无网络重试；数据缺失由上游 dataset 预处理保证。

## 6. 并发细节

- **单线程训练**：三个模型均同步训练，无工作线程；MLP 在 CPU/GPU 上跑 batch 循环。
- **无锁**：模型对象在 fit 后即冻结，predict 只读。
- **多进程边界**：因子并行在 dataset 叶子（prepare_data），模型训练单进程；MLP 可通过 device="cuda" 用 GPU（不在本仓库源码内）。
- **取消传播**：MLP 早停是训练循环内的状态退出，无外部取消令牌。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `vnpy/alpha/model/template.py`：AlphaModel 抽象基类
- `vnpy/alpha/model/models/lasso_model.py`：Lasso 实现
- `vnpy/alpha/model/models/lgb_model.py`：LightGBM 实现
- `vnpy/alpha/model/models/mlp_model.py`：PyTorch MLP 实现

**Out-of-Scope（不在本仓库源码内）**
- **scikit-learn**：`Lasso` 线性回归（`lasso_model.py:3`，不在本仓库源码内）。
- **LightGBM**：`lgb.Booster`/`lgb.Dataset`/`lgb.train`（`lgb_model.py:5`，不在本仓库源码内）。
- **PyTorch**：`torch.nn`/`optim`/MlpNetwork 训练与推理（`mlp_model.py:9-11`，不在本仓库源码内）。
- **matplotlib**：LGB 特征重要性绘图（`lgb_model.py:6`，不在本仓库源码内）。
- **特征工程**：见 alpha-dataset 叶子；本叶子只消费 fetch_learn/fetch_infer 产出的矩阵。
- **回测/信号**：见 alpha-strategy 叶子；本叶子产出预测 np.ndarray，由上层组装为 signal_df。

## 8. 与相邻子系统交互

- **上游 → 本叶子**：AlphaDataset（alpha-dataset 叶子）经 `fetch_learn(TRAIN/VALID)` 提供训练矩阵与 label，经 `fetch_infer(segment)` 提供推理特征。
- **本叶子 → 下游**：`predict(dataset, segment)` 返回 `np.ndarray` 预测值，由用户组装为 `signal_df`（datetime/vt_symbol/signal）传给 `BacktestingEngine.add_strategy(strategy_class, setting, signal_df)`（见 alpha-strategy 叶子）。
- **持久化**：训练好的模型经 `AlphaLab.save_model(name, model)` pickle 落盘（见 alpha-lab 叶子）。

## 9. 语言专项适配口径（纯 Python：能力缝 + 注册表 + 插件体系）

- **能力缝**：核心能力缝是"**模型抽象基类 + 多后端矩阵**"。`AlphaModel`（ABCMeta，`template.py:9`）用 `fit`/`predict` 两个抽象方法定义"训练-预测"契约，三个实现（Lasso/LGB/MLP）分别封装 sklearn/lightgbm/torch 三套后端——这是纯 Python 框架典型的"统一基类契约 + 多后端实现矩阵"。
- **注册表**：本叶子**没有**字符串→模型类路由（不像 transformers AutoModel）；`model/__init__.py` 只导出 `AlphaModel`（`model/__init__.py:4-6`），具体模型类需用户显式 `from vnpy.alpha.model.models.lasso_model import LassoModel`。扩展新模型 = 子类化 AlphaModel 实现 fit/predict，无自动注册。
- **配置面**：每个模型的超参在 `__init__` 签名里（dataclass 风格但用普通属性），LGB 把参数收进 `self.params` dict（`lgb_model.py:40-45`），MLP 保留全部超参为属性。
- **双轨/后端切换**：同一 AlphaModel API 下可换 sklearn/LGB/torch 后端——由用户在构造时选类，非配置驱动运行时切换。
- **图型侧重**：fit/predict 是训练-预测两阶段循环，MLP 训练含 early stopping 状态机，补 lifecycle 图表达"未训练→训练中→早停/完成→可预测"的模型状态变迁；architecture 图表达三后端矩阵。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 预测模型架构图 | [alpha-model-architecture.html](alpha-model-architecture.html) | architecture | standard |
| 模型训练生命周期图 | [alpha-model-lifecycle.html](alpha-model-lifecycle.html) | lifecycle | standard |

- JSON IR 源文件位于 `json/` 目录。
- 本叶子未生成 sequence 图：fit/predict 是单模型对 dataset 的同步调用，无多方消息交互；未生成 dataflow 图：训练数据流向与 alpha-dataset 重叠；未生成 workflow 图：无带泳道流程。
