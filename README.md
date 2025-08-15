
---

markdown
# TTapi/README.md

# Tank Trouble 客户端模块（TTapi.py）调用手册

本文档面向接入方/插件作者，介绍 `TTapi.py` 的安装、核心类与方法、事件回调、返回码语义与示例代码，帮助你快速完成登录、拉取房间列表、加入/退出房间、订阅实时状态等功能。

---

## 目录

- [安装与依赖](#安装与依赖)
- [快速上手示例](#快速上手示例)
- [配置项与常量](#配置项与常量)
- [核心概念](#核心概念)
- [User 类：属性与方法](#user-类属性与方法)
  - [属性](#属性)
  - [事件回调 API](#事件回调-api)
  - [自动刷新房间列表](#自动刷新房间列表)
  - [账号相关](#账号相关)
  - [房间相关](#房间相关)
  - [WebSocket 连接](#websocket-连接)
- [返回码对照表](#返回码对照表)
- [数据结构](#数据结构)
- [最佳实践与注意事项](#最佳实践与注意事项)
- [Admin 类](#admin-类)
- [附：增量同步协议](#附增量同步协议)

---

## 安装与依赖

- Python 3.9+
- 依赖：
  - `requests`
  - `python-socketio`

安装示例：
```bash
pip install requests "python-socketio[client]"
````

---

## 快速上手示例

```python
from TTapi import User

# 1) 创建用户实例
u = User(username="Alice_01", password="Abc12345")

# 2) 注册（首次）或直接登录
ret = u.register()   # 可能返回 "username_exist" 等
print("register:", ret)

ret = u.login()
print("login:", ret)

# 3) 绑定房间列表自动刷新回调（已登录且不在房间时，每2秒拉取）
def on_rooms(rooms, user):
    print("[rooms]", rooms)

u.bind_roomlist_update(on_rooms)
u.start_auto_refresh(interval=2.0)

# 4) 创建房间或加入已有房间
ret = u.create_room(playermax=4)
print("create_room:", ret)

# 或：加入房间
# u.join_room(room_id=1)

# 5) 绑定 socket 事件
def on_join(payload, user):
    print("[join_room]", payload["status"], "room_id=", payload["data"]["room_id"])
user_cb = lambda p, usr: print("[room_update]", p.get("type"), "players=", len(p["data"]["players"]))
u.on("join_room", on_join)
u.on("room_update", user_cb)

# 6) 退出房间
# u.leave_room()
```

---

## 配置项与常量

```python
login_url = "http://localhost:25000/"
game_url  = "http://localhost:25001/"
room_url  = "http://localhost:25002/"
ADMIN_TOKEN = "admin_token"

# 用户名/密码校验（本地预检）
USERNAME_MIN_LEN = 3
USERNAME_MAX_LEN = 16
PASSWORD_MIN_LEN = 8
PASSWORD_MAX_LEN = 16
```

> 生产环境请将 `login_url / game_url / room_url / ADMIN_TOKEN` 替换为你自己的地址与凭据。

---

## 核心概念

* **User**：客户端会话对象，封装登录态、HTTP 调用、SocketIO 事件处理等。
* **轻量事件系统**：通过 `on/once/off` 绑定回调，可订阅 `connect/disconnect/join_room/room_update/game_update` 等事件。
* **自动刷新房间列表**：当“已登录且不在任意房间”时，后台线程每隔 N 秒调用一次 `get_all_room_info(brief=1)`，并触发绑定回调。
* **增量同步**：服务端每 5 秒全量推一次 `game_updata`，其余 tick 推差量。客户端用 `dict_patch` 应用补丁拿到最新状态。

---

## User 类：属性与方法

### 属性

```python
user.username: str
user.password: str
user.is_ban: bool                     # 预留
user.token: Optional[str]             # 登录成功后获得
user.uid: Optional[int]               # 登录成功后获得
user.status: int                      # 0=未登录, 1=已登录
user.room_data: Optional[dict]        # 最近一次房间全量/合成后的状态
user.room: dict                       # 解析后的房间简易视图（等于 room_data）
user.room_players: list               # 解析后的玩家列表
user.rooms: list                      # 最近一次 get_all_room_info 的结果（brief 模式）
user.sio: socketio.Client             # SocketIO 客户端实例
user.is_online: bool                  # SocketIO 连接状态
user.room_data_temp: dict             # 增量同步的“当前基线”
```

### 事件回调 API

```python
user.on(event, func)     # 绑定回调；func(payload, user)
user.once(event, func)   # 绑定一次性回调；触发后自动解绑
user.off(event, func?)   # 解绑指定回调；省略 func 则解绑该事件的全部回调
```

建议的事件名：

* `connect` / `disconnect`
* `auth_success`（由服务端发送到客户端时会以普通消息形式出现，通常在 `connect` 后即可视为鉴权通过）
* `join_room` / `leave_room`
* `room_update`  （房内成员变化）
* `game_update`  （游戏状态差量/全量——在本模块中对应 `on_game_updata` 触发后转发为 `game_update`）

> 注：底层已对服务端的 `room_update`/`room_updata` 名称差异做了兼容。

### 自动刷新房间列表

```python
user.bind_roomlist_update(func)  # 设置 rooms 刷新回调；func(rooms, user)
user.start_auto_refresh(interval=2.0, callback=None)  # 启动后台刷新线程
user.stop_auto_refresh()         # 停止后台刷新
```

工作逻辑：

* 仅当 `status == 1`（已登录）且 `user.room` 为空（不在房间）时，才会调用 `get_all_room_info()`。
* 默认间隔 `2.0` 秒，可自定义。
* 若同时设置了 `bind_roomlist_update` 与 `start_auto_refresh(callback=...)`，以 `callback` 为准。

### 账号相关

#### `register() -> str`

* 本地校验：

  * 用户名：`3~16` 位，仅允许字母/数字/下划线
  * 密码：`8~16` 位，必须同时包含字母与数字（仅允许字母数字）
* 可能返回：

  * `"regis_success" | "username_exist" | "parameter_error" | "network_error" | "unknown_error" | "username_invalid" | "password_invalid"`

#### `login() -> str`

* 本地校验同上。
* 成功则自动尝试 `SocketIO` 连接，并 **启动自动刷新房间列表**（若不在房间）。
* 可能返回：

  * `"login_success" | "parameter_error" | "password_error" | "network_error" | "unknown_error"`

### 房间相关

#### `get_all_room_info() -> Union[list, str]`

* 需要已登录且 token 存在。
* 请求服务端 `brief=1` 精简视图，结果写入 `user.rooms` 并返回。
* 错误时返回：`"not_login" | "token_error" | "network_error" | "unknown_error"`

返回列表示例：

```python
[
  { "room_id": 1, "playermax": 6, "player_num": 2, "create_player_name": "Alice" },
  ...
]
```

#### `create_room(playermax=6) -> str`

* 成功后会自动通过 WebSocket 发送 `join_room` 进入该房。
* 可能返回：

  * `"create_success" | "not_login" | "token_error" | "room_full" | "player_in_room" | "network_error" | "unknown_error"`

#### `join_room(room_id: int) -> Optional[str]`

* 通过 WebSocket 发送加入请求。
* 成功后将触发 `on_join_room`，并更新 `room_data / room / room_players`。
* 本方法本身不返回服务器码，建议结合事件回调监听。

#### `leave_room(room_id: Optional[int]=None)`

* 通过 WebSocket 发送退出请求。
* `room_id` 为空时将读取当前 `user.room['room_id']`。
* 成功将触发 `on_leave_room` 与自动刷新列表恢复。

#### `get_room_info(room_id: Optional[int]) -> Union[dict, str]`

* 一般不必主动调用，因为进入房间后服务端会主动推送全量/差量。
* 返回 `"room_not_exist"`/`"token_error"`/`"network_error"`/`"unknown_error"` 或 `dict`（全量房间）。

### WebSocket 连接

#### `connect()`

* 手动连接：`user.sio.connect(game_url, auth={'token': user.token})`
* 一般无需手动调用，`login()` 成功后会自动连接。

---

## 返回码对照表

| 场景                    | 返回值（字符串）           | 说明                                       |
| --------------------- | ------------------ | ---------------------------------------- |
| `register()`          | `regis_success`    | 注册成功（同时获得 `token`/`uid`，但 `status` 仍为 0） |
|                       | `username_exist`   | 用户名已存在                                   |
|                       | `parameter_error`  | 请求参数不合法/缺失                               |
|                       | `network_error`    | 网络/超时等异常                                 |
|                       | `unknown_error`    | 未归类异常                                    |
|                       | `username_invalid` | 本地校验：用户名不符合格式                            |
|                       | `password_invalid` | 本地校验：密码不符合格式                             |
| `login()`             | `login_success`    | 登录成功并连接 SocketIO；`status=1`              |
|                       | `parameter_error`  | 本地校验失败                                   |
|                       | `password_error`   | 用户名或密码错                                  |
|                       | `network_error`    | 网络/超时等异常                                 |
|                       | `unknown_error`    | 未归类异常                                    |
| `get_all_room_info()` | `not_login`        | 未登录或缺少 token                             |
|                       | `token_error`      | token 无效                                 |
|                       | `network_error`    | 网络错误                                     |
|                       | `unknown_error`    | 未归类异常                                    |
| `create_room()`       | `create_success`   | 创建成功（随后会自动尝试 join）                       |
|                       | `not_login`        | 未登录/无 token                              |
|                       | `token_error`      | token 无效                                 |
|                       | `room_full`        | 达到房间上限                                   |
|                       | `player_in_room`   | 已在其他房间                                   |
|                       | `network_error`    | 网络错误                                     |
|                       | `unknown_error`    | 未归类异常                                    |

---

## 数据结构

### `room_info(room_data) -> (room, players)`

* 简单拆分工具；`room` 即 `room_data`，`players` 为 `room_data['players']`（默认空列表）。

### `user.room_data`（房间全量/合成后）

* 通过 `on_join_room` / `on_room_updata` / `on_game_updata` 更新。
* `on_game_updata` 内部使用 `dict_patch` 将差量补丁应用到 `room_data_temp` 基线，并回写到 `room_data`。

---

## 最佳实践与注意事项

1. **总是通过 `login()` 获取 token 并建立 SocketIO 连接。**
2. 列表页请优先使用 `get_all_room_info()` 的 **精简视图**（内部已默认携带 `brief=1`，可极大减少流量）。
3. 进入房间后，**不要频繁主动拉取**；依赖服务端的全量/差量推送即可。
4. UI 层使用 `on('room_update', ...)` 与 `on('game_update', ...)` 即时刷新。
5. 建议在退出房间后调用 `start_auto_refresh()` 恢复房间列表自动刷新；模块已在合适时机自动处理。
6. 用户名/密码有本地严格校验，不符合规范不会发起网络请求。
7. 若你需要统一日志，可替换/包装 `log_info()`。

---

## Admin 类

当前为占位（仅保存 `token`），后续可扩展管理员接口调用封装：

```python
from TTapi import Admin
admin = Admin(token="your_admin_token")
```

---

## 附：增量同步协议

* 服务端：每 5 秒推送一次 **全量** 房间信息，其余 tick 推送 **差量**：

  * 差量由 `dict_diff(old, new)` 生成
* 客户端：在 `on_game_updata` 中：

  * 使用 `dict_patch(old, diff)` 将差量应用到 `room_data_temp`
  * 回写 `room_data` 与 `room_data_temp`，并据此更新 UI

如需自定义你的状态合并逻辑，可直接复用本模块中提供的 `dict_diff / dict_patch` 工具函数。


