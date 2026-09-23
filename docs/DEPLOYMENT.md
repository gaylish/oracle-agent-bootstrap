# Oracle Agent 部署文档（v1）

> 原文档随架构演进多次修订；本版为最终稳态：GitHub Bootstrap + Oracle 控制器（VPS）+ Runner Agent（出站拉取）。

## 1. 部署环境

- **主机**：Oracle Cloud VPS `vnic-1`（Ubuntu 24.04 aarch64，公网 IP `213.35.122.16`）
- **访问**：SSH `HuaweiAgent@213.35.122.16:22`（备用通道：MCP 隧道 `127.0.0.1:13005`）
- **控制器**：`/opt/oracle-agent/`，专用用户 `oracle-agent`，systemd 单元 `oracle-agent.service`
- **监听**：`127.0.0.1:8700`（**仅本机**；公网入口经 Cloudflare 隧道，VCN 无开放端口）

## 2. 网络拓扑（生产）

```
Runner ──outbound HTTPS──► https://oracle-agent.femboy.us.ci
                                  │ Cloudflare 隧道（自动 TLS）
                                  ▼
                        VPS cloudflared ──► http://127.0.0.1:8700
                                            oracle-agent.service
```

- 隧道：token 隧道（ID `f316b3f1-8693-4e0c-b03e-3af886e4c50b`，Zero Trust 远程托管）
- 公网主机名：`oracle-agent.femboy.us.ci → http://localhost:8700`（面板配置，云端自动下发）

## 3. 已部署组件

| 组件 | 位置 (VPS) | 说明 |
|---|---|---|
| Oracle 控制器 (Server) | `/opt/oracle-agent/server/` | FastAPI，`oracle-agent.service` 常驻 |
| Python 环境 | `/opt/oracle-agent/venv/` | fastapi + uvicorn |
| 数据库 | `/opt/oracle-agent/server/data/oracle.db` | SQLite，agents/tasks 两表 |
| 管理 CLI | `/opt/oracle-agent/venv/bin/python -m server.ctl` | 以 `oracle-agent` 用户运行 |

## 4. 远程管理命令

```bash
# 服务状态 / 日志
systemctl status oracle-agent
journalctl -u oracle-agent -f

# 查看注册表/任务
sudo -u oracle-agent bash -c 'cd /opt/oracle-agent && venv/bin/python -m server.ctl list'

# 下发任务（test / exec / shutdown / 预留类型）
sudo -u oracle-agent bash -c 'cd /opt/oracle-agent && venv/bin/python -m server.ctl task push <agent_id> test "{\"hi\":\"there\"}"'
sudo -u oracle-agent bash -c 'cd /opt/oracle-agent && venv/bin/python -m server.ctl task push <agent_id> exec "{\"command\":[\"echo\",\"hello\"],\"timeout_sec\":30}"'
sudo -u oracle-agent bash -c 'cd /opt/oracle-agent && venv/bin/python -m server.ctl task push <agent_id> shutdown "{}"'

# 任务状态
sudo -u oracle-agent bash -c 'cd /opt/oracle-agent && venv/bin/python -m server.ctl task status <task_id>'
```

## 5. 从零部署清单（若需重建）

```bash
# 1. 代码 + venv
sudo mkdir -p /opt/oracle-agent
sudo tar xzf oracle-agent-server.tar.gz -C /opt/oracle-agent   # server/ 目录
sudo python3 -m venv /opt/oracle-agent/venv
sudo /opt/oracle-agent/venv/bin/pip install fastapi uvicorn

# 2. 专用用户
sudo useradd --system --home /opt/oracle-agent --shell /usr/sbin/nologin oracle-agent
sudo chown -R oracle-agent:oracle-agent /opt/oracle-agent

# 3. systemd 单元（仓库 server 侧同款；绑定 127.0.0.1:8700）
sudo cp oracle-agent.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now oracle-agent

# 4. 验证
curl -s http://127.0.0.1:8700/healthz        # {"ok":true}

# 5. 隧道（Cloudflare Zero Trust 面板）
#    Public Hostname: oracle-agent.femboy.us.ci → http://localhost:8700
```

## 6. 验证记录（2026-09-23）

### 6.1 VPS 本地闭环

| 任务 | 状态 | 结果 |
|---|---|---|
| test | success | `{"echo": {...}, "message": "hello from runner"}` |
| exec echo | success | stdout `hello from runner`, exit_code 0 |
| exec exit 1 | failed | stdout `oops`, exit_code 1 |
| exec sleep 5 (limit 2s) | timeout | error `command timed out after 2s` |

### 6.2 公网隧道闭环（agent 指向 `https://oracle-agent.femboy.us.ci`）

| 任务 | 状态 | 结果 |
|---|---|---|
| test | success | `hello from runner`（经 CF 隧道） |
| exec echo | success | stdout `hello via CF tunnel`, exit_code 0 |

> 坑位记录：Cloudflare 默认拦截 `Python-urllib/*` User-Agent（error 1010）；agent 已改为自定义 UA `oracle-agent/0.1.0`。

### 6.3 GitHub Actions 全链路（真实 Runner）

仓库：`sourspicysoup/oracle-agent-bootstrap`（private），workflow `Oracle Runner Bootstrap`，run #35849192311

| 环节 | 结果 |
|---|---|
| 触发 workflow → 新建 GH 托管 VM | ✅ runner 上线 |
| VM 上 SSH（`runner:runner` + sshd）/ shell-mcp | ✅（后续步骤能跑即证明） |
| 安装 Oracle Agent → 注册 | ✅ `runner-runnervmlun5p` online |
| Oracle 下发 `test` / `exec` | ✅ success（拿到 runner 公网 IP `172.182.211.21`） |
| Oracle 下发 `shutdown` | ✅ VM 断电 → GH job `completed(failure)`（预期） |

**结论**：GitHub 只做 Bootstrap；Runner 出站 HTTPS 经 CF 隧道连 Oracle；任务下发与生命周期（关机）由 Oracle 掌握——与设计一致。

## 7. 已完成清理项

- ✅ 旧架构 `proxy-manager`（8000 端口）删除；参考代码存档 `docs/reference/proxy-manager/`
- ✅ PostgreSQL `mailrelay` 库删除
- ✅ 测试 agent 目录与临时文件清理
- ✅ 旧 token 误建的空仓库 `femboyenjoy/oracle-agent-bootstrap` 删除

## 8. 待办候选

- **Agent Stream v1.1（反向 MCP 通道）**：协议已定（协议附录 C）；实现 Runner 侧 `mcp_adapter.py` + Oracle 侧 MCP Gateway，让 MCP 复用 Runner 出站连接
- Cloudflare Access Service Token 加固（Runner 端带 header）
- 协议 `ack` 租约语义、重投递上限（见协议 §5.2）
- 新任务类型：`install_cloudflare` / `configure_tunnel` / `install_mcp` / `install_docker` 等（协议 §6.2）