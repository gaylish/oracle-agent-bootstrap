# Oracle Agent Protocol v1

> Runner(Agent) ↔ Oracle(Controller) 控制通道协议。
> 设计原则：**Runner 永远不开放公网入站**，全部通信由 Runner 主动发起（outbound HTTPS）。

---

## 1. 概览

- **方向**：永远 `Runner ──► Oracle`（outbound）。Oracle 不向 Runner 建立连接。
- **传输**：HTTPS（生产，TLS 强制）/ HTTP（本地联调），JSON 请求/响应。
- **版本**：所有路径前缀 `/api/v1/`，向后不兼容变更升 v2。
- **认证**：每 Agent 一个 `agent_id + token`，Bearer 头传递，Server 只存 token 哈希。
- **关键接口**：`pull` 采用**长轮询**（Long-poll），实现"Oracle 决定任务、Runner 主动拉取"。

## 2. 核心概念

| 概念 | 说明 |
|---|---|
| Agent | 一个已注册的 Runner 客户端（进程 + 唯一 agent_id） |
| Task | 一次可执行的工作单元；由 Oracle 创建，Agent pull 后执行并回报 |
| Capability | Agent 声明的能力列表（`ssh`、`agent`、`docker`、`cloudflare-tunnel`…），Oracle 据此决策下发 | 
| Heartbeat | Agent 周期性上报在线状态，Oracle 据此维护在线表 |
| Long-poll | pull 请求在 Server 端挂起等待（默认 30s，最大 60s），有任务立即返回 |

## 3. 认证

- 首次注册：`POST /register`，Server 返回 `agent_id + token`（**token 只在注册响应中出现一次**，Agent 必须持久化到本地 `state.json`，权限 600）。
- 后续所有请求：`Authorization: Bearer <token>`。
- Server 只存 `token_hash`（sha256 hex），校验用 `hmac.compare_digest`。
- 缺失/无效 token → `401`；token 与 agent_id 不匹配 → `403`。

## 4. 接口定义

Base URL: `https://oracle.example.com/api/v1/agent`

### 4.1 `POST /register`

注册或重注册（幂等）。

请求：
```json
{
  "agent_id": "runner-01",
  "token": "aBcDeF...",
  "name": "runner-01",
  "version": "0.1.0",
  "capabilities": ["ssh", "agent"],
  "meta": {"hostname": "runner-01", "os": "ubuntu-22.04"}
}
```
> `agent_id` 首次注册时由 Agent 自行生成（实现：`runner-<hostname>`，hostname 非法字符替换为 `-`）。
> `token` 为**重注册确认字段**：`agent_id` 已存在时必须携带当前 token 才能重新注册（防止身份被冒领）；全新注册时省略。

响应 `200`：
```json
{
  "agent_id": "runner-01",
  "token": "aBcDeF...",
  "heartbeat_interval_sec": 60,
  "server_time": "2026-09-23T12:00:00Z"
}
```

规则：
- `agent_id` 不存在 → 创建新 Agent，生成新 token（仅在本次响应返回一次）。
- `agent_id` 已存在 + token 正确 → 重注册（幂等），响应 `token` 为 `null`（Agent 沿用本地 token），更新 capabilities/meta。
- `agent_id` 已存在但 token 缺失/错误 → `403`。
- Agent 丢失本地 token（state.json 被删）时，带原 `agent_id` 重注册会得到 403；应换新 `agent_id` 注册，或由 Oracle 侧重置（见附录 A）。

### 4.2 `POST /heartbeat`

请求：
```json
{
  "agent_id": "runner-01",
  "status": "online",
  "version": "0.1.0",
  "capabilities": ["ssh", "agent"],
  "load": {"cpu": 0.1, "mem": 0.4}
}
```
`status`: `starting | online | degraded`

响应 `200`：
```json
{"ok": true, "server_time": "2026-09-23T12:00:05Z", "heartbeat_interval_sec": 60}
```
- Server 更新 `last_seen`、`status`、`capabilities`。`last_seen > 3×interval` 视为 offline。

### 4.3 `POST /pull`（长轮询）

请求：
```json
{"agent_id": "runner-01", "timeout_sec": 30}
```

响应 A（有任务，任务被 Server 标记为 `assigned`）：
```json
{
  "task": {
    "task_id": "01J2K3L4M5N6P7Q8R9S0T1U2V3",
    "type": "exec",
    "params": {"command": ["echo", "hello"], "timeout_sec": 30},
    "timeout_sec": 60,
    "created_at": "2026-09-23T12:00:10Z"
  }
}
```

响应 B（无任务，最多挂起 `timeout_sec` 后返回）：
```json
{"task": null}
```

