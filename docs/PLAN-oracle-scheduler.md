# 计划：Oracle Runner 调度与部署系统（Task Template + 多 GitHub 账号）

> 状态：**计划落盘，未实施**（2026-09）
> 本文档合并了此前两份草案（multi-PAT / Task Template）并吸收最新调整：
> - Workflow 只负责"把 Runner 拉起来"；业务部署全部移交 Oracle。
> - Template = 部署任务（named steps 线性依赖），Secret 独立存储只留引用。
> - 不依赖"多个 exec 天然 FIFO"（Agent Stream 未来支持并发 multiplex），模板显式顺序。
> - 与多账号（Credential Profile）在 `/admin/deploy` 组合。

---

## Part A：GitHub Account / Credential Profile（多账号扩容）

### A1 概念分层

```
GitHub Account (Credential Profile)
      │  PAT
      ▼
GitHub Workflow
      │  Run
      ▼
Oracle Run
      │  Agent
      ▼
Runner
```

- PAT 不与 Runner/Agent 绑定；Agent 完全不知道 PAT/GitHub 用户/repo，协议零变化。
- 多 profile 可同 owner 不同 PAT（primary / primary-2 / secondary）。

### A2 数据模型

```sql
CREATE TABLE github_accounts (
  id                  INTEGER PRIMARY KEY AUTOINCREMENT,
  name                TEXT UNIQUE,      -- primary / secondary / primary-2
  owner               TEXT,
  repo                TEXT,
  credential          TEXT,             -- PAT，仅服务端私用
  enabled             INTEGER DEFAULT 1,
  max_concurrent_runs INTEGER DEFAULT 20,   -- 配置化，不写死
  created_at          TEXT, updated_at TEXT
);
```

runs 表加 `account_id`；现有行回填默认 profile。

### A3 API（Phase 2）

| 端点 | 改动 |
|---|---|
| `POST /admin/trigger` | 加可选 `account` |
| `GET /admin/github/workflows` | 加可选 `account` |
| `POST /admin/cancel` | 按 `run.account_id` 自动反查 PAT，调用者零账号信息 |
| `GET /admin/accounts`（新） | in_progress / max_concurrent_runs / 余量 |

调度规则仅：`当前 in_progress < max_concurrent_runs` → 可分发。

---

## Part B：Task Template（部署任务模板）

### B1 定位

```
GitHub Workflow（最小）
    ├── SSH           ← 排障备用入口
    ├── shell-MCP     ← Runner 基础能力
    ├── Oracle Agent  ← 控制入口
    └── sleep 99999   ← 维持 Job
```

Xray / cloudflared / atlas / Telegram Bot **全部移出 Workflow**。Workflow 不需要知道 atlas1 是什么。

### B2 数据模型：步骤与 Secret 分离

```sql
CREATE TABLE task_templates (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  name        TEXT UNIQUE,        -- atlas1 / atlas2 / ...
  description TEXT,
  steps       TEXT,               -- JSON：named steps，线性依赖
  enabled     INTEGER DEFAULT 1,
  created_at  TEXT, updated_at TEXT
);

CREATE TABLE template_secrets (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  template_id INTEGER REFERENCES task_templates(id),
  key         TEXT,               -- CF_TUNNEL_TOKEN / BOT_TOKEN / CHAT_IDS
  value       TEXT                -- 真正 secret 单独存储
);
```

- Template 只持有 **secret references**（key 名）；值在 `template_secrets`。
- `GET /admin/templates/{name}` 只返回 `"secrets": ["CF_TUNNEL_TOKEN","BOT_TOKEN","CHAT_IDS"]`，**永不返回值**。

### B3 Steps 语义（关键调整）

```json
{
  "name": "atlas1",
  "steps": [
    { "name": "install_xray",       "command": ["bash","-c","..."], "timeout_sec": 120 },
    { "name": "validate_xray",      "command": ["xray","-test","-c","/etc/xray/config.json"] },
    { "name": "start_xray",         "command": ["bash","-c","systemctl enable --now xray"] },
    { "name": "install_cloudflared", "command": ["bash","-c","cloudflared service install"], "env_refs": ["CF_TUNNEL_TOKEN"] },
    { "name": "install_atlas",      "command": ["bash","-c","..."] },
    { "name": "install_tgbot",      "command": ["bash","-c","..."],  "env_refs": ["BOT_TOKEN","CHAT_IDS"] }
  ]
}
```

