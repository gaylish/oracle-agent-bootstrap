# Oracle Runner Agent (v1)

纯 Python 标准库（零第三方依赖）的 Runner 控制客户端，实现 [Oracle Agent Protocol v1](../docs/oracle-agent-protocol-v1.md)。

## 目录

```
agent.py                主循环：register → heartbeat/pull → execute → report
config.json             配置（oracle_url、agent_name、capabilities…）
state.json              运行时生成：agent_id + token（chmod 600，仅注册响应写入一次）
handlers/               任务类型处理器注册表
  test.py               test 任务：连通性验证，返回 "hello from runner"
  exec.py               exec 任务：带超时的命令执行，返回 stdout/stderr/exit_code
oracle-agent.service    systemd unit
install.sh              安装到 /opt/oracle-agent 并注册服务
```

## 本地运行（联调）

```bash
# 先确认 server 已启动，然后：
cd agent
python3 agent.py
```

首次启动会自动 `POST /api/v1/agent/register`，把返回的 token 写入 `state.json`（600 权限）。之后启动都会带 token 重注册（幂等）。

## 安装为系统服务（Runner 上）

```bash
# 先改 config.json 里的 oracle_url 为 Oracle 服务器地址
sudo bash install.sh
```

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