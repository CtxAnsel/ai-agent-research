# Multica Agent 参数化方案设计分析

> 作者：CtxAnsel  
> 日期：2026-05-09  
> 研究目标：分析 Multica 的智能体动态指令、Skill 系统、环境变量注入、自定义参数传递方案

---

## 摘要

Multica 的 agent 参数化系统由三个核心组件构成：**Prompt 构建层**（分层指令文件 + per-turn 动态 prompt）、**Skill 注入层**（provider 原生路径写入 SKILL.md）、**环境配置层**（硬编码 MULTICA_* 变量 + 自定义 CustomEnv 合并）。整体设计采用"配置与执行分离"——server 只负责下发配置元数据，daemon 负责把配置翻译成各 provider 的 CLI 参数和文件系统布局。

---

## 1. 动态 / 指令系统 — Prompt 构建架构

### 1.1 两层 Prompt 架构

Multica 的 prompt 系统不是单次发送静态指令，而是**持久化元技能文件 + per-turn 动态 prompt** 配合：

```
第一层：Meta Skill 文件（持久化）
  → 写入 workdir/CLAUDE.md（Claude）
  → 写入 workdir/AGENTS.md（Codex/Copilot/OpenCode/OpenClaw/Hermes/Pi/Cursor/Kimi/Kiro）
  → 写入 workdir/GEMINI.md（Gemini）

第二层：Per-turn Prompt（每次轮询构建）
  → BuildPrompt(task) → 任务类型分发
```

**元技能文件内容**（`runtime_config.go:buildMetaSkillContent`）：
- Agent 身份名称 + 核心指令
- 可用命令（multica CLI 参考）
- Repositories 配置
- 项目上下文
- 工作流步骤说明
- Skills 列表
- @ 指令说明
- 附件处理规则
- 输出格式要求

**Per-turn Prompt 构建**（`prompt.go`）：
```
BuildPrompt(task) → 按触发类型分发：
  - ChatSessionID     → buildChatPrompt        （聊天会话）
  - TriggerCommentID  → buildCommentPrompt     （评论触发）
  - AutopilotRunID    → buildAutopilotPrompt   （自动驾驶）
  - QuickCreatePrompt → buildQuickCreatePrompt（快速创建）
  - default           → generic "assigned issue" prompt（通用 issue）
```

### 1.2 评论触发任务的特殊处理

评论触发任务（`TriggerCommentID`）的 prompt 中**完整嵌入评论内容**，防止恢复会话时的过期上下文问题。每次轮询都重新发送 trigger comment ID，防止 `--parent` 值漂移。

### 1.3 指令注入链路

```
Server (DB) → ClaimTaskByRuntime → TaskContextForEnv.AgentInstructions
  → InjectRuntimeConfig → workdir/{AGENTS,CLAUDE,GEMINI}.md
  → OpenClaw 特殊：execOpts.SystemPrompt 直接内联（不走文件）
```

---

## 2. Skill 系统

### 2.1 Provider 原生 Skill 路径

Skill 文件写入各 provider 在 workdir 下的原生发现路径，而不是统一的中间层：

| Provider | Skill 路径 |
|----------|-----------|
| claude | `{workDir}/.claude/skills/{name}/SKILL.md` |
| codex | `{envRoot}/codex-home/skills/{name}/SKILL.md`（CODEX_HOME） |
| copilot | `{workDir}/.github/skills/{name}/SKILL.md` |
| opencode | `{workDir}/.config/opencode/skills/{name}/SKILL.md` |
| pi | `{workDir}/.pi/skills/{name}/SKILL.md` |
| cursor | `{workDir}/.cursor/skills/{name}/SKILL.md` |
| kimi | `{workDir}/.kimi/skills/{name}/SKILL.md` |
| kiro | `{workDir}/.kiro/skills/{name}/SKILL.md` |
| hermes/gemini（default） | `{workDir}/.agent_context/skills/{name}/SKILL.md` |

**设计思路**：直接复用各 provider 的原生 skill 发现机制，Multica daemon 只负责把 skill 内容写到正确的路径，无需理解各 provider 的 skill 加载逻辑。

### 2.2 Codex 特殊处理

- Skill 写入**per-task CODEX_HOME** 下的 `skills/` 目录
- `CODEX_HOME` 通过 `agentEnv` 注入，而非直接写在 context.go
- 从 `~/.codex/` seed per-task CODEX_HOME（symlink 共享认证/会话，copy 隔离配置）
- 复制 `config.toml` 时会 strip `[[skills.config]]` 块（CLI 0.114 TOML 反序列化 bug）

### 2.3 Local Skill 系统

独立于 task-bound skills 的**运行时级别 skill 发现**（`local_skills.go`）：
- `listRuntimeLocalSkills(provider)` — 遍历运行时级别的 skills 根目录
- `loadRuntimeLocalSkillBundle(provider, skillKey)` — 加载指定 skill 的内容 + 文件
- 通过 skill handler endpoints 提供服务，与 task-bound skills 完全独立

