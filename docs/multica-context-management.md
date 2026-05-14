# Multica 上下文管理方案 — 深度技术调研

## 1. 架构总览

```
Daemon (daemon.go)
│
├── handleTask()           — 接收任务，组装上下文
│
├── runTask()              — 构建 TaskContextForEnv
│   │
│   ├── LoadAgentSkills() — 从 DB 加载 agent 绑定的 skills
│   ├── convertSkillsForEnv()
│   ├── convertReposForEnv()
│   └── convertProjectResourcesForEnv()
│
└── execenv.Prepare() / execenv.Reuse()
    │
    ├── 目录结构创建
    │
    ├── writeContextFiles()
    │   ├── .agent_context/issue_context.md
    │   ├── .multica/project/resources.json
    │   └── skills → {provider-native path}
    │
    ├── InjectRuntimeConfig()
    │   └── CLAUDE.md / AGENTS.md / GEMINI.md
    │
    └── (codex only) prepareCodexHome() + CODEX_HOME/sandbox config
```

---

## 2. 目录结构

每个 task 创建独立的隔离目录：

```
{workspacesRoot}/{workspaceID}/{taskID_short}/
├── workdir/               — agent 的工作目录（git worktree 检出位置）
│   ├── CLAUDE.md          — Claude provider 写入
│   ├── AGENTS.md          — 其他 provider 写入
│   ├── .agent_context/     — 通用上下文
│   │   ├── issue_context.md
│   │   └── skills/        — fallback skills 路径（hermes/gemini）
│   ├── .claude/skills/    — Claude 原生 skills 发现路径
│   ├── .github/skills/    — Copilot 原生 skills 发现路径
│   ├── .cursor/skills/    — Cursor 原生 skills 发现路径
│   └── .multica/project/  — 项目资源
│       └── resources.json
├── output/                — 保留目录（部分清理时保留）
└── logs/                  — 保留目录（部分清理时保留）

{workspacesRoot}/{workspaceID}/{taskID_short}/
├── codex-home/           — 仅 Codex provider
│   ├── skills/           — Codex skills（直接写入）
│   ├── sessions/          — symlink 到 ~/.codex/sessions/
│   ├── auth.json          — copy
│   ├── config.json        — copy + sanitize
│   └── instructions.md    — copy
```

---

## 3. 文件写入顺序

### Prepare() 完整流程

```
1. 清理旧目录（如果存在）
2. 创建目录树：workdir/, output/, logs/
3. writeContextFiles(workDir, provider, task)
   │
   ├─ 3a. 创建 .agent_context/
   ├─ 3b. renderIssueContext() → 写入 .agent_context/issue_context.md
   │       （根据任务类型选择渲染器）
   ├─ 3c. resolveSkillsDir() 确定 skills 写入路径
   ├─ 3d. writeSkillFiles() 写入 skills 到 provider 原生路径
   │       （codex 跳过，codex 在 Prepare 里单独处理）
   └─ 3e. writeProjectResources() → .multica/project/resources.json
4. InjectRuntimeConfig(workDir, provider, task)
   └─ buildMetaSkillContent() → 写入 CLAUDE.md / AGENTS.md / GEMINI.md
5. (仅 codex)
   ├─ prepareCodexHomeWithOpts()
   └─ writeCodexWorkspaceSkills(codex-home/skills/)
6. 返回 Environment{RootDir, WorkDir, CodexHome}
```

### Reuse()（任务恢复）流程

```
1. 检查 workdir 是否存在
2. writeContextFiles() — 刷新 issue_context.md 和 skills
3. (仅 codex) 重新运行 prepareCodexHomeWithOpts() + skills
4. 返回 Environment
```

---

## 4. 任务类型与渲染器

`writeContextFiles()` 通过 `renderIssueContext()` 根据任务类型选择不同渲染器：

| 任务类型 | 触发条件 | 渲染器 | 特点 |
|----------|----------|--------|------|
| **Issue 任务** | `TriggerCommentID == ""` 且非 autopilot/quickcreate | `renderIssueContext()` | 标准工作流：issue get → comment list → in_progress → work → comment → in_review |
| **Comment 触发** | `TriggerCommentID != ""` | `renderIssueContext()` | 强调回复特定 comment，强调"不要 @mention 回复你的 agent"，禁止循环触发 |
| **Quick Create** | `QuickCreatePrompt != ""` | `renderQuickCreateContext()` | 无 issue，只能运行一次 `multica issue create`，禁止 issue get/status/comment |
| **Autopilot** | `AutopilotRunID != ""` | `renderAutopilotContext()` | 无 issue，可以运行 `autopilot get`，禁止 issue get/status/comment（除非指令明确） |

