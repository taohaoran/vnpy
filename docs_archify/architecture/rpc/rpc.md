# RPC 通信（rpc）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`vnpy` 4.4.0（master），commit `fa5206fe`。

## 1. 域职责

`vnpy.rpc` 是 vnpy 框架的跨进程通信层，基于 pyzmq（ZeroMQ）实现两种通信范式：

- **请求-应答（REQ/REP）**：把本地 MainEngine 的函数暴露为远程可调用方法，远程进程通过 `RpcClient` 透明代理跨进程调用，实现"像调本地函数一样调远端引擎接口"。
- **发布-订阅（PUB/SUB）**：服务端主动向所有订阅进程推送带主题的消息（行情 tick、委托/持仓变化、日志等），客户端覆写 `callback(topic, data)` 接收。
- **心跳保活**：服务端每 10 秒广播心跳，客户端 30 秒收不到任何订阅消息即触发断线告警，用于跨进程健康检测。

本域在整个 vnpy 架构中定位为"**进程间通信骨架**"——它不实现任何交易/行情业务逻辑，只提供序列化传输与方法路由能力，典型部署是把 MainEngine 跑在一个进程、GUI/策略跑在另一个进程，通过 RPC 桥接。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 / 数据流图 | 职责一句话 |
|---|---|---|---|---|
| rpc-communication | [rpc-communication.md](rpc-communication/rpc-communication.md) | [架构图](rpc-communication/rpc-communication-architecture.html) | [RPC 时序图](rpc-communication/rpc-communication-sequence.html) | pyzmq 远程过程调用 + 发布订阅通信骨架 |

## 3. 域级机制细节

- **双 socket 通道分离**：RpcServer 同时持有 REP（同步应答）与 PUB（异步广播）两个 socket（`vnpy/rpc/server.py:25-28`）；RpcClient 对应持有 REQ 与 SUB。请求与推送走不同物理通道，互不阻塞——这是本域的核心设计取舍：同步 RPC 不阻塞行情/事件推送。
- **函数注册表 + 动态代理**：服务端 `_functions: dict[str, Callable]` 以函数名为键（`server.py:19,123`）；客户端 `__getattr__` + `@lru_cache(100)` 把任意属性访问转为远程调用闭包（`client.py:55-86`），调用方无需 stub 代码。
- **异常透传而非重试**：服务端 `except Exception` 兜底后把 `traceback` 序列化回客户端（`server.py:106-107`），客户端包装为 `RemoteException`（`client.py:11`）；超时由调用方自行决定是否重试，框架不做自动重试。
- **心跳与断线**：`HEARTBEAT_INTERVAL=10s`（服务端广播周期）、`HEARTBEAT_TOLERANCE=30s`（客户端无消息即判断线），常量集中在 `common.py:8-10`。
- **线程模型**：服务端与客户端各起一个工作线程处理各自 socket；客户端 REQ 通道用 `threading.Lock` 保护"发送-等待-接收"三步原子性，REP 协议要求严格交替。
- **序列化**：统一用 `send_pyobj`/`recv_pyobj`（基于 pickle），仅支持 Python 进程间通信，不跨语言。

## 4. 域级图

![RPC 域架构图](rpc-architecture.html)
![RPC 数据流图](rpc-dataflow.html)
![心跳与订阅推送时序图](rpc-sequence.html)

- 三张域级图均为 showcase 档；JSON IR 位于 `json/` 目录。
- 本域未生成 workflow 图：无多角色带泳道的审批/分步流程；未生成 lifecycle 图：RpcServer/RpcClient 仅 active 布尔翻转，无多阶段状态机。
