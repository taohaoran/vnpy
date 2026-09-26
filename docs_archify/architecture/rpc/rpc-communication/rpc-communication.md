# RPC 远程通信（rpc-communication）

> 本文是 `rpc` 域下的叶子子系统文档。域级总览见 `../rpc.md`。
>
> 源码基准：`vnpy` 4.4.0（master），commit `fa5206fe`；纯 Python，主语言口径见 `../..` 语言专项。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 远程过程调用（请求-应答） | 客户端把"函数名+参数"序列化为 Python 对象发往服务端，服务端查表执行后回传结果或异常堆栈 | `vnpy/rpc/server.py:83`（`RpcServer.run`）、`vnpy/rpc/client.py:56`（`RpcClient.__getattr__`） |
| 函数注册表 | 服务端通过 `@register` 把本地可调用对象按 `__name__` 登记进字典，客户端按名字透明远程调用 | `vnpy/rpc/server.py:123`（`RpcServer.register`） |
| 发布-订阅广播 | 服务端可主动向所有订阅者推送带主题的消息（如行情、持仓变化） | `vnpy/rpc/server.py:116`（`RpcServer.publish`）、`vnpy/rpc/client.py:158`（`RpcClient.subscribe_topic`） |
| 心跳保活与断线检测 | 服务端每 10 秒广播一次心跳主题；客户端在 30 秒容忍窗口内收不到任何订阅消息即判定断线回调 | `vnpy/rpc/common.py:8`（常量）、`vnpy/rpc/server.py:129`（`check_heartbeat`）、`vnpy/rpc/client.py:135`（断线判定） |
| 远程异常透传 | 服务端把异常 `traceback` 序列化回客户端，包装为 `RemoteException` 抛出 | `vnpy/rpc/server.py:106`、`vnpy/rpc/client.py:11`（`RemoteException`） |
| 透明代理远程方法 | 客户端用 `__getattr__` 动态生成远程函数闭包，调用方形如 `client.xxx(args)` 即可跨进程调用 | `vnpy/rpc/client.py:55`（`@lru_cache(100)` 缓存闭包） |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `RpcServer` | `vnpy/rpc/server.py:11` | 服务端：持有函数注册表、REP/PUB 两个 ZeroMQ socket、工作线程与心跳计时器 |
| `RpcServer.register(func)` | `vnpy/rpc/server.py:123` | 扩展点：把本地函数登记为可被远程调用的服务方法（注册表键 = `func.__name__`） |
| `RpcServer.publish(topic, data)` | `vnpy/rpc/server.py:116` | 扩展点：业务侧主动推送带主题数据（加锁后经 PUB socket 发出） |
| `RpcServer.run()` | `vnpy/rpc/server.py:83` | 服务端工作线程主循环：poll REP → 解包请求 → 查表执行 → 回包 |
| `RpcClient` | `vnpy/rpc/client.py:29` | 客户端：持有 REQ/SUB socket、工作线程；`__getattr__` 动态生成远程调用闭包 |
| `RpcClient.__getattr__(name)` | `vnpy/rpc/client.py:56` | 核心能力缝：未定义属性即视为远程函数名，返回 `dorpc` 闭包（`@lru_cache(100)` 缓存） |
| `RpcClient.callback(topic, data)` | `vnpy/rpc/client.py:152` | 扩展点（模板方法）：默认 `raise NotImplementedError`，子类/使用者覆写以处理推送消息 |
| `RpcClient.on_disconnected()` | `vnpy/rpc/client.py:164` | 扩展点：心跳丢失时的回调，默认打印告警 |
| `RemoteException` | `vnpy/rpc/client.py:11` | 异常类型：封装远端返回的错误信息（超时或服务端异常堆栈） |
| `HEARTBEAT_TOPIC / HEARTBEAT_INTERVAL / HEARTBEAT_TOLERANCE` | `vnpy/rpc/common.py:8-10` | 常量：心跳主题 `"heartbeat"`、间隔 10s、容忍 30s |

## 3. 关键调用链

**调用链 1：一次同步远程调用（客户端主动）**

