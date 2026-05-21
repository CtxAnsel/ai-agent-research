# Multica 技术调研文档

> 作者：CtxAnsel  
> 更新时间：2026-05-22  
> 标签：AI Agent, Multi-Agent Collaboration, Multica

---

## 目录

1. [概述](#1-概述)
2. [系统架构](#2-系统架构)
3. [路径体系](#3-路径体系)
4. [Agent 管理](#4-agent-管理)
5. [实时通信机制](#5-实时通信机制)
6. [Worktree 管理](#6-worktree-管理)
7. [桌面端 CLI 自动安装](#7-桌面端-cli-自动安装)
8. [竞品分析](#8-竞品分析)
9. [关键文件索引](#9-关键文件索引)

---

## 1. 概述

**Multica** 是一个 AI Agent 协作平台，支持多个 AI Agent 在同一个 Workspace 中协同工作。用户可以通过 Web Dashboard 或 Desktop Client 连接本地 Runtime，实现代码编写、Code Review、任务自动化等场景。

核心定位：**程序员的 AI 团队助手** — 多个 Agent 分工明确（Planner、Coder、Reviewer），像真实团队一样协作。

---

## 2. 系统架构

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│                        Frontend                              │
│   Web Dashboard (React Query) ←──── HTTP polling (5s)        │
│   Desktop Client (Electron)  ←──── WebSocket                 │
└─────────────────────┬────────────────────────────────────────┘
                      │ WebSocket + HTTP
┌─────────────────────▼────────────────────────────────────────┐
│                     Go Backend (server/)                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ REST API    │  │ WebSocket   │  │ EventBus            │  │
│  │ (Chi router)│  │ Hub         │  │ (in-memory)         │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────┬────────────────────────────────────────┘
                      │ WebSocket / HTTP
┌─────────────────────▼────────────────────────────────────────┐
│                   Runtime (Go daemon)                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Agent       │  │ Task        │  │ Workspace           │  │
│  │ Executor    │  │ Scheduler   │  │ Sync               │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────┬────────────────────────────────────────┘
                      │ Local Process
          ┌───────────┼───────────┐
          ▼           ▼           ▼
      Claude      OpenCode      Hermes
       Code        (本地)       (本地)
```

### 2.2 Daemon ↔ Server 通信

| 方向 | 机制 | 详情 |
|------|------|------|
| Daemon → Server | HTTP POST | `POST /api/daemon/register` 上报本地 Runtime |
| Daemon → Server | HTTP POST | `POST /api/daemon/deregister` 关机时注销 |
| Daemon → Server | HTTP 轮询 | 每 10 分钟轮询任务 (`poll: no tasks`) |
| Runtime → Server | **WebSocket** | 任务执行事件上报 (`task wakeup websocket connected`) |
| Web → Server | HTTP 轮询 | `GET /api/logs` 每 5 秒，无 SSE/WebSocket |

**关键洞察**：Daemon 本身是 HTTP 轮询，Runtime（具体 Agent 执行进程）才是 WebSocket 主动推送。

### 2.3 WebSocket 事件类型

位置：`server/pkg/protocol/events.go`

| 事件 | 触发时机 | 前端行为 |
|------|----------|----------|
| `task:queued` | 任务入队 | `setQueryData` — 写入缓存 |
| `task:dispatch` | Daemon 认领任务 | `setQueryData` — 更新状态为 running |
| `task:message` | 每条执行消息 | `setQueryData` — 追加到 `["task-messages", task_id]` |
| `task:completed` | 任务完成 | `invalidateQueries` — React Query 重新拉取 DB |
| `task:failed` | 任务失败 | `invalidateQueries` — React Query 重新拉取 DB |

**设计逻辑**：
- `task:message` 高频事件直接写缓存，避免网络风暴
- `task:completed/failed` 触发 DB 重新拉取，保证 UI 最终一致

---

## 3. 路径体系

### 3.1 环境与目录

| 变量/路径 | 值 |
|-----------|-----|
| `HOME` | `/opt/data/home` |
| Daemon 配置 | `~/.multica/` |
| Workspaces | `~/multica_workspaces/` |
| Hermes Skills | `~/.hermes/skills/` |
| Env 变量源 | `/opt/data/.env` |

### 3.2 Daemon 配置目录结构

```
~/.multica/  (= /opt/data/home/.multica/)
├── config.json      # 服务器 URL、workspace_id、token
├── daemon.log       # 运行时日志（持续写入，约 2MB）
├── daemon.id        # daemon 实例标识
└── daemon.pid       # 进程 ID
```

### 3.3 Workspace 工作目录

```
~/multica_workspaces/
├── 13ce6d74-ffa5-43b9-bba7-24a1379f4c43/   # 主 workspace (chentianxi1212)
│   └── [issue_id]/                          # 每个 issue 一个目录（worktree）
├── f6d86d70-1397-43f5-b931-5d5f0640e4b2/   # 另一个 workspace (weavefox-studio)
└── .repos/                                   # Git 仓库克隆目录
```

### 3.4 Agent Profile 存储

**Agent 配置不存本地，存在云端**：
- 每个 Agent 有 `runtime_id` 指向运行时实例
- Agent 定义（名称、描述、skills、instructions）从 API 获取
- 本地只记录 Agent 的 `runtime_id` 和状态

---

## 4. Agent 管理

### 4.1 内置 Agent 适配器

位置：`server/pkg/agent/`

| 文件 | Agent | 协议 |
|------|-------|------|
| `claude.go` | Claude (Anthropic) | ACP |
| `openclaw.go` | OpenClaw | ACP |
| `opencode.go` | OpenCode | ACP |
| `hermes.go` | Hermes | ACP |
| `cursor.go` | Cursor | ACP |
| `kimi.go` | Kimi | ACP |

每个 Agent 通过 **ACP 协议 (Agent Communication Protocol)** 动态发现模型，标准化接口统一管理。

### 4.2 Agent 分配机制

**核心结论**：Agent 分配是**显式绑定**的，Daemon 不主动拉取 Agent List。

| 触发方式 | 机制 |
|----------|------|
| Issue 指定 assignee | `POST /api/issues` → server 校验 assignee 是否为有效 agent |
| Autopilot | `autopilot.go` 验证 `AssigneeID` 是否为 workspace 有效 agent |
| @mention in chat | `trigger.go` — 抑制重复触发 |
| Agent 自查 | Agent 自己调 `multica agent list --output json` |

### 4.3 Agent 可见性

| 可见性 | 说明 |
|--------|------|
| `workspace` | 任何 workspace 成员可指派，协作模式 |
| `private` (默认) | 仅 owner/admin 可指派，保护敏感配置 |

**private 设计原因**：Agent 绑定本地 Runtime，private 可防止误指派给未上线的个人 Agent。

### 4.4 活跃 Workspace Agent 列表

```
planner       — 规划者，任何时候先 plan 再执行
Coder         — 资深编码专家
reviewer      — 测试工程师，验证 coder 输出的代码
studio-team   — Agent 中枢，调用 @planner/@Coder/@reviewer 协作
生活管家       — 生活小管家（私有 agent）
情绪价值官     — 提供情绪价值（workspace 可见）
测试          — 测试用（私有）
```

---

## 5. 实时通信机制

### 5.1 WebSocket Hub

位置：`server/internal/realtime/hub.go`  
使用库：`gorilla/websocket`

```
Runtime (Go) → EventBus → WebSocket Hub → Frontend (React Query)
```

### 5.2 前端 WebSocket 客户端

位置：`packages/core/api/ws-client.ts`  
位置：`packages/core/realtime/use-realtime-sync.ts`

前端通过 WebSocket 接收事件后：
- `task:message` → 直接 append 到 React Query 缓存
- `task:completed/failed` → `invalidateQueries` 触发 refetch

### 5.3 Daemon 任务轮询

Daemon 每 10 分钟轮询一次任务：
```
22:07:19.223 WRN claim task failed ... error="task workspace isolation check failed"
22:07:20.319 WRN claim task failed ... error="task workspace isolation check failed"
```

**workspace isolation check failed**：Daemon 注册了多个 workspace，但任务认领时 workspace 隔离校验失败（多 workspace 共用一个 Runtime 导致）。

---

## 6. Worktree 管理

### 6.1 复用机制

Multica 的 Worktree 有三重缓解机制：

| 机制 | 说明 |
|------|------|
| **Worktree 复用** | 同一 repo 同一 task 多次调度只更新现有 worktree，不新建 |
| **Active Root 引用计数** | `markActiveEnvRoot/unmarkActiveEnvRoot`，GC 扫描时跳过正在运行的目录 |
| **Per-Agent 并发限制** | 每个 agent 有 `max_concurrent_tasks`（通常 1-3） |

### 6.2 GC 清理机制

- Issue 状态变为 `done`/`cancelled` + 24h TTL 后清理
- GC 扫描时检查 `isActiveEnvRoot`，运行中任务的目录不会被删除
- 日志示例：`gc: eligible for artifact cleanup component=daemon dir=xxx kind=issue`

### 6.3 潜在风险

高频自动化短时间产生大量不同 issue 时，worktree 会持续积累（等 issue done/cancelled + 24h TTL 才清理）。

---

## 7. 桌面端 CLI 自动安装

### 7.1 Bootstrap 流程

桌面端（Electron）首次启动时自动安装 multica CLI：

位置：`apps/desktop/src/main/cli-bootstrap.ts`

```
App 启动 → state: "installing_cli" → bootstrapCli()
  1. fetch checksums.txt from GitHub Releases
  2. 选择平台 asset (darwin/linux/windows, arm64/amd64)
  3. 下载到 tmp 目录
  4. SHA256 校验
  5. 解压 → 移动到安装目录
     macOS: ~/Library/Application Support/multica-desktop/bin/multica
  6. chmod 755 + ad-hoc codesign (macOS)
```

### 7.2 Daemon 管理 IPC

位置：`apps/desktop/src/main/daemon-manager.ts`

| IPC Handler | 功能 |
|-------------|------|
| `daemon:start` | 启动 daemon |
| `daemon:stop` | 停止 daemon |
| `daemon:restart` | 重启 daemon |
| `daemon:auto-start` | 设置/取消自动启动 |

### 7.3 远程机器连接

1. 远程机器安装 multica CLI
2. `multica login` 认证
3. `multica daemon start` 启动 daemon
4. 出现在 Web/Desktop 的 "Connect Remote" 对话框

---

## 8. 竞品分析

### 8.1 Slock

| 项目 | 信息 |
|------|------|
| 网站 | https://slock.ai / https://app.slock.ai |
| 母公司 | Botiverse (2025) |
| 核心模型 | AI agents as teammates in chat channels/DM，@mention 唤醒 |
| 本地 daemon | `npx @slock-ai/daemon` |
| 开源项目 | cch123/slock-clone, coppynight/slark, first-principle-ai/flock |

**通信机制**：WebSocket 协议，通过 @mention 唤醒/休眠 Agent。

### 8.2 Moxt

（待补充调研）

### 8.3 Cursor Plugins (cursor-team-kit)

（用户研究参考，待补充）

---

## 9. 关键文件索引

### Go Backend

| 文件 | 用途 |
|------|------|
| `server/internal/daemon/` | Daemon 主逻辑 (daemon.go, client.go, types.go) |
| `server/internal/realtime/hub.go` | WebSocket Hub (gorilla/websocket) |
| `server/internal/events/` | EventBus 事件总线 |
| `server/pkg/protocol/events.go` | 事件类型定义 |
| `server/cmd/server/listeners.go` | WebSocket 监听器 |
| `server/internal/handler/issue.go` | Issue API 处理器 |
| `server/internal/service/task.go` | Task 服务层 |
| `server/pkg/agent/*.go` | Agent 适配器 (claude, opencode, hermes, cursor...) |

### Frontend

| 文件 | 用途 |
|------|------|
| `packages/core/api/ws-client.ts` | WebSocket 客户端 |
| `packages/core/realtime/use-realtime-sync.ts` | 实时同步 Hook |
| `packages/core/issues/ws-updaters.ts` | WS 事件处理器 |
| `packages/core/issues/queries.ts` | React Query 查询定义 |
| `packages/core/issues/mutations.ts` | React Query 变更 Hook |
| `packages/core/types/issue.ts` | TypeScript 类型定义 |
| `packages/views/issues/components/issues-page.tsx` | Issue 页面组件 |

### Desktop Client

| 文件 | 用途 |
|------|------|
| `apps/desktop/src/main/cli-bootstrap.ts` | CLI 自动安装 |
| `apps/desktop/src/main/daemon-manager.ts` | Daemon 进程管理 |
| `apps/views/runtimes/components/connect-remote-dialog.tsx` | 远程连接 UI |

### Local Daemon

| 路径 | 用途 |
|------|------|
| `~/.multica/config.json` | Daemon 配置 |
| `~/.multica/daemon.log` | Daemon 日志 |
| `~/multica_workspaces/` | Workspace 工作目录 |
| `~/.hermes/skills/` | Hermes Skills |
| `/opt/data/.env` | 环境变量注入源 |

---

## 附录

### A. 环境信息

- **OS**: Linux ARM64 (aarch64)
- **HOME**: `/opt/data/home`
- **Daemon PID**: 937
- **Daemon 版本**: 0.2.29
- **Daemon 运行时长**: ~137h (约 5.7 天)

### B. 已知问题

1. **GitHub Token 权限限制**：当前 token 只有 pull 权限，无法创建 issue/PR
2. **Linux ARM64 兼容性**：标准 x86_64 二进制包不适用，需使用 ARM64 版本
3. **Workspace Isolation Check Failed**：多 workspace 共用 Runtime 导致认领任务时 workspace 隔离校验失败

### C. 相关文档

- [multica CLI Reference](./CLI_AND_DAEMON.md)
- [Multica GitHub](https://github.com/multica-ai/multica)
