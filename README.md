# Tank Trouble 服务端 & 客户端（TTapi）说明

> 一进程多线程运行：
>
> * 登录/注册 HTTP：`0.0.0.0:25000`
> * 房间 HTTP：`0.0.0.0:25002`
> * WebSocket（游戏推送）：`0.0.0.0:25001`（Flask-SocketIO）

* 数据持久化：**MySQL**（`users` 表，含 `is_ban`）
* Token：内存字典 + 过期堆 + 空闲回收（后台清理线程，30s 一轮）
* 管理后台：`/admin/*`，带 CSRF、登录、用户管理、Token 管理、房间监控、导出等
* 模板：`templates/admin/`（已提供美化版；不改后端接口）

---

## 目录

* [环境与依赖](#环境与依赖)
* [数据库初始化](#数据库初始化)
* [配置](#配置)
* [启动](#启动)
* [管理后台](#管理后台)
* [HTTP API](#http-api)
* [WebSocket 协议](#websocket-协议)
* [TTapi 客户端用法](#ttapi-客户端用法)
* [返回码对照表](#返回码对照表)
* [故障排查](#故障排查)
* [安全与上线建议](#安全与上线建议)

---

## 环境与依赖

* Python ≥ 3.9（开发环境可用 3.13）
* MySQL 5.7/8.x

安装依赖（按你实际 `requirements.txt` 为准，示例）：

```bash
pip install flask flask-socketio eventlet requests pymysql bcrypt
```

> 如用 gevent，请相应调整 `async_mode`；默认使用 `eventlet/threading`。

---

## 数据库初始化

服务端启动时会执行 `ensure_schema()` 兜底建表（至少包含 `users` 表）。
`users` 表字段（示例）：

| 字段             | 类型              | 说明        |
| -------------- | --------------- | --------- |
| uid (PK, AI)   | BIGINT UNSIGNED | 用户ID      |
| username (UK)  | VARCHAR(64)     | 用户名       |
| password\_hash | VARCHAR(100)    | Bcrypt 哈希 |
| ip             | VARCHAR(45)     | 注册IP      |
| is\_ban        | TINYINT(1)      | 是否封禁(0/1) |
| created\_at    | TIMESTAMP       | 创建时间      |
| updated\_at    | TIMESTAMP       | 更新时间      |

> 管理后台若启用账号与审计，建议一并建：`admin_users`、`admin_audit_logs`（DDL 可参考注释或你的实现）。

---

## 配置

根据你的实现，配置可通过环境变量或常量传入（以下为常见项，按你代码为准）：

| 变量名                      | 默认值           | 说明                             |
| ------------------------ | ------------- | ------------------------------ |
| `MYSQL_HOST`             | 127.0.0.1     | MySQL 主机                       |
| `MYSQL_PORT`             | 3306          | MySQL 端口                       |
| `MYSQL_DB`               | tank\_trouble | 数据库名                           |
| `MYSQL_USER`             | tt\_user      | 用户名                            |
| `MYSQL_PASS`             | tt\_password  | 密码                             |
| `ADMIN_TOKEN`            | …             | 管理 API 令牌（/admin\_get\_data 等） |
| `MAX_TOKENS`             | 20000         | Token 池上限                      |
| `TOKEN_TTL_SECONDS`      | 86400         | Token 有效期（秒）                   |
| `TOKEN_IDLE_TTL_SECONDS` | 3600          | Token 空闲过期（秒）                  |

---

## 启动

服务端（在一个进程里多线程托起三个服务）：

```bash
python server.py
```

> 主线程跑 SocketIO(`:25001`)，子线程各跑一个 Flask（`:25000`, `:25002`）。
> **注意**：务必关闭 Flask 自带 reloader（代码已关闭），避免多进程导致线程重复。

---

## 管理后台

* 登录页：`GET /admin/login` → 表单 `POST /admin/login`（带 `_csrf`）
* 仪表盘：`GET /admin/`（用户数/在线 token/房间数）
* 用户管理：`GET /admin/users`（搜索/分页）

  * 重置密码：`POST /admin/users/<uid>/resetpw`（`_csrf`,`password`）
  * 封禁/解禁：`POST /admin/users/<uid>/ban`（`_csrf`,`is_ban`=0/1）
* Token 管理：`GET /admin/tokens`（可按 UID 过滤）

  * 撤销某 token：`POST /admin/tokens/revoke`（`_csrf`,`token`）
  * 撤销某 UID 全部 token：`POST /admin/tokens/revoke`（`_csrf`,`uid`）
* 房间监控：`GET /admin/rooms`（空房可一键解散，`POST /admin/rooms/<room_id>/dismiss`）
* 导出：`GET /admin/export`

> 模板在 `templates/admin/`，已包含 `_base.html`、`_macros.html` 与四个页面。**后端无需改动**。

---

## HTTP API

### 1）登录/注册服务（:25000）

#### `POST /login`

请求 JSON：

```json
{"username": "str", "password": "str"}
```

成功 (200)：

```json
{"status":102,"message":"Login successful! 登录成功","token":"<token>","uid":123}
```

失败 (200)：

* `100` 参数缺失
* `101` 用户名或密码错误
* `103` 用户被封禁（新增）
* `500` 服务器错误

#### `POST /register`

请求 JSON：

```json
{"username":"str","password":"str"}
```

返回 (200)：

* `202` 注册成功（返回 `token`,`uid`）
* `201` 用户名已存在
* `203` IP 注册过于频繁（限频 60s）
* `204` 同一 IP 账号数超上限
* `200` 参数缺失 / 无效请求

#### 管理兼容接口（面向旧工具/脚本）

* `GET /admin_get_data?token=ADMIN_TOKEN[&include_tokens=1][&limit=1000][&offset=0]`
  返回：

  ```json
  {
    "status":300,"message":"Admin page! 管理员页面",
    "USER_DATABASE":[{"uid":1,"username":"u","password":"bcrypt-hash","ip":"...","is_ban":0}, ...],
    "UIDMAX": 12345,
    "users_count": 999,
    "tokens_count": 88,
    "TOKEN_DATABASE": { ... } // 当 include_tokens=1
  }
  ```
* `POST /admin_send_data?token=ADMIN_TOKEN`
  兼容性写入（慎用；DB 模式下一般仅用于导入工具）。
* `GET /get_token?token=ADMIN_TOKEN`
  `1000` 有效 / `1001` 无效，并可返回 `TOKEN_DATABASE`。

---

### 2）房间服务（:25002）

所有接口都需要 `?token=<登录返回的token>`。

#### `POST /create_room?token=...`

```json
{"playermax": 4}
```

返回：

* `1000` 创建成功（返回房间 `data`）
* `1001` token 无效
* `1002` 参数缺失 / 房间满 / 已在其他房间

#### `GET /get_room_info?token=...&room_id=1`

* `1000` 获取成功（`data` 为房间详情）
* `1001` token 无效
* `1002` 参数缺失 / 房间不存在

#### `GET /get_all_room_info?token=...[&brief=1]`

* `brief=1` 返回精简列表
* 返回 `1000`；错误同上

> 兼容接口：`POST /game_ready`（已标注废弃，仍保留）

---

## WebSocket 协议（:25001）

**连接**：
客户端需以 `auth={'token': '<登录 token>'}` 连接 Socket.IO。

事件：

* 服务端 → 客户端

  * `auth_success`：连接鉴权通过 → `{"status":1000,"data":{"uid":123}}`
  * `game_updata`：**游戏状态推送**（20Hz；每 5 秒一次全量，其余为差量 `dict_diff`）
  * `room_update`：房间成员变化广播（`type: 'join' | 'leave'`）

* 客户端 → 服务端

  * `join_room`：`{"token":"...","room_id":1}`

    * 成功 `{"status":1000,"data":<room_info>}`
  * `leave_room`：`{"token":"...","room_id":1}`

    * 成功 `{"status":1000}`
  * 错误：`error` 事件，`status` 与 HTTP 同语义

---

## TTapi 客户端用法

`TTapi.py` 提供了简化调用。常见属性/方法（以你的实现为准）：

* 地址：

  ```python
  base = "http://127.0.0.1"
  login_url = f"{base}:25000/"
  room_url  = f"{base}:25002/"
  game_url  = f"{base}:25001/"
  ```

* 典型流程：

  ```python
  from TTapi import TTapi

  api = TTapi(username="user1", password="pass1")
  ret = api.login()
  if ret == "login_success":
      # token/uid 已在 api.token/api.uid
      api.sio.connect(game_url, auth={'token': api.token})  # 你代码里已有
      print(api.get_all_room_info())  # 拉取房间
  elif ret == "user_banned":
      print("该账号已被封禁，无法登录")
  else:
      print("登录失败:", ret)
  ```

* `login()` 返回值（字符串）：

  * `login_success` / `parameter_error` / `password_error`
  * `user_banned`（**新增**：服务端 `status==103`）
  * `network_error` / `unknown_error`

---

## 返回码对照表

| 模块   | 码    | 说明                  |
| ---- | ---- | ------------------- |
| 登录   | 100  | 参数缺失/无效请求           |
| 登录   | 101  | 用户名或密码错误            |
| 登录   | 102  | 登录成功（返回 token, uid） |
| 登录   | 103  | **用户被封禁（新增）**       |
| 登录   | 500  | 服务器错误               |
| 注册   | 200  | 参数缺失/无效请求           |
| 注册   | 201  | 用户名已存在              |
| 注册   | 202  | 注册成功（返回 token, uid） |
| 注册   | 203  | IP 频率限制             |
| 注册   | 204  | 同 IP 账号数上限          |
| 房间   | 1000 | 成功                  |
| 房间   | 1001 | token 无效            |
| 房间   | 1002 | 参数缺失 / 满员 / 不存在等    |
| 管理兼容 | 300  | 成功                  |
| 管理兼容 | 301  | `ADMIN_TOKEN` 无效    |
| 管理兼容 | 302  | 无效请求                |
| 管理兼容 | 1000 | get\_token：有效       |
| 管理兼容 | 1001 | get\_token：无效       |

---

## 故障排查

* **`jinja2.exceptions.UndefinedError: 'csrf_field' is undefined`**
  使用美化模板时，需添加 `templates/admin/_macros.html` 并在每个页面顶部：
  `{% import "admin/_macros.html" as ui %}`，然后用 `{{ ui.csrf_field(csrf) }}`。
  如仍报错，检查 `render_template()` 是否传入了 `csrf`。

* **WebSocket 连接不上**
  确认：`login()` 已成功返回 `token`；WS 连接时 `auth={'token': token}`；端口 `25001` 未被防火墙拦截。

* **房间偶发 KeyError/并发异常**
  建议对 `GAMEROOMS['room_list']` 的读写加全局锁（已在示例补丁中说明）。

---

## 安全与上线建议

* 管理后台务必置于内网或开启鉴权（你已有登录与 CSRF；如有需要可加 CSP/TLS）。
* `ADMIN_TOKEN` 仅用于兼容旧接口，不要暴露公网。
* 部署建议：前置 Nginx/Caddy（TLS/限速/超时），后端运行单进程；Load balancer 横向扩展需改 Token 与房间状态为共享存储/消息总线。
* 登录接口已加入**封禁校验**（`status=103`）。如有爆破风险，可开启轻微随机延时（50–150ms）。

---

如需我把 README 拆成中/英双语版本，或生成一份 `curl`/Postman 集合，一条命令就能本地联调的脚本，我可以直接补上。
