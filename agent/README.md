# Oracle Runner Agent (v1)

纯 Python 标准库（零第三方依赖）的 Runner 控制客户端，实现 [Oracle Agent Protocol v1](../docs/oracle-agent-protocol-v1.md)。

## 目录

```
agent.py                主循环：register → heartbeat/pull → execute → report
config.json             配置（oracle_url、agent_name、capabilities、频率参数、user_agent）
state.json              运行时生成：agent_id + token（chmod 600，仅注册响应写入一次）
handlers/               任务类型处理器注册表
  test.py               test 任务：连通性验证，返回 "hello from runner"
  exec.py               exec 任务：带超时的命令执行，返回 stdout/stderr/exit_code
  shutdown.py           shutdown 任务：Oracle 下发关机指令（默认延迟 1 分钟，先回报再断电）
oracle-agent.service    systemd unit
install.sh              安装到 /opt/oracle-agent 并注册服务（支持 ORACLE_URL / AGENT_NAME 覆盖）
```

## 配置项（config.json）

| 键 | 默认 | 说明 |
|---|---|---|
| `oracle_url` | 仓库默认 `https://oracle-agent.femboy.us.ci` | Oracle 控制器公网地址 |
| `agent_name` | `runner-gh` | 显示名；`agent_id` 缺省时生成为 `runner-<hostname>` |
| `agent_id` | 空 | 显式指定 agent_id（一般不需要） |
| `capabilities` | `["ssh","agent"]` | 上报给 Oracle 的能力列表 |
| `heartbeat_interval_sec` | `60` | 心跳间隔；Server 超 180s 未见心跳判离线 |
| `pull_timeout_sec` | `30` | pull 长轮询挂起时长；任务拾取最差延迟 ≤ 该值 |
| `user_agent` | `oracle-agent/0.1.0` | **必填自定义 UA**：Cloudflare 会拦截 `Python-urllib/*`（error 1010） |

## 本地运行（联调）

```bash
# 先确认 server 已启动，然后：
cd agent
python3 agent.py
```

首次启动自动 `POST /register`，把返回的 token 写入 `state.json`（600 权限）；之后启动带 token 重注册（幂等，保持同一 agent_id）。

## 安装为系统服务（Runner 上）

```bash
# 默认用 config.json；或环境变量覆盖：
sudo ORACLE_URL=https://oracle-agent.femboy.us.ci AGENT_NAME=runner-01 bash install.sh
```

## 通信频率（运维关注）

- `pull`：每 `pull_timeout_sec`（30s）一次长轮询；Oracle 建任务后最差 30s 被拉走
- `heartbeat`：每 `heartbeat_interval_sec`（60s）一次
- 单 Runner 流量 ≈ 每小时 120 次 pull + 60 次心跳，每次 < 1KB

## 扩展新任务类型

在 `handlers/` 新增 `<type>.py`：

```python
from . import HandlerError, HandlerTimeout, register

@register("install_cloudflare")
def run(params: dict) -> dict:
    # ... 安装/配置/健康检查 ...
    if bad:
        raise HandlerError("cloudflared install failed", output={...})
    return {"status": "installed", "version": "..."}
```

Server 对 type/params/result 不透明，无需改动 Server 即可支持新类型。

## 停止

```bash
sudo systemctl stop oracle-agent   # SIGTERM → 优雅退出
```