规则：
- `timeout_sec` 默认 30，最大 60（服务端钳制）。
- **领取语义**：任务在 pull 时 `pending → assigned`（租约）。Agent 必须在任务 `timeout_sec` 内回报 `result`。
- 超时未回报：Server 可将其重置为 `pending`（重新投递，attempts+1）。v1 实现为"超过 15 分钟自动重置"，重投递上限后续版本再定。
- 无入站：Oracle 想立即触发任务时，只能创建任务等 Agent 下一次 pull 拉走（长轮询下延迟 ≤ 30s）。

### 4.4 `POST /result`

请求：
```json
{
  "agent_id": "runner-01",
  "task_id": "01J2K3L4M5N6P7Q8R9S0T1U2V3",
  "status": "success",
  "output": {"stdout": "hello\n", "stderr": "", "exit_code": 0},
  "error": null,
  "started_at": "2026-09-23T12:00:11Z",
  "finished_at": "2026-09-23T12:00:12Z"
}
```
`status`: `success | failed | timeout | rejected`

响应：
```json
{"ok": true}
```
- `output` 与 `error` 均为任意 JSON（对 Server 不透明，仅存档；Admin 查看）。
- 幂等：对已终结任务重复提交相同 status → `ok`，不报冲突。

### 4.5 `POST /report`

事件上报（能力就绪、日志、错误等），与任务生命周期解耦。

请求：
```json
{
  "agent_id": "runner-01",
  "event": "tunnel_ready",
  "data": {"hostname": "ssh.runner01.example.com"},
  "ts": "2026-09-23T12:00:20Z"
}
```
v1 事件枚举（宽松校验，未知事件也记录）：
`log | error | capability_installed | tunnel_ready | shutdown_requested`

响应：
```json
{"ok": true}
```

## 5. 任务模型

### 5.1 Schema

```json
{
  "task_id": "string (ULID/uuid)",
  "agent_id": "string",
  "type": "test | exec | ...",
  "params": {},
  "status": "pending | assigned | running | success | failed | timeout | rejected",
  "timeout_sec": 60,
  "attempts": 1,
  "created_at": "RFC3339",
  "started_at": null,
  "finished_at": null,
  "result": null
}
```

### 5.2 状态机

```
pending ──(pull 领取)──► assigned ──(Agent 开始执行)──► running
                            │                              │
                            │ 租约过期(15min)              │
                            ▼                              ▼
                         pending                    success | failed | timeout | rejected
                            ▲
                            └────── attempts+1 重新投递
```

- `running` 状态由 Agent 在 result 中隐含（v1 无独立"开始执行"接口；如需要可扩展 `POST /start`）。

## 6. 任务类型（Task Types）

### 6.1 v1 已实现

| type | params | result |
|---|---|---|
| `test` | 任意 JSON（连通性验证） | `{"echo": <params>, "message": "hello from runner"}` |
| `exec` | `{"command": ["prog", "arg1", ...], "timeout_sec": 300}` | `{"stdout": "...", "stderr": "...", "exit_code": 0}` |
| `shutdown` | `{"delay_sec": 60}`（默认 60：先回报再断电） | `{"status": "shutdown-issued", "delay_sec": 60}` |

- `exec` 超时（超出 `timeout_sec`）→ result status `timeout`。
- `exec` 非零退出码 → status `failed`，但 `output` 仍携带 stdout/stderr。

### 6.2 已预留（定义草案，后续版本实现）

| type | params 草案 | 说明 |
|---|---|---|
| `install_cloudflare` | `{"version": "latest", "tunnel_name": "..."}` | 安装 cloudflared |
| `configure_tunnel` | `{"tunnel_name": "...", "hostnames": [{"hostname": "ssh.runner01.example.com", "service": "ssh://localhost:22"}]}` | 拉取配置、启动 tunnel、健康检查、上报 `tunnel_ready` |
| `install_mcp` | `{"source": "git-url", "config": {}}` | 安装 MCP Client/Server |
| `install_docker` | `{}` | 安装 docker/podman |
| `configure_ssh` | `{"authorized_keys": [""]}` | 写入 authorized_keys |
| `start_service` / `stop_service` | `{"service": "oracle-agent"}` | 管理本机 systemd 服务 |

### 6.3 扩展规则

- 新增能力 = 在 Agent `handlers/<type>.py` 新增 handler + `handlers/__init__.py` 注册表登记。
- Server 对 `type/params/result` **不透明**（任意 JSON，仅存档），天然支持加类型；可按类型配置默认超时（后续版本）。
- 不建议把"任意 shell 字符串"作为唯一协议（可维护性与可审计性差）；shell 应包在 `exec` 类型里，显式带数组命令与超时。

