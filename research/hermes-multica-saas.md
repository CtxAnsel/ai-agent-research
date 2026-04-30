# Hermes Agent、Multica 与 SaaS 层打通说明

> 对应 Linear：WEA-6（调研 Hermes agent 与 Multica 集成）。本文把结论与「Multica ↔ SaaS」交互写清楚，便于后续落产品或对接自建服务。

## 1. Hermes 与 Multica：如何打通

**结论：二者可直接打通，标准路径不需要专用中间协议。**

- [Multica](https://github.com/multica-ai/multica) 在本机 **Agent Daemon** 中，会在 `PATH` 上自动探测多种 coding agent CLI，其中包含 **`hermes`**（与 `claude`、`codex`、`openclaw`、`opencode` 等并列）。
- [Hermes Agent](https://github.com/NousResearch/hermes-agent)（Nous Research）提供本机 **`hermes` CLI**；Multica 需要的是「在 Runtime 上可执行的 Hermes CLI」作为 agent 后端，而不是再实现一层「Hermes Gateway ↔ Multica」的私有协议。

**推荐步骤（产品向）：**

1. 在打算接 Multica 的机器上按 [Hermes 文档](https://hermes-agent.nousresearch.com/docs/) 安装，确保终端能直接运行 `hermes`。
2. 安装 Multica CLI，执行 `multica setup`（连 Multica Cloud）或 `multica setup self-host`（自建服务端）。
3. 启动 daemon 后，在 Web 端 **Settings → Runtimes** 确认本机为在线 Runtime，且已识别 Hermes。
4. 在 **Settings → Agents** 新建 Agent，选择对应 Runtime 与 Hermes 提供商；之后在 Issue 上指派任务即可由本机 Hermes 执行。

## 2. ACP（Agent Communication Protocol）与上述路径的关系

Hermes 在官方包中可通过可选依赖 **`hermes-agent[acp]`** 接入 **ACP**（基于 PyPI 上的 `agent-client-protocol`），并提供 **`hermes-acp`** 等入口，用于 **编辑器 / 客户端协议**（例如 IDE 侧通过 ACP 连 Hermes）。

- **Multica 默认打通 Hermes**：依赖的是 **本机 `hermes` CLI + daemon**，**不是**必须装 `[acp]`。
- 若目标是 **Cursor / VS Code 等通过 ACP 连 Hermes**，才需要单独安装 ACP extra 并使用 `hermes-acp`（以本地 `hermes --help` / 官方文档为准）。

## 3. Multica 与 SaaS 层如何交互

此处的 **SaaS 层**指 Multica 托管的 **Web 应用 + API + 数据层**（自建部署时结构相同，只是域名与运维归你）。

官方架构概要：

| 层级 | 技术栈 | 职责 |
|------|--------|------|
| 前端 | Next.js（App Router） | 看板、Issue、Runtime/Agent 配置、登录后 UI |
| 后端 | Go（Chi、sqlc、WebSocket 等） | 认证、工作区、任务排队、向 Runtime 下发、接收进度与结果 |
| 数据 | PostgreSQL（含 pgvector） | 持久化 |
| 本机 | `multica` CLI + Agent Daemon | 鉴权、注册 Runtime、在本地调用 `hermes` 等 CLI |

**交互要点：**

1. **`multica login` / `multica setup`**：浏览器完成认证，CLI 持有调用 **云端（或自建）Go API** 的凭据。
2. **Daemon**：后台运行，向服务端 **上报本机可用 CLI 与在线状态**，使 SaaS 能在 **Settings → Runtimes** 中展示并调度。
3. **任务流**：用户在 SaaS 前端创建/指派 Issue → **Go 后端**将任务路由到已连接的 Runtime → **Daemon** 拉取任务并在本机执行 **`hermes`（或其他已选提供商）** → 通过 **WebSocket 等机制**回传进度与结果 → 前端与评论中可见。

因此：**SaaS 负责任务与人机协作面；本机 Daemon 是「连上云的执行节点」；Hermes 仍只在 Runtime 进程边界内作为 CLI 被调用。**

## 4. 与「自有 SaaS」二次集成时的注意点

若你们另有 **studio / 自有产品 SaaS**，希望与 Multica 协同：

- Multica 已提供的是 **其官方前端 + Go API + Daemon** 这条闭环；与自有 SaaS 的对接通常需要你们定义 **是否调用 Multica 公开/自建 API**、**Webhook**、或 **仅把 Multica 当作独立子系统由运营使用** 等方案。
- 本文不替代 Multica 官方 API 文档；具体接口与版本以 [multica-ai/multica](https://github.com/multica-ai/multica) 仓库及你们部署版本为准。

## 5. 参考链接

- [Multica README](https://github.com/multica-ai/multica/blob/main/README.md)（架构图、CLI、Runtime 说明）
- [Hermes Agent 文档](https://hermes-agent.nousresearch.com/docs/)
- [Hermes Architecture（开发者）](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture)