---

## 5. Provider 差异

### 5.1 Skills 写入路径

| Provider | Skills 写入路径 | 发现机制 |
|----------|----------------|---------|
| claude | `.claude/skills/{name}/SKILL.md` | Claude Code 原生 |
| codex | `codex-home/skills/{name}/SKILL.md` | Codex CLI via CODEX_HOME |
| copilot | `.github/skills/{name}/SKILL.md` | GitHub Copilot 原生 |
| opencode | `.config/opencode/skills/{name}/SKILL.md` | OpenCode 原生 |
| pi | `.pi/skills/{name}/SKILL.md` | Pi 原生 |
| cursor | `.cursor/skills/{name}/SKILL.md` | Cursor 原生 |
| kimi | `.kimi/skills/{name}/SKILL.md` | Kimi CLI 原生 |
| kiro | `.kiro/skills/{name}/SKILL.md` | Kiro CLI 原生 |
| hermes/gemini | `.agent_context/skills/{name}/SKILL.md` | 无原生发现，fallback |
| **default** | `.agent_context/skills/{name}/SKILL.md` | 无原生发现，fallback |

### 5.2 元配置文件

| Provider | 文件名 | 内容 |
|----------|--------|------|
| claude | `CLAUDE.md` | buildMetaSkillContent |
| codex/copilot/opencode/openclaw/hermes/pi/cursor/kimi/kiro | `AGENTS.md` | buildMetaSkillContent |
| gemini | `GEMINI.md` | buildMetaSkillContent |
| unknown | 无 | prompt-only 模式 |

### 5.3 Codex 特殊处理

Codex 有额外的 per-task CODEX_HOME 隔离：

```
prepareCodexHomeWithOpts(codexHome, CodexHomeOptions{CodexVersion})
  │
  ├─ symlink sessions/, auth.json（共享状态）
  ├─ copy config.json, config.toml, instructions.md（隔离修改）
  ├─ sanitizeCopiedCodexConfig() — 剥离 [[skills.config]]（TOML 解析兼容）
  └─ 写入 sandbox 策略（根据 GOOS 和 CodexVersion 选择配置）
```

**skills.config 剥离原因**：Codex Desktop 写入 `name = "superpowers:brainstorm"` 但无 `path` 字段，Codex CLI 0.114 TOML 反序列化将 `path` 视为必填，报 `missing field path` 拒绝启动。Multica 直接写 skills 到 `codex-home/skills/`，用户级注册表对 per-task 运行无关。

---

## 6. buildMetaSkillContent 完整结构

`CLAUDE.md` / `AGENTS.md` 包含以下章节（按顺序）：

```
# Multica Agent Runtime

## Agent Identity
（AgentInstructions 注入，agent 人设/人格指令）

## Available Commands
### Read
（multica issue get/list/comment list/label list/subscriber list/
 workspace get/members/agent list/repo checkout/issue runs/
 attachment download/autopilot list/get/runs）

### Write
（multica issue create/update/status/assign/label add-remove/
 subscriber add-remove/comment add-delete/label create/
 autopilot create/update/trigger/delete）

## Repositories
（workspace 关联的 repo URL 列表，提示用 multica repo checkout 检出）

## Project Context
（issue 所属 project 的标题 + 资源列表 +
 .multica/project/resources.json 结构）

### Workflow
（根据任务类型不同，渲染不同的工作流指令）
- chat: 交互式助手模式
- quick-create: 只能运行一次 issue create
- autopilot run: 无 issue，遵守 autopilot 指令
- comment-triggered: 回复特定 comment，避免循环 @mention
- assignment: 标准流程：get → list → in_progress → work → comment → in_review

## Skills
（列出 agent 绑定的所有 skills 名称）
- Claude/Codex/Copilot 等：说明"自动发现"
- hermes/gemini：说明从 .agent_context/skills/ 读取

## Mentions
（mention:// 链接的语义：issue=无害，member=通知人，agent=触发任务）
（何时不能用：回复 agent 时不要 @mention；何时能用：升级给人、委托子任务、用户明确要求）

## Attachments
（multica attachment download 用法）

## Important: Always Use the multica CLI
（禁止直接 curl/wget，必须用 CLI）

## Output
（最终结果必须通过 comment 交付，terminal 输出用户看不到）
```

---

## 7. Env 变量注入

Daemon 在启动 agent 子进程时注入：