## 7. 错误码

统一错误体：
```json
{"error": {"code": "bad_request", "message": "human readable"}}
```

| HTTP | code | 场景 |
|---|---|---|
| 400 | `bad_request` | JSON 畸形 / 缺必填字段 / 参数越界 |
| 401 | `unauthorized` | 缺失或无效 Bearer token |
| 403 | `forbidden` | token 与 agent_id 不匹配 |
| 404 | `not_found` | agent/task 不存在 |
| 409 | `conflict` | result 提交到非 running/assigned 状态且非幂等 |
| 500 | `internal_error` | 服务端异常 |

## 8. 安全说明

- 生产必须 TLS（反向代理或 uvicorn ssl）；令牌经 Bearer 头传输。
- token 仅在 register 返回一次；Server 存 sha256 哈希；校验用 `compare_digest`。
- Runner 不需要任何公网入站端口；Cloudflare Tunnel 等服务经任务动态下发后才有入站能力。
- v1 不做限流（本地/可信环境）；生产建议按 agent 限速。

## 9. 附录 A：管理端操作（Oracle 侧）

已实现（`python -m server.ctl`）：
- `list`：列出 Agent（在线/离线/能力）与最近任务。
- `task push <agent_id> <type> '<json params>' [--timeout N]`：创建任务（`pending`），等 Agent pull。
- `task status <task_id>` / `task list [--agent X] [--status S]`：查看状态与结果。

规划中：`agent reset-token <agent_id>`（重置 token 需 Agent 端重新初始化 state.json 后以新身份注册）。

## 10. 附录 B：GitHub Bootstrap 最小化原则（已落地）

GitHub 仓库只负责"把一个最小可用 Runner 变出来"：

1. checkout → 初始化 SSH（改密码/密钥）
2. 安装 Oracle Agent（本仓库 `agent/` 目录 + install.sh）
3. Agent 首次启动自行 register，此后一切由 Oracle 下发

**禁止**放入仓库：SSH 操作全集、Cloudflare Tunnel 配置、业务脚本、长期任务队列、Runner 长期管理逻辑。
这些一律作为 Oracle 下发的任务存在（`install_cloudflare`、`configure_tunnel`、`exec`…）。
---

## 11. 附录 C：Agent Stream v1.1（反向 MCP 通道）

> 新增能力，**不修改** v1 现有 5 个接口（register/heartbeat/pull/result/report）。
> 设计目标：Runner 不开放入站、不建 Cloudflare Tunnel，也能被 Oracle 当作**实时交互通道**使用（当前用途：MCP）。

### 11.1 传输层定义（重要修正）

传输是**一条长连接的流式 HTTP 请求/响应**，不是"HTTP 双向"：

```
Runner ──POST /api/v1/connect──► Oracle
          │  请求体 = 上行流（chunked）   hello / ping / mcp_response
          ▼
Oracle   ──响应体 = 下行流（NDJSON）──► Runner
                    mcp_request / ping / close
```

- **帧编码**：NDJSON——每条消息一个 JSON 对象，`\n` 分隔。
- **上行**：Runner 向请求体持续写帧；**下行**：Server 向响应体持续推帧。
- **性质**：HTTP 本身是 request/response；这里的"双向"是靠**长连接 + 两端各自流**实现的，必须显式处理代理缓冲、空闲超时、断线重连（见 §11.5）。

### 11.2 端点

```
POST /api/v1/connect
Authorization: Bearer <token>        # 沿用 v1 认证，不另造
Content-Type: application/x-ndjson
```

- token 无效：立即 `401/403`（不建立流）。
- 同一个 `agent_id` 同时只允许一条活跃流；新连接成功后，Server **关闭旧流**（latest-wins，简化断线重连竞态）。
- 建立流的会话标识由 Oracle 生成：`stream_id`（仅用于日志/诊断，不是认证凭据）。

### 11.3 帧定义

所有帧共字段：`type`、`id?`（仅请求/响应类需要）。

**Runner → Oracle**

| type | 字段 | 说明 |
|---|---|---|
| `hello` | `agent_id` | 连接建立后**第一条**必须是 hello；Server 校验后回 `hello_ack` |
| `ping` | `ts` | 上行心跳（见 §11.5） |
| `mcp_response` | `id`、`result?`、`error?` | 必须与收到的 `mcp_request.id` 完全一致 |

**Oracle → Runner**

