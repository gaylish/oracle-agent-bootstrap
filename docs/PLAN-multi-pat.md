# 计划：多 GitHub 账号（Multi-PAT）支持与任务路由

> 状态：**计划落盘，未实施**（2026-09）
> 背景：单 GitHub 账号存在并发 run 上限（如 20），需要多账号横向扩容；并支持把任务派给指定账号下的 workflow/runner 执行。

---

## 1. 目标

1. **多 PAT 扩容**：一个 Oracle 管理多个 GitHub 账号的 runner，突破单账号并发上限。
2. **任务路由**：把任务分配给指定 GitHub 账号下的 workflow/runner 执行。

## 2. 设计

### 2.1 凭据注册表（服务端，root 600）

建议文件：`/etc/default/oracle-agent.json`（与现有 env 文件 `/etc/default/oracle-agent` 并存）

```json
{
  "accounts": [
    { "name": "primary",   "owner": "sourspicysoup", "pat": "ghp_...", "repos": ["oracle-agent-bootstrap"] },
    { "name": "secondary", "owner": "femboyenjoy",  "pat": "ghp_...", "repos": ["oracle-agent-bootstrap"] }
  ],
  "default": "primary"
}
```

- PAT 只存服务端，绝不通过 API 输出；访问日志中的 Authorization 已打码。
- 兼容现状：未配置 accounts 时退化为现有 `GH_PAT / GH_OWNER / GH_REPO` 单账号模式。

### 2.2 runs 表加账号维度

```sql
ALTER TABLE runs ADD COLUMN account TEXT;
ALTER TABLE runs ADD COLUMN repo   TEXT;
-- 现有行回填："primary"
```

- run 档案带 `account`；cancel / exec 可反查归属账号，语义闭环。

### 2.3 API 改动（向后兼容）

| 端点 | 改动 |
|---|---|
| `POST /api/v1/admin/trigger` | 加可选 `account`（默认 default）→ 用该账号 PAT 发 dispatch |
| `GET /api/v1/admin/github/workflows` | 加可选 `account`；不加则列默认/全部 |
| `POST /api/v1/admin/cancel` | 按 run 的 `account` 自动选 PAT；可加 `account` 覆盖 |
| `GET /api/v1/admin/accounts`（新） | 各账号当前 in_progress 数 / 上限，展示扩容余量 |

### 2.4 任务路由（两种粒度）

**A. 组合式（改动最小）**
`trigger(account=B)` → 等该 run agent 注册 → `exec(run_id)`。
路由 = "在哪个账号下开 run"，复用现有 trigger/exec 原语，适合手动/脚本编排。

**B. 原子式（后期增强，基于生命周期模型）**
```
POST /api/v1/admin/exec-new
{ "account": "secondary", "command": [...], "wait_until": "agent_ready" }
```
→ Oracle 建 provisioning run → 触发该账号 workflow → **agent 注册即自动下发 exec**
（复用 queued→running 状态机 + AutoExec 标记）。一条调用完成"任务 → 指定账号 runner"。

### 2.5 扩容策略

- `trigger count=N` 时按各账号 in_progress 余量分发（round-robin 或手动指定 account）。
- 不把单账号并发当瓶颈，Oracle 统一调度。

## 3. 待确认项

1. 凭据文件路径：`/etc/default/oracle-agent.json` 是否 OK？
2. 路由先做 **A 组合式** 还是直接 **B 原子式**？
3. `repo` 维度是否要（不同账号不同仓库跑不同 workflow）？
4. 是否需要 `admin/accounts` 阈值告警（账号达到并发上限时怎么提示）？

## 4. 实施顺序（排期参考）

1. 凭据注册表加载 + 单/多账号适配（trigger/workflows/cancel 按 account 路由）
2. runs 表 account 列迁移 + 档案回显
3. `admin/accounts` 并发余量视图
4. 路由 A（组合式，文档示例）
5. （可选）路由 B：exec-new + AutoExec

## 5. 非目标 / 约束

- 不改 agent 端协议：agent 仍然只认 Oracle URL + register，与账号无关（设计天然支持）。
- 多账号仅影响"开 run / 停 run / 查 GitHub"这些服务端动作；agent 生命周期/exec/logs 全部复用现有层。
- PAT 安全：root 600 文件、服务端私用、日志打码、不通过 API 泄露。

## 6. 关联现有代码

- 凭据现读取：`server/api/admin.py` 的 `_gh_creds_or_raise()`（单账号）
- 生命周期：`server/db.py` runs 表 + `create_run_provisioning / sync_run_from_github`
- 执行解析：`server/api/admin.py` `_resolve_exec_target`（解析与判断分离，多账号不侵入此层）