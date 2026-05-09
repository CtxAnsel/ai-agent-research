# Multica Agent 执行环境方案技术分析

> 作者：CtxAnsel  
> 日期：2026-05-09  
> 研究目标：分析 Multica 的 agent 环境隔离、生命周期管理、通信机制

---

## 摘要

Multica 的 agent 执行方案采用**进程级隔离 + 目录级资源划分**的轻量级沙箱模型，**不依赖容器或 cgroup**。每个 agent 以标准 OS 子进程运行，通过 Go 的 `os/exec` 启动，与 daemon 共用同一用户命名空间。网络控制依赖各 provider 原生能力（主要是 Codex 的 sandbox_mode），无 OS 级网络命名空间隔离。整体设计偏向"够用就好"，以简单可靠为首要原则。

---

## 1. Agent 生命周期管理

### 1.1 Daemon 核心架构

`Daemon` 是所有 agent 的父进程，负责：

```
启动流程：
1. resolveAuth          → 从 CLI 配置解析认证 token
2. syncWorkspacesFromAPI → 从服务器同步 workspace 列表
3. registerRuntimesForWorkspace → 为每个 workspace 注册 runtime
4. 启动后台 goroutine：
   - workspaceSyncLoop   → 每 30s 轮询新 workspace
   - taskWakeupLoop      → 响应 WebSocket 推送（任务到达）
   - heartbeatLoop       → 每 15s HTTP 心跳（WS 活跃时抑制）
   - gcLoop              → 垃圾回收过期执行环境
   - serveHealth         → HTTP 健康检查端点（端口 19514）
5. 进入 pollLoop         → 通过 client.ClaimTask 争抢任务
```

**并发控制**：信号量 `MaxConcurrentTasks`（默认 20），每个任务执行前 acquire，完成后 release。

**任务超时**：`AgentTimeout`（默认 2 小时），通过 `context.WithTimeout` 强制取消。

### 1.2 任务调度流程

```
Server (WebSocket push 或 HTTP polling)
  ↓
Daemon.handleTask() → runTask()
  ↓
1. StartTask() 通知服务器任务开始
2. 创建可取消 context
3. 每 5s 轮询检查 server 是否标记为 cancelled
4. executeAndDrain() 执行 agent backend 并收集输出
5. 完成后报告结果给 server
```

**工作目录复用**：`PriorWorkDir` 机制——相同 agent + issue 组合会复用已有 workdir，刷新 context 文件而非创建全新环境。

---

## 2. 环境隔离方案

### 2.1 进程级隔离（无容器）

Agent 以 **Go `exec.CommandContext` 启动的子进程**形式运行，每个 provider 是独立的 Go struct：

```go
cmd := exec.CommandContext(runCtx, execPath, args...)
cmd.Dir = opts.Cwd          // 设置工作目录
cmd.Env = buildEnv(b.cfg.Env) // 注入环境变量
cmd.StdoutPipe()
cmd.StdinPipe()
cmd.Start()
```

**支持的 agent providers**：`claude`, `codex`, `copilot`, `opencode`, `openclaw`, `hermes`, `gemini`, `pi`, `cursor`, `kimi`, `kiro`

**无以下隔离机制**：
- ❌ 用户命名空间（user namespace）
- ❌ cgroup 资源限制
- ❌ 容器运行时（Docker/containerd）
- ❌ mount namespace 文件系统隔离
- ❌ seccomp/AppArmor/SELinux

### 2.2 目录级资源划分

每个任务在 `~/multica_workspaces/<workspace_id>/<task_id_short>/` 下创建独立目录树：

```
<root>/
  workdir/    → agent 的 CWD
  output/     → 任务完成后保留
  logs/       → 任务日志保留
```

**Codex 特殊处理**：创建 per-task `CODEX_HOME`，通过 symlink 共享认证/会话，config.toml 副本隔离。

### 2.3 Codex 的沙箱方案

这是唯一有实质性沙箱配置的 provider（`codex_sandbox.go`）：

| 操作系统 | sandbox_mode | network_access | 说明 |
|---------|-------------|----------------|------|
| Linux | `workspace-write` | `true` | Landlock 兼容，文件系统沙箱完整 |
| macOS | `danger-full-access` | — | Seatbelt bug 临时回退（openai/codex#10390） |

Daemon 通过管理 `config.toml` 中的 sandbox block（`# BEGIN/END multica-managed` 标记）实现配置注入，且是幂等的——保留用户区域内容。

---

## 3. 环境变量注入

### 3.1 Daemon 强制注入的变量