- **模板步骤显式顺序依赖**：Step N+1 只在 Step N `success` 后下发；失败即标记 template run 失败。
  - 不依赖"多个 exec 自然 FIFO"——Agent Stream 未来支持 request A/B/C 并发 multiplex，顺序不再可靠。
- 演进目标：Template Run / Task Run（独立执行实体，含步骤级状态、结果、重试）。
- 实现提示（后续）：Template Run 执行器在 result 回调触发下一步，可后台线程/惰性驱动。

### B4 Secret 注入（沿用 exec env 透传设计）

- Agent exec handler 支持 `env` 参数 → `subprocess.run(env={**os.environ, **env})`（小改动，向后兼容）。
- Token 经 env 注入、不进命令字符串；访问日志对 `env` 值整体打码。

### B5 atlas1 拆分（去掉 SSH/MCP——Bootstrap 已含）

```
atlas1
├── 01 install_xray        mkdir /var/log/v2ray; 下载 Xray; 装 /usr/local/bin/xray; 配置 /etc/xray; 装 xray.service
├── 02 validate_xray       xray -test
├── 03 start_xray          daemon-reload; enable; start
├── 04 install_cloudflared 下载 cloudflared; 装 /usr/local/bin; service install（CF_TUNNEL_TOKEN 来自 secrets）
├── 05 install_atlas       下载 atlas.service; 装 service; enable/start
└── 06 install_tgbot       下载 logsender.py + tgbot.service; 注入 BOT_TOKEN/CHAT_IDS; enable/start
```

### B6 分层（与现有 Agent 架构吻合）

```
              GitHub
                │
          Bootstrap Runner
                ▼
        ┌─────────────────┐
        │      Agent      │
        └────────┬────────┘
                 │  Agent Stream
                 ▼
        ┌─────────────────┐
        │ Oracle Controller│
        └────────┬────────┘
       ┌─────────┴─────────┐
   Control Plane       Template Plane
   exec/shutdown       atlas1 / atlas2
   install/service
```

- Template 最终通过 **Control Plane 的 exec** 执行，**不新建 Agent 协议**。
- 计划路径：`POST /admin/templates` 建模板 → `POST /admin/templates/{name}/apply {"run_id": N}` → 按序执行。

---

## Part C：组合编排（Phase 2）

```
POST /admin/deploy
{
  "account": "secondary",
  "template": "atlas1"
}
```

```
account secondary → PAT → trigger workflow → Run queued → provisioning
→ Agent registered → Template atlas1 → Step1 → Step2 → ... → deployed
```

- 与 multi-PAT 组合：GitHub 账号决定"在哪开 VM"，Template 决定"装什么"。
- GitHub 越来越像临时 VM 提供器，业务部署逻辑归 Oracle。

---

## Part D：实施顺序 / 非目标

### Phase 划分

- ✅ Phase 1（已完成）：Run→Agent 定位（run_id/id/vm_id/agent_id + 状态机 + resolved_by）。
- ⏳ Phase 2A：Task Template（表 + CRUD + apply 执行器 + exec env 透传 + 日志打码）。
- ⏳ Phase 2B：多账号（github_accounts + runs.account_id + 按账号路由 + admin/accounts）。
- ⏳ Phase 3：组合编排 `/admin/deploy` + Template Run 增强（重试/步骤状态/告警）。

### 非目标 / 约束

- 不改 agent 协议；Template 经 Control Plane exec 执行。
- Secret 永不通过 API 返回；只在服务端存储与注入；日志打码。
- Workflow 保持"最小 Bootstrap"，业务零渗透。

## 关联现有代码

- 凭据：`server/api/admin.py` `_gh_creds_or_raise()`（单账号，Phase 2B 改按 profile）
- 生命周期：`server/db.py` runs 表 + `create_run_provisioning / sync_run_from_github`
- 执行：`server/api/admin.py` `_resolve_exec_target`（Phase 2A 复用：模板步骤经它定位 run/agent）
- Agent：`agent/handlers/exec.py` 加 `env` 透传（小改）