### 2.4 SKILL.md 格式

```
{skillDir}/SKILL.md          — 主文件（必需），包含 name/description frontmatter
{skillDir}/（辅助文件）        — 可选，与 SKILL.md 同目录，通过 SkillFileData.Path 引用
```

Frontmatter 解析：提取 `name:` 和 `description:` 字段。

---

## 3. 环境变量方案

### 3.1 全量环境变量构建

环境构建在 `daemon.go:1316-1365`，最终通过 `buildEnv()` 传递给所有 backend：

```go
// daemon.go:1316-1365
agentEnv := map[string]string{
    "MULTICA_TOKEN":           ...,
    "MULTICA_SERVER_URL":       ...,
    "MULTICA_DAEMON_PORT":      ...,
    "MULTICA_WORKSPACE_ID":     ...,
    "MULTICA_AGENT_NAME":       ...,
    "MULTICA_AGENT_ID":         ...,
    "MULTICA_TASK_ID":          ...,
    "MULTICA_TASK_SLOT":        ...,
    // 条件注入：
    "MULTICA_AUTOPILOT_RUN_ID": ...,  // autopilot 任务
    "MULTICA_AUTOPILOT_ID":     ...,  // autopilot 任务
    "MULTICA_QUICK_CREATE_TASK_ID": ..., // quick-create 任务
}
// PATH 前置 daemon 二进制目录
// CODEX_HOME 注入（仅 codex provider）
// CustomEnv 合并（blocklist 过滤后）
```

### 3.2 Blocklist 机制

```go
func isBlockedEnvKey(key string) bool {
    upper := strings.ToUpper(key)
    if strings.HasPrefix(upper, "MULTICA_") { return true }
    switch upper {
    case "HOME", "PATH", "USER", "SHELL", "TERM", "CODEX_HOME": return true
    }
    return false
}
```

**关键点**：`MULTICA_*` 前缀和核心系统变量（HOME/PATH/USER/SHELL/TERM/CODEX_HOME）被 blocklist 保护，custom_env 无法覆盖。

### 3.3 CLAUDECODE 过滤

```go
func isFilteredChildEnvKey(key string) bool {
    return key == "CLAUDECODE" ||
        strings.HasPrefix(key, "CLAUDECODE_") ||
        strings.HasPrefix(key, "CLAUDE_CODE_")
}
```

CLAUDECODE 相关变量从 base environment 中过滤，custom_env 中同名变量可以覆盖（override 行为）。

### 3.4 MULTICA_KEEP_ENV_AFTER_TASK

```go
keepEnv := os.Getenv("MULTICA_KEEP_ENV_AFTER_TASK") == "true" ||
            os.Getenv("MULTICA_KEEP_ENV_AFTER_TASK") == "1"
```

调试标志。开启后 daemon 的部分清理逻辑保留 workdir（只删除 workdir/，保留 output/ 和 logs/），方便调试 agent 执行产物。

---

## 4. 自定义参数传递

### 4.1 Server → Daemon 数据结构

**AgentData**（`types.go:64-73`）：
```go
type AgentData struct {
    ID           string            `json:"id"`
    Name         string            `json:"name"`
    Instructions string            `json:"instructions"`
    Skills       []SkillData       `json:"skills"`
    CustomEnv    map[string]string `json:"custom_env,omitempty"`
    CustomArgs   []string          `json:"custom_args,omitempty"`
    McpConfig    json.RawMessage   `json:"mcp_config,omitempty"`
    Model        string            `json:"model,omitempty"`
}
```

**ClaimTask 时填充**（`handler/daemon.go:842-869`）：
- Skills：通过 `h.TaskService.LoadAgentSkills()` 从 DB 加载
- CustomEnv/CustomArgs/McpConfig：反序列化 DB 字节
- Model：`agent.Model.String`

### 4.2 两级 Model 解析

```go
// daemon.go:1388-1429
model := ""
if task.Agent != nil && task.Agent.Model != "" {
    model = task.Agent.Model  // explicit agent.model 优先
}
if model == "" {
    model = entry.Model  // 降级到 daemon 条目级默认（环境变量 MULTICA_<PROVIDER>_MODEL）
}
// 最终传递给 agent.ExecOptions.Model
```

### 4.3 ExecOptions 全貌

```go
type ExecOptions struct {
    Cwd                       string
    Model                     string
    SystemPrompt              string
    MaxTurns                  int
    Timeout                   time.Duration
    SemanticInactivityTimeout time.Duration
    ResumeSessionID           string
    ExtraArgs                 []string   // daemon-wide 默认参数
    CustomArgs                []string   // per-agent 自定义参数
    McpConfig                 json.RawMessage
}
```

---

## 5. Provider 特定配置

### 5.1 Argument 构建差异