```go
envVars := map[string]string{
    "MULTICA_WORKSPACE_ID": task.WorkspaceID,
    "MULTICA_AGENT_NAME":   agentName,
    "MULTICA_AGENT_ID":     task.AgentID,
    "MULTICA_TASK_ID":      task.ID,
    "MULTICA_TASK_SLOT":    strconv.Itoa(slot),
    // + 全部 task.Agent.CustomEnv
}
```

**blocked keys**（防止 override）：
```
MULTICA_WORKSPACE_ID, MULTICA_AGENT_NAME, MULTICA_AGENT_ID,
MULTICA_TASK_ID, MULTICA_TASK_SLOT,
PATH, HOME, USER, SHELL,
MULTICA_DAEMON_PORT, MULTICA_DAEMON_TOKEN
```

---

## 8. Repo 上下文管理

### 两种来源

1. **Workspace 级别 repos** — `multica repo add` 添加的 workspace 全域 repo
2. **Project 级别 repos** — issue 所属 project 绑定的 `github_repo` 资源

### 隔离机制

- TaskContextForEnv 只会拿到 **当前 task 需要的 repos**
- project 级别 repos 会 **覆盖** workspace 级别（如果同一个 URL）
- agent 必须通过 `multica repo checkout <url>` **按需检出**
- 检出会创建 **git worktree**，每个 task 独立分支

### repo checkout 流程

```
multica repo checkout <url>
  → daemon 接收请求
  → 检查 URL 是否在 allowedRepoURLs（workspace 或 task 级别）
  → 从 ~/.multica/repo-cache/{hash} bare repo 获取
  → 创建 worktree 到 {workdir}/{repo_name}/
  → 分支名格式：multica-{short-task-id}-{short-random}
```

---

## 9. Skill 热更新机制

| 时机 | 是否热更新 | 机制 |
|------|----------|------|
| 任务首次启动 | ✅ | Prepare() 写入 |
| 任务恢复（resume） | ✅ | Reuse() 重新调用 writeContextFiles |
| 任务执行中 | ❌ | agent 已加载 skills 到内存 |

**关键**：Reuse() 重新调用 `writeContextFiles()` → 重新写入 skills 和 issue_context.md，所以 skill 更新会在 resume 时生效。

---

## 10. 潜在风险与优化点

### 风险

1. **全量写入，无 filtering**：每个 task 启动时把 agent 绑定的 **全部 skills** 写入，即使任务不需要。如果 agent 有 20+ 个大 skills，prepare 耗时长 + prompt 膨胀。
2. **CLAUDE.md 完全覆盖项目原生文件**：Multica 生成的 CLAUDE.md 覆盖 `{workDir}/CLAUDE.md`，但如果项目 repo 被 checkout 到 `{workDir}/{repo}/`，其原生的 `CLAUDE.md` Claude Code 不会递归读取，所以不会冲突。
3. **Codex skills.config 兼容问题**：剥离是 workaround，不是原生方案，新版 Codex CLI 可能改变行为。
4. **无 context 大小限制**：没有硬性 token 限制或截断机制，完全依赖模型自动忽略无关内容。

### 优化方向

1. **按需写入 skills**：根据 issue 内容/标签/关键词过滤，只写入相关的 skills。
2. **skills 内容压缩**：大 skills 可以考虑截断或摘要模式。
3. **CLAUDE.md 合并项目原生**：如果项目根有 CLAUDE.md，可以考虑合并而不是覆盖。
4. **context 写入异步化**：prepare 不阻塞 agent 启动，写入可以在后台增量完成。

---

## 11. GC（垃圾回收）

`WriteGCMeta()` 在 task 完成后写入 `{root}/.gc_meta.json`：

```go
type GCMeta struct {
    IssueID     string
    WorkspaceID string
    CompletedAt time.Time
}
```

GC loop 根据 `.gc_meta.json` 判断 task 是否完成，清理过期的 workdir 和环境目录。

---

## 12. 关键文件速查

| 文件 | 职责 |
|------|------|
| `execenv.go` | Prepare/Reuse 入口，Environment 结构，目录结构 |
| `context.go` | writeContextFiles, issue_context 渲染，skills 写入 |
| `runtime_config.go` | InjectRuntimeConfig, buildMetaSkillContent |
| `codex_home.go` | prepareCodexHomeWithOpts, Codex 隔离配置 |
| `codex_skill_strip.go` | 剥离 skills.config 兼容性问题 |
| `daemon.go` (runTask) | 组装 TaskContextForEnv，env 变量注入 |
| `repocache/cache.go` | bare repo cache, worktree 创建/删除 |
| `service/task.go` | LoadAgentSkills, DB 查询 |