| 变量名 | 用途 |
|--------|------|
| `MULTICA_TOKEN` | 认证 token（blocklisted，不允许 custom_env 覆盖） |
| `MULTICA_SERVER_URL` | 服务器地址 |
| `MULTICA_DAEMON_PORT` | 本地健康检查端口 |
| `MULTICA_WORKSPACE_ID` | 所属 workspace |
| `MULTICA_AGENT_NAME` | Agent 标识名 |
| `MULTICA_AGENT_ID` | Agent UUID |
| `MULTICA_TASK_ID` | 任务 UUID |
| `MULTICA_TASK_SLOT` | 并发槽位索引 |
| `MULTICA_AUTOPILOT_RUN_ID` | Autopilot 任务 ID（如适用） |
| `MULTICA_AUTOPILOT_ID` | Autopilot ID（如适用） |
| `MULTICA_QUICK_CREATE_TASK_ID` | Quick create 任务 ID（如适用） |
| `PATH` | 预置 daemon 二进制目录，确保 `multica` CLI 全局可达 |
| `CODEX_HOME` | Codex provider 专用 |
| `MULTICA_KEEP_ENV_AFTER_TASK` | 调试标志 |

### 3.2 自定义环境变量

`task.Agent.CustomEnv` 合并进环境变量，但 **`isBlockedEnvKey` blocklist** 阻止覆盖关键变量。

---

## 4. 通信机制

### 4.1 Daemon ↔ Server：三层通道

```
1. HTTP 心跳（主通道，15s 轮询）
   Daemon → POST /api/daemon/heartbeat → Server
   Server 响应 pending actions（更新、模型列表请求、本地 skill 导入）

2. WebSocket（实时通道）
   Daemon ↔ ws://<server>/ws
   事件类型：
   - task_assigned        → 新任务到达
   - daemon:heartbeat_ack → WS 心跳确认（抑制 HTTP 心跳）
   - 服务器主动取消

3. CLI 子进程 IPC
   - Codex: JSON-RPC 2.0 over stdio（codex app-server --listen stdio://）
   - Claude/Copilot: 流式 JSON event over stdout
```

### 4.2 Agent Prompt 流转

```
Server → Daemon (HTTP/WS)
  ↓ BuildPrompt(task)
Daemon → AgentBackend.Execute(prompt)
  ↓
CLI 子进程（Claude Code / Codex CLI 等）
```

### 4.3 Agent → Daemon 的本地通信

Agent 子进程通过 `MULTICA_DAEMON_PORT`（默认 19514）访问 daemon 的 HTTP 健康端点。例如 `multica repo checkout` 需要访问 daemon 的本地 repo 缓存。

---

## 5. 资源限制与垃圾回收

### 5.1 资源限制（无 cgroup）

| 限制类型 | 实现方式 |
|---------|---------|
| 并发数 | Go 信号量（MaxConcurrentTasks=20） |
| 任务超时 | `context.WithTimeout`（AgentTimeout=2h） |
| Codex 静默超时 | 10min 无有效输出则终止（CodexSemanticInactivityTimeout） |
| CPU | ❌ 无 |
| 内存 | ❌ 无 |
| 磁盘配额 | ❌ 无 |

**安全风险**：恶意 agent 可通过 fork 炸弹耗尽 CPU/内存，直至 2 小时超时。

### 5.2 垃圾回收策略（gc.go）

| TTL 类型 | 默认值 | 清理对象 |
|---------|--------|---------|
| GCTTL | 24h | 已完成/取消的 issue 目录 |
| GCOrphanTTL | 72h | 无 GC 元数据的孤儿目录（崩溃遗留） |
| GCArtifactTTL | 12h | 仍在 open 的 issue 中的可重建产物（node_modules, .next, .turbo） |

GC 通过引用计数保护活跃环境，防止正在执行的任务被回收。

---

## 6. 安全模型分析

### 6.1 已有安全机制

| 机制 | 实现位置 |
|-----|---------|
| 关键环境变量 blocklist | `isBlockedEnvKey()` |
| Repo URL 白名单 | `workspaceRepoAllowed` check |
| Codex 沙箱配置管理 | `codex_sandbox.go`，macOS 有 fallback 日志警告 |
| 多 agent 默认关闭 | Codex `features.multi_agent` 通过 managed block 禁用 |
| GC 元数据追踪 | `.gc_meta.json` 防止跨 workspace 数据泄露 |
| 无凭据共享 | 每个任务独立 workdir / CODEX_HOME |

### 6.2 未覆盖的攻击面

```
⚠️ 无 OS 级进程沙箱
  - 无 seccomp profile
  - 无 AppArmor/SELinux profile
  - 恶意 agent 可尝试系统调用逃逸

⚠️ 无网络命名空间隔离
  - 非 Codex agent 可访问任意网络
  - Codex 在 macOS 上降级为 danger-full-access

⚠️ 无磁盘配额
  - agent 可耗尽磁盘

⚠️ 目录隔离依赖路径约定
  - 无 mount namespace
  - 路径遍历可能访问 host 文件

⚠️ 资源耗尽风险
  - 无 cgroup 内存/CPU 限制
  - fork 炸弹可在超时前瘫痪系统
```

