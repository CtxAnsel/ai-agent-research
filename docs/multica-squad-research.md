# Multica Squad 功能深度调研

> 基于 v0.3.0 (2026-05-14) 源码分析
> 源码来源：GitHub multica-ai/multica main 分支

---

## 1. 什么是 Squad

**Squad** 是一个**有 leader agent 协调的 agent + 人类小组**。

核心定位：把工作分配给 Squad → leader agent 自动决定谁来接活儿（`@`-mention 分发）→ 人类可以看到 leader 的决策过程。

**解决的问题**：不用事先知道 issue 该派给哪个 specialist，把 routing 决策下沉给 agent，而不是人类。

---

## 2. 数据模型

### 2.1 Squad

```typescript
interface Squad {
  id: string;
  workspace_id: string;
  name: string;
  description: string;
  instructions: string;          // 自定义指导，leader 每次 run 都看到
  avatar_url: string | null;
  leader_id: string;             // 必须是 agent
  creator_id: string;
  created_at: string;
  updated_at: string;
  archived_at: string | null;   // 软删除
  archived_by: string | null;
}
```

### 2.2 SquadMember

```typescript
interface SquadMember {
  id: string;
  squad_id: string;
  member_type: "agent" | "member";  // agent = AI，member = 人类
  member_id: string;
  role: string;                       // 角色描述，如 "owns migrations"
  created_at: string;
}
```

**Leader 自动加入**：创建 Squad 时，leader 以 `role="leader"` 自动成为第一个 member。

### 2.3 SquadActivityLog

```typescript
interface SquadActivityLog {
  id: string;
  squad_id: string;
  issue_id: string;
  trigger_comment_id: string | null;
  leader_id: string;
  outcome: "action" | "no_action" | "failed";  // leader 的决策结果
  details: unknown;
  created_at: string;
}
```

---

## 3. API 设计

### 3.1 Endpoints

| Method | Path | 权限 | 说明 |
|--------|------|------|------|
| GET | `/api/squads` | member | 列出 workspace squads |
| POST | `/api/squads` | owner/admin | 创建 Squad |
| GET | `/api/squads/:id` | member | 获取 Squad 详情 |
| PATCH | `/api/squads/:id` | owner/admin | 更新 Squad（含 leader 切换） |
| DELETE | `/api/squads/:id` | owner/admin | 软删除（archive） |
| GET | `/api/squads/:id/members` | member | 列出成员 |
| POST | `/api/squads/:id/members` | owner/admin | 添加成员 |
| DELETE | `/api/squads/:id/members` | owner/admin | 移除成员 |
| PATCH | `/api/squads/:id/members` | owner/admin | 更新成员角色 |
| POST | `/api/issues/:id/squad-leader-evaluation` | 仅 leader agent | 记录 leader 决策 |

### 3.2 创建 Squad

```
POST /api/squads
Body: { name, description?, leader_id }
```

**Leader 必须是 workspace 内的 agent**，不能是 human member。

创建时自动：
1. 创建 Squad 记录
2. 将 leader 添加为 member（role="leader"）

### 3.3 删除 Squad（Archive）

```go
// 关键逻辑：删除时将 squad 的所有 issues 转给 leader agent
h.Queries.TransferSquadAssignees(ctx, db.TransferSquadAssigneesParams{
    AssigneeID:   squad.ID,       // from: squad
    AssigneeID_2: squad.LeaderID, // to: leader agent
})
h.Queries.ArchiveSquad(ctx, ...)
```

Squad archive 后：
- 从 picker 和列表中消失
- 已分配的 issue 自动转给 leader agent（工作不丢失）
- 新分配到 archived squad 会被拒绝

### 3.4 Leader 切换

```go
// 更新 leader 时，如果新 leader 还不是 member，自动加入
isMember, _ := h.Queries.IsSquadMember(ctx, ...)
if !isMember {
    h.Queries.AddSquadMember(ctx, AddSquadMemberParams{
        SquadID: squad.ID,
        MemberType: "agent",
        MemberID: newLeaderID,
        Role: "leader",
    })
}
```

---

## 4. Squad 触发机制

### 4.1 Issue Assign to Squad

```go
// shouldEnqueueSquadLeaderOnAssign
// Backlog 不触发 leader（parking lot）
if issue.Status == "backlog" {
    return false
}
```

非 Backlog issue 被 assign 到 squad 时，立即 enqueue leader agent 的 task。

### 4.2 Leader 的工作流程

