# Oracle Runner Bootstrap

GitHub Actions **一次性 Bootstrap**：把一台 GitHub Runner 变成"已初始化 + 可被 Oracle 控制"的机器。

架构（Bootstrap 后 GitHub 不再参与）：

```
Runner ──HTTPS(outbound)──► oracle-agent.femboy.us.ci ──Cloudflare隧道──► Oracle VPS :8700
```

## 这个仓库做什么

Runner 上跑完 workflow 后：

1. **SSH**：修改密码 `runner:runner`，启用 sshd
2. **MCP**：安装 shell-mcp（systemd 常驻）
3. **Oracle Agent**：安装到 `/opt/oracle-agent` 并注册到 Oracle 控制器（`oracle-agent.service` 常驻，出站轮询拉任务）

之后 Runner 只出站、不开放入站，一切任务由 Oracle 下发（pull 长轮询）。

## 手动触发

GitHub 仓库 → **Actions** → **Oracle Runner Bootstrap** → **Run workflow**：

- `agent_name`：显示在 Oracle 里的名字（默认 `runner-gh`）
- `oracle_url`：Oracle 控制器地址（默认 `https://oracle-agent.femboy.us.ci`）

## 在 Oracle 侧查看

```bash
# VPS 上
sudo -u oracle-agent bash -c 'cd /opt/oracle-agent && venv/bin/python -m server.ctl list'
sudo -u oracle-agent bash -c 'cd /opt/oracle-agent && venv/bin/python -m server.ctl task push <agent_id> test "{\"hello\":\"runner\"}"'
sudo -u oracle-agent bash -c 'cd /opt/oracle-agent && venv/bin/python -m server.ctl task push <agent_id> exec "{\"command\":[\"echo\",\"hello\"],\"timeout_sec\":30}"'
```

## 目录

| 路径 | 说明 |
|---|---|
| `.github/workflows/bootstrap.yml` | 手动触发的 bootstrap 工作流 |
| `agent/` | Oracle Agent 客户端（纯标准库，零依赖） |
| `mcp/`   | shell-mcp 服务端（HTTP/SSE，127.0.0.1:6942），vendor 自 `pornnewbee/socks5-for-serv00`（本人仓库），原样保留 |

## 注意

- 这是"一次性 Bootstrap"通道：长期控制走 Oracle Agent 任务协议，不要把业务脚本堆进这个仓库
- workflow 里直接写死 `runner:runner` SSH 密码，仅用于自有 GitHub Runner 初始化
- MCP（`mcp/`）为 vendor 版本：**原样保留，不主动改**；上游如需更新，手动同步 `mcp/shell_mcp.py`，并同步 venv 依赖版本（`mcp==2.2.0` 等，见 workflow）