| type | 字段 | 说明 |
|---|---|---|
| `hello_ack` | `stream_id`、`heartbeat_sec` | Server 确认流建立，并告知心跳要求 |
| `mcp_request` | `id`、`method`、`params?`、`timeout_ms?` | 逻辑 MCP 方法调用（`tools/*`、`resources/*`、`prompts/*`、`initialize`、`ping`） |
| `ping` | `ts` | 下行心跳 |
| `close` | `reason` | 优雅关闭（运维/升级/限流）；Runner 可自行决定是否重连 |

示例：

```json
{"type":"hello","agent_id":"runner-01"}
{"type":"hello_ack","stream_id":"s-9f31","heartbeat_sec":30}
{"type":"mcp_request","id":"m1","method":"tools/list","params":{},"timeout_ms":30000}
{"type":"mcp_response","id":"m1","result":{"content":[{"type":"text","text":"{\"stdout\":\"...\"}"}]}}
{"type":"close","reason":"operator_reload"}
```

### 11.4 并发与 ID 规则

- 每个 `mcp_request` 由 **Oracle 生成唯一 `id`**（连接内唯一即可）；Runner **原样回显**。
- 允许并发 multiplex：Oracle 可在同一流上同时发出多个 `mcp_request`，Runner 按任意顺序回 `mcp_response`（无顺序保证，按 `id` 匹配）。
- Runner 未发起请求式帧（v1.1）；未来若支持，必须使用 `r_` 前缀的 id 空间避免冲突。
- 单个连接并发上限：默认 64（超出回 `mcp_error_response` code=`too_many_inflight`）。

### 11.5 生命周期 / 心跳 / 超时 / 重连

| 项 | 值 | 说明 |
|---|---|---|
| 上行心跳 | 每 `heartbeat_sec`（Server 在 hello_ack 下发，默认 30s） | Runner 必须发 `ping`；**任何上行帧都算活动** |
| 下行活动 | Server 空闲 30s 发 `ping` | 任何下行帧也算活动 |
| 静默判定 | 任一端 90s 未收到任何字节 | Server 关流；Runner 判定断线 |
| Cloudflare | 空闲 100s 会断 | 心跳 30s < 100s，且有 90s 静默判定兜底 |
| 断线重连 | 指数退避 1s→2s→4s…上限 60s + 随机抖动 | 重连后必须重发 `hello`；过程与"latest-wins"配合无竞态 |
| 断线时在途请求 | Oracle 对未收 `mcp_response` 的在途 `mcp_request` 回网关侧 `error(code=connection_lost)` | 不持久化；MCP 是实时通道，不落任务队列 |

### 11.6 错误帧

协议级错误：

```json
{"type":"error","code":"bad_frame","message":"invalid JSON line"}
{"type":"error","code":"unknown_type","message":"..."}
```

规则：收到畸形帧/未知 type → 回 `error`；**连续两次协议错误 → 关流**。MCP 方法错误（含超时、适配器不可达、后端错误）：

```json
{"type":"mcp_error_response","id":"<原 id>","error":{"code":"...","message":"..."}}
```

code 枚举：`unknown_method | timeout | backend_unreachable | too_many_inflight | internal_error | connection_lost`（`connection_lost` 由 Gateway 侧合成）。

### 11.7 Runner 侧组件（协议外约定，实现时照此）

```
Agent Stream（隶属于 Oracle Agent）
     │
     ▼
mcp_adapter.py      ← 桥接，纯 stdlib；不实现任何 MCP 工具
     │  逻辑方法调用 → 本地 MCP 会话
     ▼
127.0.0.1:6942/mcp  ← 现有 shell-mcp（HTTP/SSE），**不改动**
```

- Adapter 自行管理到本地 6942 的 MCP 会话（initialize → 复用会话）。
- `mcp_response.result` = 本地 MCP 返回的原始结果（Gateway 负责包装成客户端所需格式）。

### 11.8 Oracle 侧组件（协议外约定）

```
MCP Client（Cline / TG）
     │
     ▼  https://mcp-gateway.femboy.us.ci/mcp/<runner_id>   ← Cloudflare 只护这一层
MCP Gateway（VPS）
     │  initialize → 认证/路由 → 绑定 runner 流
     ▼
Runner connection（/api/v1/connect）
```

- 路由：路径显式指定目标 runner（`/mcp/runner-01`）；或由 Gateway 配置/认证身份决定。
- Gateway 与 Runner 流之间是 1:1 绑定；客户端断开不销毁 runner 流（可再连）。

### 11.9 兼容性

- `/pull`、`/result`、`/report` 及 Task API 全部保留：**Task = 生命周期/后台任务**，**Stream = 实时交互**，职责分离。
- 本附录是 v1 的**增量**，服务端与 Agent 未实现前不影响 v1 运行。
