# 微信推送通知（notify-wechat）

> 本文是 `trader-service` 域下的叶子子系统文档。域级总览见 `../trader-service.md`。
> 本文只展开 vnpy 与微信 iLink 机器人协议对接的扫码绑定、消息收发与错误语义，不重复展开 UI 侧 `WechatDialog`/`WechatWorker` 的职责（见 `../trader-ui/main-window/main-window.md`）。
>
> 源码基准：vnpy 4.4.0，commit `fa5206fe`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 凭据数据结构 | `Credentials` dataclass：bot_id/token/base_url，扫码登录成功后获得 | `vnpy/trader/wechat.py:46` |
| 发送文本消息 | `send_text(creds, user_id, text)`：构造 bot 主动消息体，POST `ilink/bot/sendmessage` | `wechat.py:57` |
| 长轮询收消息 | `poll(creds, sync_buf)`：POST `ilink/bot/getupdates`，返回 `(user_ids, next_sync_buf)` | `wechat.py:87` |
| 请求二维码 | `request_qrcode()`：GET `ilink/bot/get_bot_qrcode`，返回 `(qrcode, scan_url)` | `wechat.py:122` |
| 等待扫码确认 | `wait_for_login(qrcode, deadline)`：轮询 `ilink/bot/get_qrcode_status`，处理 confirmed/expired/redirect | `wechat.py:145` |
| 统一 POST/GET 封装 | `_post`/`_get`：拼装 base_info/请求头、超时异常转换、调用 `_parse` 校验业务码 | `wechat.py:193`、`wechat.py:225` |
| 响应校验 | `_parse`：HTTP 状态、JSON 格式、`ret`/`errcode` 业务码；-14 映射 `SessionExpired` | `wechat.py:254` |
| 请求头生成 | `_headers`：每请求随机 `X-WECHAT-UIN`（base64 随机 uint32）、Bearer token | `wechat.py:285` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `WeixinError` | `wechat.py:34` | SDK 通用异常基类 |
| `SessionExpired` | `wechat.py:38` | iLink 会话过期（ret/errcode == -14） |
| `WeixinTimeout` | `wechat.py:42` | HTTP 请求超时 |
| `Credentials` | `wechat.py:46` | 扫码成功后持有的访问凭据（bot_id + token + base_url） |
| `send_text` | `wechat.py:57` | 主动给指定 user_id 发文本（message_type=2 bot 主动消息） |
| `poll` | `wechat.py:87` | 长轮询入站消息，提取发送方 user_id（忽略 bot 自身消息） |
| `request_qrcode` / `wait_for_login` | `wechat.py:122` / `wechat.py:145` | 两步扫码绑定：取码 → 轮询状态直到确认/过期 |
| `_parse` | `wechat.py:254` | 业务码统一校验中心，错误码→异常映射 |

## 3. 关键调用链

**调用链一：扫码绑定（`request_qrcode` → `wait_for_login`）**
1. `wechat.py:128`–`135`：`request_qrcode()` GET `get_bot_qrcode?bot_type=3`，取 `qrcode` 与 `qrcode_img_content`（扫码链接）。
2. `wechat.py:157`–`188`：`wait_for_login` 在 deadline 内每秒轮询 `get_qrcode_status`：
   - `status == "confirmed"`（`wechat.py:170`）：取 `ilink_bot_id`/`bot_token`/`baseurl` 构造 `Credentials` 返回。
   - `status == "scaned_but_redirect"`（`wechat.py:180`）：切到 `redirect_host` 作为后续 base_url。
   - `status == "expired"`（`wechat.py:185`）：返回 None；超时返回 None。
   - 捕获 `WeixinTimeout`（`wechat.py:166`）则 continue 继续轮询。

**调用链二：发送文本（`send_text`）**
1. `wechat.py:65`–`74`：构造消息体（`message_type=2` bot 主动消息、`item_list` 含 type=1 文本项、随机 `client_id`）。
2. `wechat.py:76`–`84`：开 `requests.Session`，`_post("ilink/bot/sendmessage", token, {msg})`，超时 30s。

**调用链三：响应业务码校验（`_parse`）**
1. `wechat.py:256`：`response.ok` 为假抛 `WeixinError`。
2. `wechat.py:260`：非 JSON 抛 `WeixinError`。
3. `wechat.py:271`：`ret==-14 or errcode==-14` 抛 `SessionExpired`。
4. `wechat.py:273`：其他非零 ret/errcode 抛 `WeixinError` 并带 errmsg。

## 4. 配置项

| 配置项 / 常量 | 默认 / 行为 | 位置 |
|---|---|---|
| `DEFAULT_BASE_URL` | `https://ilinkai.weixin.qq.com` | `wechat.py:19` |
| `_CHANNEL_VERSION` | `"2.2.0"`，每个请求 body 的 base_info.channel_version | `wechat.py:22` |
| `_APP_ID` | `"bot"`，iLink-App-Id 头 | `wechat.py:25` |
| `_CLIENT_VERSION` | `131584`（2.2.0 编码为 24 位整数） | `wechat.py:28` |
| `_REQUEST_TIMEOUT` | 30.0 秒（send/qrcode/status）；poll 用 60s 长轮询 | `wechat.py:31`、`wechat.py:103` |