1. 业务侧执行 `client.foo(a, b=1)`。`foo` 在客户端本不存在，触发 `RpcClient.__getattr__("foo")`（`vnpy/rpc/client.py:56`），经 `@lru_cache(100)` 命中或新建闭包 `dorpc`。
2. `dorpc` 从 kwargs 弹出 `timeout`（默认 30000ms，`vnpy/rpc/client.py:63`），组装请求 `[name, args, kwargs]`（`vnpy/rpc/client.py:66`）。
3. 在 `self._lock` 保护下 `send_pyobj(req)` 发送（`vnpy/rpc/client.py:70`），随后 `self._socket_req.poll(timeout)` 等待（`vnpy/rpc/client.py:73`）；超时未收包则 `raise RemoteException`（`vnpy/rpc/client.py:76`）。
4. 服务端工作线程 `run()` 在 `self._socket_rep.poll(1000)` 后 `recv_pyobj()` 取出请求（`vnpy/rpc/server.py:89-96`），解包 `name, args, kwargs`（`vnpy/rpc/server.py:99`）。
5. 查表 `func = self._functions[name]` 并执行 `func(*args, **kwargs)`（`vnpy/rpc/server.py:103-104`）；成功回 `[True, r]`，异常回 `[False, traceback.format_exc()]`（`vnpy/rpc/server.py:105-107`），经 `send_pyobj` 回包（`vnpy/rpc/server.py:110`）。
6. 客户端 `recv_pyobj()` 后按 `rep[0]` 分流：True 则返回 `rep[1]`，False 则 `raise RemoteException(rep[1])`（`vnpy/rpc/client.py:81-84`）。

**调用链 2：服务端主动推送（发布-订阅）**

1. 业务侧调用 `server.publish(topic, data)`，在锁内 `self._socket_pub.send_pyobj([topic, data])`（`vnpy/rpc/server.py:120-121`）。
2. 客户端工作线程 `run()` 在 `self._socket_sub.poll(pull_tolerance)`（容忍窗口 30s）后 `recv_pyobj(flags=zmq.NOBLOCK)` 取包（`vnpy/rpc/client.py:135-140`）。
3. 若 `topic == HEARTBEAT_TOPIC` 则刷新 `_last_received_ping`；否则调用 `self.callback(topic, data)` 交给使用者（`vnpy/rpc/client.py:142-146`）。

**调用链 3：心跳与断线检测**

1. 服务端 `start()` 时初始化 `_heartbeat_at = time() + HEARTBEAT_INTERVAL`（`vnpy/rpc/server.py:65`）。
2. 每轮 `run()` 末尾调用 `check_heartbeat()`（`vnpy/rpc/server.py:90`）；到点即 `publish(HEARTBEAT_TOPIC, now)` 并推进时间戳（`vnpy/rpc/server.py:135-140`）。
3. 客户端若在 `pull_tolerance = HEARTBEAT_TOLERANCE*1000`（30s）内 SUB socket 无任何消息，则调用 `on_disconnected()` 打印告警（`vnpy/rpc/client.py:132-136, 164-169`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `rep_address` / `pub_address` | `start()` 入参，服务端 bind 地址（如 `tcp://*:2014` / `tcp://*:2015`） | `vnpy/rpc/server.py:42-55` |
| `req_address` / `sub_address` | `start()` 入参，客户端 connect 地址 | `vnpy/rpc/client.py:88-101` |
| `timeout`（dorpc kwarg） | 默认 30000ms，单次远程调用等待应答上限 | `vnpy/rpc/client.py:63` |
| `HEARTBEAT_INTERVAL` | 10 秒，服务端心跳广播周期 | `vnpy/rpc/common.py:9` |
| `HEARTBEAT_TOLERANCE` | 30 秒，客户端无订阅消息即判定断线 | `vnpy/rpc/common.py:10` |
| `TCP_KEEPALIVE` / `TCP_KEEPALIVE_IDLE` | 客户端 REQ/SUB 均开启 TCP keepalive，空闲 60s 探测 | `vnpy/rpc/client.py:44-46` |
| `@lru_cache(100)` | 客户端缓存 100 个远程方法闭包，避免重复构造 | `vnpy/rpc/client.py:55` |
| `SIGINT = SIG_DFL` | common 模块导入时把 SIGINT 恢复为默认处理，使 Ctrl-C 能中断阻塞在 recv 上的线程 | `vnpy/rpc/common.py:5` |

