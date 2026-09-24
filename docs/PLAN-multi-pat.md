# 计划：GitHub Account/Credential Profile 支持与任务路由（v2）

> 状态：**计划落盘，未实施**（2026-09）
> v2 要点：把 "GitHub 账号" 作为 Run 的**一等属性**设计进去，即使多 PAT 暂不实现；
> 概念上明确为 **Account / Credential Profile**（∎ 不是 github_user，允许多 profile 同 owner 不同 PAT）。
> 关键边界：**Phase 1（Run→Agent 定位）已完成；Phase 2（account_id→PAT→workflow→Run）后续单独实施**，两者不混。

---

## 1. 概念分层

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

- **PAT 不与 Runner/Agent 绑定**：Agent 完全不知道 PAT / GitHub 用户 / repo，协议零变化。
- 多个 GitHub 账号（甚至同 owner 多 PAT）都由同一个 Oracle Controller 管理。

## 2. 数据模型

### 2.1 github_accounts（Credential Profile）

```sql
CREATE TABLE github_accounts (
  id                 INTEGER PRIMARY KEY AUTOINCREMENT,
  name               TEXT UNIQUE,     -- 逻辑名：primary / secondary / primary-2
  owner              TEXT,            -- GitHub 用户名
  repo               TEXT,            -- 单 profile = 单一仓库
  credential         TEXT,            -- PAT 等，仅服务端私用
  enabled            INTEGER DEFAULT 1,
  max_concurrent_runs INTEGER DEFAULT 20,   -- 不写死：配置化
  created_at         TEXT,
  updated_at         TEXT
);
```

- 一个 GitHub 用户可对应多个 profile（不同 PAT / 仓库 / 权限）：
  `primary(userA, token-1)` + `primary-2(userA, token-2)` + `secondary(userB, token-3)`
- PAT 安全：仅服务端存储（root 600），绝不通过 API 输出，日志打码。

### 2.2 runs 表（加 account_id，替代简单 account 列）

```sql
ALTER TABLE runs ADD COLUMN account_id INTEGER REFERENCES github_accounts(id);
-- 现有行回填到默认 profile（或 NULL=历史未知）
```

标识完整性（四个身份层次）：

```
run_id      = GitHub Run 身份
account_id  = 该 Run 由哪个 GitHub Account/Credential Profile 触发
agent_id    = 实际 Runner/Agent
vm_id       = 实际 Azure VM
```

## 3. API 设计

| 端点 | 现状 | 计划 |
|---|---|---|
| `POST /admin/trigger` | 单账号 dispatch | 加可选 `account`（profile 名/ID；默认=配置 default，或调度器选择） |
| `GET /admin/github/workflows` | 单账号 | 加可选 `account`；缺省列默认/全部 |
| `POST /admin/cancel` | 单账号 | **按 run.account_id 自动选 PAT**，调用者零账号信息 |
| `GET /admin/accounts`（新） | — | 各 profile：owner/repo/enabled/in_progress/max_concurrent_runs/余量 |
| `POST /admin/exec` | 目标解析 | 加可选 `account` 提示（见 §4） |

**cancel 链路（调用者为空账号）**：
```
run_id → runs.account_id → github_accounts → PAT → GitHub API
```

## 4. 任务路由

- **组合式（先用）**：`trigger(account=secondary)` → 等 agent → `exec(run_id)`。
- **原子式（后做）**：
  ```
  POST /admin/exec { "account": "secondary", "command": ["docker","ps"] }
  ```
  → secondary profile → PAT → workflow → Run → Runner Agent → exec。
- `account` 缺省时：使用默认 profile / 调度器选择。
- 兼容现有 `/admin/exec` 分层：Run target（run_id/id/vm_id）与 Agent target（agent_id）之上再叠 account 提示层；边界干净。

## 5. 调度器

- 不把 20 写死进核心逻辑；判断规则仅：
  ```
  当前 in_progress < profile.max_concurrent_runs   → 可分发
  ```
- GitHub 限制变化只改配置，不改代码。
- 多账号视图：
  ```
               Oracle Scheduler
                   │
     ┌─────────────┼─────────────┐
     ▼             ▼             ▼
 Account A    Account B     Account C
  20 slots      20 slots     15 slots
     │             │             │
     ▼             ▼             ▼
  Runners       Runners       Runners
  ```

## 6. 实施顺序（明确分期）

- ✅ **Phase 1（已完成）**：Run→Agent 定位（run_id/id/vm_id/agent_id 解析 + 状态机 + resolved_by）。
- ⏳ **Phase 2（本计划，单独实施）**：
  1. `github_accounts` 表 + 凭据注册表加载（兼容单账号退化）
  2. runs 加 `account_id` + 迁移回填 + 档案回显
  3. trigger/workflows/cancel 按 account 路由（cancel 自动反查 PAT）
  4. `admin/accounts` 并发余量视图 + 调度器判断
  5. （可选）exec 的 `account` 提示 → 原子式路由

## 7. 非目标 / 约束

- 不改 agent 端协议；Agent 不知道 PAT/GitHub 账号/repo。
- Phase 1 与 Phase 2 边界：本计划独立落盘，不与当前 `/admin/exec` 修改混在一起实现。
- PAT 治理：多 profile 是配置/凭据层的事，与 Runner 生命周期解耦。

## 8. 关联现有代码

- 凭据现读取：`server/api/admin.py` `_gh_creds_or_raise()`（单账号；Phase 2 改造为按 profile 查）
- 生命周期：`server/db.py` runs 表 + `create_run_provisioning / sync_run_from_github`
- 执行解析：`server/api/admin.py` `_resolve_exec_target`（Phase 2 不侵入此层，只加 account 提示层）