## 5. 错误与重试语义

- **超时**：`requests.exceptions.Timeout` 统一转 `WeixinTimeout`（`wechat.py:217`、`wechat.py:246`）；`wait_for_login` 捕获后 `continue` 继续轮询（`wechat.py:166`），`poll` 在 `WechatWorker` 中同样 continue。
- **会话过期**：iLink 返回 -14 → `SessionExpired`（`wechat.py:271`），需重新扫码绑定。
- **其他 HTTP 错误**：`requests.RequestException` → `WeixinError`（`wechat.py:219`）；非 2xx / 非 JSON / 业务码非零均抛 `WeixinError`。
- 本模块**不做自动重发**：发送失败直接抛异常，由调用方（WechatEngine/WechatWorker）决定重试。
- 二维码响应缺 `qrcode`（`wechat.py:138`）、扫码确认缺凭据（`wechat.py:176`）均抛 `WeixinError`。

## 6. 并发细节

- 每次 `send_text`/`poll`/`request_qrcode` 内部 `with requests.Session()` 新建并关闭会话（`wechat.py:76`、`wechat.py:96`），无长连接复用。
- `wait_for_login` 是**同步阻塞轮询循环**（`time.sleep(1)`，`wechat.py:188`），由 UI 侧 `WechatWorker(QThread)` 放到后台线程执行（见 main-window 叶子），避免阻塞 Qt 主线程。
- `poll` 为 60s 长轮询（`wechat.py:103`），同样在后台线程循环调用。
- 无锁；`Credentials` 不可变数据类，跨线程只读共享。
- 取消传播：UI 侧通过 `WechatWorker._stop` 标志在下一个循环点退出（不在本模块）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `vnpy/trader/wechat.py`：iLink 协议的请求/响应封装、扫码绑定流程、异常体系。

**Out-of-Scope（不在本仓库源码内）**
- 微信 iLink 服务器（`ilinkai.weixin.qq.com` 及 redirect_host）——不在本仓库源码内。
- `requests` 库（HTTP 客户端）——不在本仓库源码内。
- UI 绑定对话框 `WechatDialog` 与后台线程 `WechatWorker`——在 `vnpy/trader/ui/widget.py`，归 main-window 叶子。
- `WechatEngine`（实际策略推送触发点）——在 `vnpy/trader/engine.py`，归 trader-core 域。

## 8. 与相邻子系统交互

- **上游调用方**：`WechatEngine`（engine.py）在策略事件触发时调 `send_text`；`WechatWorker`（widget.py）在绑定时调 `request_qrcode`/`wait_for_login`/`poll`。
- **下游依赖**：`wechat.py` 仅依赖 `requests` 与标准库（base64/json/secrets/uuid/time），不依赖 vnpy 其他模块。
- **交互方向**：vnpy ↔（HTTPS）↔ iLink 机器人网关；凭据 `Credentials` 在 UI 绑定流程产出后持久化供 `WechatEngine` 复用。

## 9. 语言专项适配口径（纯 Python：能力缝 + 注册表 + 插件体系）

- **能力缝（外部 SDK 适配层）**：本叶子是典型的"外部 HTTP 协议适配层"——把微信 iLink 的私有协议（base_info/channel_version、随机 X-WECHAT-UIN、-14 会话码）封装成 `send_text`/`poll`/`request_qrcode` 等简单 Python 函数，框架其余部分不感知 iLink 细节。
- **异常体系分层**：自定义 `WeixinError` 基类派生 `SessionExpired`/`WeixinTimeout`（`wechat.py:34`–`43`），把第三方库（`requests`）的异常翻译成业务语义异常，供上层按类型分支处理。
- **数据传输对象**：`Credentials` 用 `@dataclass`（`wechat.py:46`）承载跨函数/跨线程凭据，不可变、易序列化。
- **可选依赖**：`requests` 为核心依赖（pyproject 已声明），无懒加载；本模块无注册表/工厂，是无状态函数集 + 凭据对象。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| notify-wechat 组件架构图 | `notify-wechat-architecture.html` | architecture | showcase |
| 扫码绑定时序图 | `notify-wechat-sequence.html` | sequence | showcase |

JSON IR 位于 `json/` 目录。
本叶子未生成 dataflow / lifecycle / workflow 图：扫码绑定是清晰的请求-响应交互链（已由 sequence 表达），无独立数据变换管道；二维码状态（待扫/已扫确认/过期）虽有状态色彩，但它是远端 iLink 服务端状态、由 `wait_for_login` 轮询驱动，本质仍属时序交互，不另画 lifecycle。