## 5. 错误与重试语义

- **远端函数异常**：服务端用 `except Exception` 兜底（`vnpy/rpc/server.py:106`，注释 `# noqa`），不中断工作循环，把 `traceback.format_exc()` 作为 `[False, err]` 回传；客户端收到后包装为 `RemoteException` 抛出（`vnpy/rpc/client.py:84`）。**无自动重试**。
- **调用超时**：客户端 `poll(timeout)` 为 0 时抛 `RemoteException("Timeout of ...")`（`vnpy/rpc/client.py:74-76`），不重试，由调用方决定是否重试。
- **断线检测**：客户端 SUB 30s 无消息触发 `on_disconnected()`，默认仅 `print` 告警，**不自动重连**；`_last_received_ping` 被刷新但未在代码中进一步用于重连逻辑。
- **REP  socket 顺序约束**：ZeroMQ REP 必须严格"收一请求→发一应答"，本实现每轮循环恰好一次 recv + 一次 send，保持协议约束；`poll(1000)` 超时轮询用于及时感知 `_active=False` 退出。
- **未注册函数**：`self._functions[name]` 若 KeyError，会被 `except Exception` 捕获并序列化为远端异常回客户端——即"未注册方法"也表现为 `RemoteException`。

## 6. 并发细节

- **服务端工作线程**：`start()` 创建 `threading.Thread(target=self.run)`（`vnpy/rpc/server.py:61`）；`stop()` 仅置 `_active=False`（`vnpy/rpc/server.py:75`），依赖 `run()` 中 `poll(1000)` 每秒醒来检查退出；`join()` 等待线程结束后关闭 socket（`vnpy/rpc/server.py:77-81, 113-114`）。
- **客户端工作线程**：`start()` 创建 `threading.Thread(target=self.run)`（`vnpy/rpc/client.py:107`）；退出同样靠 `_active` 标志 + `poll(pull_tolerance)` 唤醒；退出时关闭 REQ/SUB socket（`vnpy/rpc/client.py:148-150`）。
- **锁**：服务端 `self._lock`（`vnpy/rpc/server.py:33`）仅保护 `publish()` 对 PUB socket 的并发发送——因为工作线程在 `run()` 中并不使用 PUB socket（心跳在 `check_heartbeat` 内调用 `publish`，而业务线程可能同时 `publish`）。客户端 `self._lock`（`vnpy/rpc/client.py:51`）保护 REQ socket 的"send_pyobj→poll→recv_pyobj"三步原子性，防止多线程并发远程调用破坏 REQ 严格交替协议（`vnpy/rpc/client.py:69-78`）。
- **无共享状态竞争**：函数注册表 `_functions` 通常在 `start()` 前由主线程注册完成，运行期只读。
- **取消传播**：无显式取消令牌；停止靠布尔标志 `_active` + socket poll 超时唤醒，属协作式退出。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `vnpy/rpc/server.py`：RpcServer 的 REP/PUB socket 管理、请求分派、心跳发布
- `vnpy/rpc/client.py`：RpcClient 的 REQ/SUB socket 管理、`__getattr__` 远程代理、订阅回调骨架、RemoteException
- `vnpy/rpc/common.py`：心跳常量与 SIGINT 处理
- `vnpy/rpc/__init__.py`：对外导出 `RpcServer` / `RpcClient`

**Out-of-Scope（不在本仓库源码内）**
- **pyzmq（ZeroMQ Python 绑定）**：REP/REQ/PUB/SUB socket 语义、`send_pyobj`/`recv_pyobj` 序列化、`poll`、TCP keepalive 均由 pyzmq 提供（不在本仓库源码内）。
- **网络传输层**：TCP/IPC 连接管理、连接重试、网络分区处理由 ZeroMQ 内核侧承担，本层不做。
- **序列化协议**：`send_pyobj` 基于 pickle 序列化任意 Python 对象（不在本仓库源码内），跨语言调用不支持。
- **上层业务逻辑**：注册哪些函数、推送哪些主题、回调如何处理，由使用方（如 MainEngine、行情/交易网关）决定，本叶子只提供通信骨架。
- **认证 / 鉴权 / TLS**：本层不提供，ZeroMQ CURVE 等安全机制未启用。