---

## 7. Context 文件注入

`execenv/context.go` 负责在任务执行前向 workdir 写入上下文文件：

### 7.1 通用上下文

- `.agent_context/issue_context.md` — issue 元数据、可用命令、工作流说明

### 7.2 Provider 原生 Skills 目录

| Provider | Skills 路径 |
|----------|------------|
| Claude | `.claude/skills/{name}/SKILL.md` |
| Codex | per-task `CODEX_HOME/skills/` |
| Copilot | `.github/skills/{name}/SKILL.md` |
| OpenCode | `.config/opencode/skills/{name}/SKILL.md` |
| Cursor | `.cursor/skills/{name}/SKILL.md` |
| Kimi/Kiro | 类似 provider 原生路径 |

### 7.3 项目资源

- `.multica/project/resources.json` — 项目级资源（如有）

---

## 8. 关键文件索引

| 文件 | 职责 |
|-----|------|
| `server/internal/daemon/daemon.go` | Daemon 主结构，Run()，pollLoop，handleTask，runTask |
| `server/internal/daemon/execenv/execenv.go` | 环境准备，Prepare()，Reuse()，Cleanup() |
| `server/internal/daemon/execenv/context.go` | Context 文件注入（issue_context.md, skills） |
| `server/internal/daemon/execenv/codex_sandbox.go` | Codex 沙箱策略，macOS fallback |
| `server/internal/daemon/execenv/codex_multi_agent.go` | Codex 多 agent 禁用逻辑 |
| `server/internal/daemon/execenv/codex_home.go` | Per-task CODEX_HOME 设置（symlink/copy） |
| `server/internal/daemon/config.go` | 配置加载，环境变量，默认值 |
| `server/internal/daemon/gc.go` | 执行环境垃圾回收 |
| `server/internal/daemon/client.go` | HTTP/WS 客户端（与 server 通信） |
| `server/pkg/agent/agent.go` | Backend 接口定义 |
| `server/pkg/agent/codex.go` | Codex JSON-RPC over stdio backend |
| `server/pkg/agent/claude.go` | Claude stream-json backend |
| `server/pkg/agent/cursor.go` | Cursor backend |

---

## 9. 与行业方案的对比

### 9.1 vs Docker-based 方案（Cursor Team Kit, Copilot Workspace）

| 维度 | Multica | Docker-based 方案 |
|-----|---------|-------------------|
| 隔离级别 | 进程级 | 容器级（完整命名空间） |
| 启动延迟 | <1s | 3-10s（镜像拉取 + 容器启动） |
| 资源开销 | 极低（共用 host） | 中等（每个任务独立容器） |
| 网络隔离 | 依赖 provider 原生 | 容器网络命名空间 |
| 文件系统 | 目录级（可遍历） | 完整 overlayfs |
| 适用场景 | 轻量任务，CI/CD | 高安全要求，多租户 |

### 9.2 vs Codex Native 方案

Multica 的 Codex 沙箱配置是对 Codex 原生能力的增强管理——Codex 自身提供了 `workspace-write` 沙箱，Multica 负责在 per-task config 中正确注入和管理这个配置。这是**配置管理**而非**自建沙箱**。

---

## 10. 结论与建议

### 10.1 Multica 方案评价

**优点**：
- 架构极简，调试和维护成本低
- 启动延迟极低，适合高频短任务
- 通过 Codex 原生沙箱获得实用的安全边界
- GC 策略合理，磁盘管理自动化

**缺点**：
- 无 OS 级安全隔离，安全完全依赖信任模型
- 无资源上限，恶意/失控 agent 影响整个 host
- 网络隔离完全依赖各 provider 实现，不一致

### 10.2 适用场景

Multica 的轻量沙箱方案**适合**：
- 个人开发者/小团队场景（agent operator 可信）
- 高频短任务，对延迟敏感
- Codex 作为主要 provider（有原生沙箱保护）

**不适合**：
- 多租户/不受信任 agent 场景
- 需要严格网络隔离的环境
- 对资源耗尽风险零容忍的生产环境

### 10.3 潜在改进方向

1. **cgroup 资源限制**：至少为每个 agent 设置内存上限，防止 OOM
2. **网络命名空间**：对非 Codex provider 启用 network namespace
3. **disk quota**：防止磁盘耗尽攻击
4. **seccomp profile**：收紧系统调用，减少攻击面
5. **审计日志**：记录 agent 的文件系统访问和网络行为

---

*文档版本：v1.0*  
*研究方法：源码分析（multica commit befd37）*