```
daemon.defaultArgsForProvider() 返回 provider 默认 args
  ↓
ExtraArgs（daemon-wide）先 append
CustomArgs（per-agent）后 append
  ↓
provider 对最终 args 做 filter（移除不允许的 flag）
```

### 5.2 各 Backend 对比

| Provider | CLI 入口 | Model 参数方式 | McpConfig | 特殊处理 |
|----------|---------|--------------|-----------|---------|
| claude | `claude-code --output-format stream-json ...` | `--model` flag | 写 tempfile → `--mcp-config` | CLAUDECODE env 过滤 |
| codex | `codex app-server --listen stdio://` | `config.toml` 写入（daemon managed block） | 不支持 | JSON-RPC 2.0 over stdio |
| copilot | `gh copilot` | `--model` flag | 不支持 | 阻塞 `--listen` |
| opencode | 待补充 | 不支持 | 待补充 | 阻塞 `--model`（model 在注册时绑定） |
| openclaw | 待补充 | 不支持 | 待补充 | SystemPrompt 内联，不走文件 |
| gemini | `gemini -m <model> ...` | `-m` flag | 不支持 | — |
| kiro | 待补充 | ACP `session/set_model` | 不支持 | — |
| hermes | 待补充 | — | — | — |
| cursor | 待补充 | 待补充 | 待补充 | — |

### 5.3 Codex Config 管理

Codex 的 model 配置不通过 CLI flag，而是通过**per-task config.toml 的 managed block**：

```
# BEGIN multica-managed
sandbox_mode = "workspace-write"
sandbox_workspace_write.network_access = true
features.multi_agent = false
# END multica-managed
```

Managed block 是幂等的，保留用户 block 外内容。`sanitizeCopiedCodexConfig()` 会在复制时 strip `[[skills.config]]` 块（CLI 0.114 bug）。

### 5.4 OpenClaw 的 SystemPrompt 内联

OpenClaw 从 bootstrap 文件读取 workspace context，不从 task workdir 读。所以 OpenClaw 的 `task.Agent.Instructions` 通过 `execOpts.SystemPrompt` 直接内联传给 backend，不走 `CLAUDE.md` 文件路径。

---

## 6. 完整执行链路

```
1. Server (DB)
   └── Task + AgentData (instructions, skills, custom_env, custom_args, model, mcp_config)

2. Daemon.ClaimTask
   └── TaskContextForEnv { AgentInstructions, AgentSkills, AgentCustomEnv, AgentCustomArgs, AgentMcpConfig, AgentModel }

3. ExecEnv.Prepare
   └── 创建目录结构 {workspacesRoot}/{workspaceID}/{taskID_short}/workdir/

4. writeContextFiles
   ├── .agent_context/issue_context.md              （通用上下文）
   ├── skills → provider-native paths                （Skill 注入）
   └── .multica/project/resources.json               （如有）

5. InjectRuntimeConfig
   └── CLAUDE.md / AGENTS.md / GEMINI.md             （Meta Skill 文件）

6. BuildPrompt(task)
   └── Per-turn 动态 prompt（任务类型分发）

7. daemon.runTask → buildAgentEnv
   ├── MULTICA_* 硬编码变量
   ├── PATH 前置 daemon 目录
   ├── CODEX_HOME（仅 codex）
   └── custom_env 合并（blocklist 过滤）

8. agent.Backend.Execute(prompt, execOpts)
   └── 启动 CLI 子进程（stdio 通信）
```

---

## 7. 设计亮点与局限性

### 7.1 亮点

| 设计 | 价值 |
|-----|------|
| Provider 原生 Skill 路径 | 无需为每个 provider 实现 skill loader，daemon 只负责写文件 |
| Meta Skill + Per-turn 分离 | 持久上下文只写一次，per-turn prompt 保持精简 |
| Managed Config Block（Codex） | 在用户配置中安全注入 daemon 管理的配置段 |
| Blocklist + 两级 Model | 保证关键变量不被篡改，同时提供足够的定制灵活性 |
| McpConfig 透传 | 支持 MCP（Model Context Protocol）配置扩展 |

### 7.2 局限性

| 问题 | 影响 |
|-----|------|
| CustomEnv 深度合并缺失 | 无法通过 custom_env "unset" 一个 base env 变量 |
| OpenClaw SystemPrompt 内联 | 与其他 provider 的文件路径方式不一致，增加耦合 |
| McpConfig 只支持 Claude | 其他 provider 的 MCP 扩展能力被堵死 |
| Agent.Model 非标准 provider | OpenClaw/Kiro 等使用非标准 model 绑定方式（注册时绑定 vs 运行时 flag） |
| 无 per-turn Skill 动态注入 | Skills 只在 Prepare 阶段写入，任务执行中无法动态更新 |

---

*文档版本：v1.0*  
*研究方法：源码分析（multica commit befd37）*