## 8. 与相邻子系统交互

- **上游（调用方）→ 本叶子**：量化主引擎（`vnpy/trader/engine.py` MainEngine）或独立进程通过 `RpcServer.register()` 把交易/查询函数注册为远程服务；远端进程通过 `RpcClient` 透明代理调用这些函数。典型场景：跨进程把 MainEngine 的接口暴露给 GUI/策略进程（见 examples/client_server、simple_rpc）。
- **本叶子 → 下游**：
  - 客户端 → pyzmq REQ socket → 服务端 REP socket（请求-应答，同步阻塞在 `self._lock` 内）。
  - 服务端 PUB socket → pyzmq → 客户端 SUB socket（发布-订阅，单向广播）。
  - 服务端 `publish()` 推送的数据通常来自 `vnpy.trader` 的行情/委托/持仓对象（见 trader-core 域）；客户端 `callback()` 覆写后把推送转发给 `vnpy.event` 事件引擎分发。
- **相邻叶子指引**：事件分发与订阅者注册见 trader-core 域 event-engine 叶子；交易网关接口见 trader-core 域 gateway-converter 叶子。

## 9. 语言专项适配口径（纯 Python：能力缝 + 注册表 + 插件体系）

- **能力缝（capability seam）**：本叶子的核心能力缝是"**远程方法动态代理 + 模板方法回调**"。`RpcClient.__getattr__`（`client.py:56`）是典型的 Python 动态代理扩展缝——调用方无需实现任何接口，任何未定义属性都会被解释为远程函数名，返回闭包 `dorpc`；`RpcClient.callback`（`client.py:152`）与 `on_disconnected`（`client.py:164`）是模板方法缝，基类留 `NotImplementedError`/默认 print，由子类覆写。
- **注册表与工厂**：服务端 `_functions: dict[str, Callable]`（`server.py:19`）是显式注册表，`register()`（`server.py:123`）以 `func.__name__` 为键做字符串→函数路由；客户端侧用 `@lru_cache(100)`（`client.py:55`）把"方法名字符串→闭包对象"做了一层进程内缓存工厂，避免每次远程调用都重建闭包。注册时机：`register()` 为显式调用（非 import 副作用），通常在 `start()` 前完成。
- **可选依赖**：pyzmq 是核心依赖（pyproject 必装），不在 extras 内；本叶子无懒加载/降级。
- **并发模型**：纯 Python 多线程（`threading.Thread`）+ 两把 `threading.Lock`，无 asyncio；锁粒度按 socket 通道拆分（服务端 PUB 一把、客户端 REQ 一把），符合"保护共享 socket 的并发发送"这一最小临界区原则。
- **图型侧重**：本叶子调用链清晰（请求-应答 + 发布-订阅两条消息流），故除 architecture 外补 sequence 图表达客户端-服务端单次 RPC 交互；dataflow 图因与 sequence 信息高度重复，按资源节省原则省略（见第 10 节）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| RPC 通信架构图 | [rpc-communication-architecture.html](rpc-communication-architecture.html) | architecture | showcase |
| 客户端-服务端 RPC 时序图 | [rpc-communication-sequence.html](rpc-communication-sequence.html) | sequence | showcase |

- JSON IR 源文件位于 `json/` 目录（`rpc-communication-architecture.json`、`rpc-communication-sequence.json`）。
- 本叶子未生成 dataflow 图：RPC 的"数据流"本质就是请求-应答消息交互，已由 sequence 图完整表达；再画 dataflow 会与 sequence 信息重复，按资源节省原则省略。
- 本叶子未生成 workflow 图：无多角色带泳道的审批/分步流程；未生成 lifecycle 图：RpcServer/RpcClient 虽有 start/stop 状态，但状态迁移极简（仅 active 布尔翻转），无多阶段状态机语义。