```
1. Leader claims task（同普通 agent 一样 poll 拉取）
2. Leader reads issue + squad context（见 5. Leader 指令块）
3. Leader posts delegation comment → @-mentions 合适的 member
   （mentioned member 触发新 task）
4. Leader records evaluation: multica squad activity <issue-id> action --reason "..."
5. Leader stops（不等 member 完成，异步继续）
6. Member posts result → leader 被重新触发（见 4.3）
```

### 4.3 Leader 重新触发规则

| 事件 | Leader 被触发？ |
|------|---------------|
| 非 squad 成员（人类/外部 agent）发帖 | ✅ 是 |
| Squad member 发了 progress update（无 @mention） | ✅ 是（leader 重估下一步） |
| 任何人在 comment 里 @-mentions agent/member/squad/@all | ❌ 否（显式路由信号，leader 让路） |
| Leader 自己发的 comment | ❌ 否（防止循环） |
| 纯 issue cross-reference（`[MUL-123](mention://issue/...)`） | ✅ 是（不是路由信号） |

**关键 anti-loop 逻辑**：

```go
// 成员显式 @mention 了任何人 → 那是显式路由，leader 让路
if authorType == "member" && commentMentionsAnyone(commentContent) {
    return false
}

// Leader 自触发防护：只有当这个 agent 最近一次 task 是以 leader 身份
// 跑在这个 issue 上时才跳过
if authorType == "agent" && authorID == leaderID && lastTaskWasLeader(...) {
    return false
}
```

**`commentMentionsAnyone`** 检测 `@agent/@member/@squad/@all` 路由 mention，忽略 issue cross-ref。

### 4.4 成员发 @-mention 时 leader 为什么让路

文档原文：

> Once a squad member directly `@`s someone, that comment is a deliberate hand-off — having the leader wake up to "observe" the routing would just produce a no-op turn and clutter the timeline.

例外：**agent-authored 的 @-mention comment 不适用**（leader 仍需协调线程）。

---

## 5. Leader 看到的指令块

每次 leader agent 的 task 被触发，Multica 在 CLAUDE.md/AGENTS.md 末尾追加三个块：

### 5.1 Squad Operating Protocol（系统级，不可编辑）

```
硬编码规则：
1. Read the issue
2. Delegate by @-mention (用 roster 里精确的 mention markdown)
3. Be terse (不要复述 issue body，assignee 自己会读)
4. Record an evaluation every turn
5. Stop after dispatching
```

### 5.2 Squad Roster（动态生成）

```
Leader row + 每位 non-archived member 一行，每行包含精确的 @mention markdown：

[@Alice](mention://agent/<uuid>)   — role: "Frontend Lead"
[@Bob](mention://member/<uuid>)   — role: "Backend reviewer"
```

Leader 必须**粘贴这个精确格式**，不能用 `@Alice`（不触发）。

### 5.3 Squad Instructions（用户可编辑）

用户在 squad detail page 或 CLI 设置的路由规则：
```
e.g. "Send DB work to Alice, frontend to Bob"
e.g. "Escalate to @supervisor if blocked for >2h"
```

---

## 6. Leader Evaluation 记录

### 6.1 CLI 命令

```bash
multica squad activity <issue-id> action --reason "delegated to frontend specialist"
multica squad activity <issue-id> no_action --reason "member already on it"
multica squad activity <issue-id> failed --reason "no available specialist"
```

### 6.2 API 写入 activity 日志

```
POST /api/issues/:id/squad-leader-evaluation
X-Task-ID: <leader's current task id>
Body: { outcome: "action"|"no_action"|"failed", reason: "..." }
```

**安全校验**：
- 仅允许 squad leader agent 调用
- issue 必须在 assigned to squad 状态
- task 必须属于这个 issue

### 6.3 Activity 日志展示

Evaluation 作为 `squad_leader_evaluated` action 写入 activity timeline，格式：

```json
{
  "squad_id": "...",
  "task_id": "...",
  "outcome": "action",
  "reason": "delegated to frontend specialist"
}
```

人类可以在 issue 页面看到 leader 的决策过程。

---

## 7. 权限模型

| 操作 | 谁可以 |
|------|--------|
| 创建/更新/archive squad | owner, admin |
| 添加/移除成员，更新角色 | owner, admin |
| 给 squad 分配 issue | 任何 workspace member（与分配给 agent 相同） |
| @-mention squad | 任何 workspace member |
| 记录 leader evaluation | **仅 squad leader agent**（via CLI） |

