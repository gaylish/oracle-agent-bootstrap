# 计划：Oracle Runner 调度与部署系统（Task Template + 多 GitHub 账号 + Supervisor 常驻守护）

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

### A4 不对外开放 / 路径隐藏（2026-09 定）

- **多账号模块 API 不对外开放**：账号管理（github_accounts 增删改、`/admin/accounts` 并发视图）一律 `include_in_schema=False`，**不出现在 Swagger**，仅服务端内部 / agentctl 使用。
- 公开端点（`trigger` / `cancel` / `github/workflows`）只加**可选 `account` 参数**（不算新模块，对外无新增管理路径）。
- cancel 按 `runs.account_id` 自动反查 PAT——调用者不需要知道账号，自然无需暴露账号管理。

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

### B7 模板不在公开 API（2026-09 定）

- **API 不做 templates 模块**：移除原计划的 `/admin/templates` CRUD 公开端点。
- `task_templates` / `template_secrets` 表由**服务端配置 / 本地 CLI（agentctl）直接维护**（或独立配置文件）。
- 执行入口（apply）同样**隐藏**：`include_in_schema=False`（不出现在 Swagger），或仅 agentctl / 服务端内部调用；模板与 secret 值不进公网 API。

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

## Part E：Supervisor（常驻任务守护 / Keep-alive 调度）

> 需求来源：原实现用 **Cloudflare Worker（PAT + 定时触发器）** 每分钟检查 atlas1 是否在跑。
> 迁入 Oracle 后：不用敲 CF Worker 代码，多 PAT 天然续命，状态可审计。

### E1 目标

对指定 workflow（如 `gaylish/codespace/.github/workflows/atlas1.yml`）做**常驻守护**：

- 每 60s 检查一次"有没有正在运行的 run"
- 若**没有运行中的 run** → 触发一个新的
- 若**最老的运行中 run 已跑满 5h55m（21300s）** → 再触发一个新的（滚动换台；`renew_after_sec` 可配）
- 多账号（Profiles）下：当前账号失效/超限 → **自动换下一个可用账号的 PAT** 触发，保证任务始终有 agent

> 5h55m 的由来：GitHub 托管 job 上限 6h（360min），在到期前滚动换新避免"裸奔空窗"。

### E2 数据模型（服务端配置，非公开 API）

```jsonc
// 服务端配置（如 /etc/default/oracle-agent/supervisors.json）
{
  "supervisors": [
    {
      "name": "atlas1",
      "repo": "gaylish/codespace",
      "workflow_file": ".github/workflows/atlas1.yml",
      "accounts": ["primary", "secondary"],     // 触发顺序：失败/超限自动换下一个（多 PAT 续命）
      "check_interval_sec": 60,
      "min_running": 1,
      "renew_after_sec": 21300,                  // 5h55m
      "template": "atlas1",                      // 可选：Phase 2A，agent 注册后自动 apply
      "enabled": true
    }
  ]
}
```

### E3 巡检逻辑（Oracle 后台循环）

```
每 check_interval_sec（60s）：
  for each supervisor(enabled):
    runs = GitHub API(当前 profile PAT) 查 repo/workflow 的 in_progress runs
    live = [r for r in runs if start_time < now - renew_after_sec? 不在换台窗口]
    if len(live) < min_running:
        trigger_dispatch(当前 profile)          // 触发失败 → 换下一个 profile 再试
    elif oldest_start < now - renew_after_sec:
        trigger_dispatch(当前 profile)          // 滚动换台（可选 discard 旧的：cancel 旧 run）
```

- 实现形态：Oracle 进程内后台 asyncio 循环（或独立线程），配置驱动；**不在 Swagger**，状态仅 agentctl / 隐藏端点可查。
- API 节流：60s × N supervisor 的 GitHub 查询要合并/限频，避免触发 rate limit。

### E4 与其它部分的组合

- **多 PAT（Part A）**：`accounts` 顺序即故障转移链；账号失效自动切下一个。
- **Task Template（Part B）**：可选——office atlas1.yml 已是完整 workflow（自带业务+keep-alive），supervisor 只需"保活"；若工作流最小化，则 supervisor 触发 bootstrap 后由 `template` 布置业务。
- **生命周期（现有 runs 状态机）**：supervisor 写 run 档案照常（queued→provisioning→running→终态），`renew` 决策可基于 runs 表/ GitHub 双向。

### E5 边界

- 与 CF Worker 的差异：不再需要单独部署 Worker；PAT 不出 Oracle；切换/告警/审计可控。
- 不接管通用调度（先做单 supervisor 常驻；多 supervisor、优先级等后续）。

---
## Part D：实施顺序 / 非目标

### Phase 划分

- ✅ Phase 1（已完成）：Run→Agent 定位（run_id/id/vm_id/agent_id + 状态机 + resolved_by）。
- ⏳ Phase 2A：Task Template（表 + **本地 CLI/服务端维护（无公开 CRUD）** + apply 隐藏端点/agentctl + exec env 透传 + 日志打码）。
- ⏳ Phase 2B：多账号（github_accounts + runs.account_id + 按账号路由 + **模块 API 隐藏、不对外开放**）。
- ⏳ Phase 3：组合编排 `/admin/deploy` + Template Run 增强（重试/步骤状态/告警）。
- ⏳ Phase 4：**Supervisor 常驻守护**（`supervisors` 配置 + 后台巡检循环 + 滚动换台 + 多 PAT 故障转移）。

### 非目标 / 约束

- 不改 agent 协议；Template 经 Control Plane exec 执行。
- Secret 永不通过 API 返回；只在服务端存储与注入；日志打码。
- Workflow 保持"最小 Bootstrap"，业务零渗透。

## 关联现有代码

- 凭据：`server/api/admin.py` `_gh_creds_or_raise()`（单账号，Phase 2B 改按 profile）
- 生命周期：`server/db.py` runs 表 + `create_run_provisioning / sync_run_from_github`
- 执行：`server/api/admin.py` `_resolve_exec_target`（Phase 2A 复用：模板步骤经它定位 run/agent）
- Agent：`agent/handlers/exec.py` 加 `env` 透传（小改）