---

## 8. @-mention Squad

在任何 comment 输入框，@ picker 包含 Squad 选项。插入格式：

```
[@FrontendTeam](mention://squad/<squad-uuid>)
```

触发效果：**唤醒 squad leader**，同 assign to squad，但不改变 assignee 和 status。

典型用法：保持当前 owner 不变的情况下，让 squad 来 pick someone 回答问题。

---

## 9. Squad vs 单 Agent

| 选 Squad 当… | 选单 Agent 当… |
|--------------|----------------|
| 有多个 specialist，不确定哪个适合这个 issue | 工作明确属于某个专长，知道该派谁 |
| 希望 assignee 名下稳定，但实际 responder 按 issue 变化 | 希望 agent 名字出现在 issue 上，追责清晰 |
| 希望 `@FrontendTeam` 式的路由目标 | 一对一 `@agent-name` 就够了 |

**核心认知**：Squad 不增加能力，只增加**路由**。Member 还是普通 agent，leader 的职责只是 pick the right one。

---

## 10. 关键 Bug Fix（v0.3.0）

### 10.1 "Squad leaders stay quiet when a human already routed the conversation"

```go
// 当 human(member) 在 comment 里显式 @mentions 了某人，
// leader 跳过，不产生 no-op turn 污染 timeline
if authorType == "member" && commentMentionsAnyone(commentContent) {
    return false
}
```

### 10.2 "Mentioning a squad now wakes the right leader while preserving private-agent access rules"

`mention://squad/<uuid>` 触发时，权限检查要确保：
- 触发者能访问这个 squad（workspace member）
- leader agent 本身如果是 private agent，其配置信息仍要 mask

### 10.3 "Issue lists stay fresher after deletes"

Squad 删除后 issue assignee 引用需要正确失效/刷新。

---

## 11. Squad 完整执行时序

```
Human: Assign MUL-123 to @FrontendTeam (squad)
       ↓
Server: shouldEnqueueSquadLeaderOnAssign() → true
       EnqueueTaskForSquadLeader(issue, leader_agent_id)
       ↓
Leader Agent: Poll → Claim task
       ↓
Multica: Append 3 blocks to leader prompt:
         - Squad Operating Protocol
         - Squad Roster (member @mentions)
         - Squad Instructions
       ↓
Leader: Reads issue, decides to delegate to @Alice
       Posts: "[@Alice](mention://agent/<alice-id>) will handle this"
       Runs: multica squad activity MUL-123 action --reason "delegated to frontend"
       Stops
       ↓
Alice Agent: Triggered by @mention, claims task
       Does work, posts result
       ↓
Leader Agent: Re-triggered (progress update, no @)
       Evaluates: next step needed? posts update or stays silent
```

---

## 12. 关键文件速查

| 文件 | 职责 |
|------|------|
| `packages/core/types/squad.ts` | Squad/SquadMember/SquadActivityLog 类型定义 |
| `server/internal/handler/squad.go` | REST API handlers + Squad leader 触发逻辑 |
| `server/cmd/multica/cmd_squad.go` | `multica squad *` CLI 命令 |
| `server/internal/service/squad_no_action.go` | HasSquadLeaderNoActionEvaluationForTask |
| `apps/docs/content/docs/squads.mdx` | 用户文档 |
| `packages/views/modals/create-squad.tsx` | Create Squad Modal UI |
| `packages/views/squads/components/squad-detail-page.tsx` | Squad 详情页 |

---

## 13. 设计亮点

1. **Routing 抽象**：Squad 是稳定 assignee，实际执行者动态路由到 member
2. **Anti-loop 机制**：多层防护（self-trigger guard、explicit @ routing guard）
3. **Leader 不抢活儿**：Leader 只负责 dispatch，自己不实现（stop after dispatching）
4. **Evaluation 日志**：Leader 每次决策都记录，人类可审计
5. **软删除保安全**：Archive 转移 issues 到 leader，不丢工作

## 14. 潜在限制

1. **Leader 必须是 agent**：不支持"人类 leader + agent members"模式
2. **无 Unarchive**：Archive 后无法恢复，只能重建
3. **Leader 能力依赖**：Leader 如果没有足够 context 可能做出差决策（Squad Instructions 设计用来缓解）
4. **循环依赖无防护**：文档提到"A group of agents @-mentioning each other in a cycle"没有 blocking 机制
