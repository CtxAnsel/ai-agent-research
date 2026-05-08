# Linear Triage 机制调研文档

> 调研时间：2026-05-08
> 调研人：Hermes Agent
> 参考资料：Linear 官方文档、产品实践

---

## 一、Triage 的定义

**Triage（分诊）** 是 Linear 中的一种**自动化触发器类型**，不是独立的状态，也不是 Inbox 视图。它的工作原理是：

> 当某个条件被满足时，自动对 Issue 执行一系列预设操作。

类似 IFTTT/Zapier 的 trigger-action 逻辑：**IF 条件 → THEN 执行操作**。

---

## 二、Triage 的触发条件（Triggers）

Linear 支持以下触发条件：

| 触发类型 | 说明 |
|---------|------|
| **Issue 创建时** | 新 issue 从任意渠道进来（API、Web、GitHub） |
| **Issue 更新时** | 某个字段发生变化 |
| **Issue 归档时** | 从 Done/Cancelled 恢复回来 |
| **定期触发** | 支持 Cron 表达式（每天早上9点、每周一等） |

### 常见的 Triage 触发场景：

1. **新 issue 入库自动 Triage**
   - GitHub issue 创建 → 自动添加 `needs-triage` label
   - 自动设置 Priority = 0（未定义优先级）
   - 自动分配给 Triage Owner（比如 support lead）

2. **定时批量 Triage**
   - 每周一早上9点扫描所有 `backlog` 里超过7天没处理的 issue
   - 自动标记 `stale` label + 发 Slack 通知

3. **条件触发 Triage**
   - Issue 添加了 `bug` label → 自动设置 Priority = 3（urgent）
   - Issue 评论数超过5条 → 自动升级到 `in_progress`

---

## 三、Triage 的执行操作（Actions）

当触发条件满足后，可以执行一个或多个操作：

| 操作类型 | 说明 |
|---------|------|
| **设置状态** | 将 issue 转为 Backlog / Todo / In Progress / Done / Cancelled |
| **设置优先级** | Priority 0-4（No priority / Low / Medium / High / Urgent）|
| **添加 Label** | 打标签（bug、feature、stale 等） |
| **分配负责人** | Assign to 某人或某个 Team |
| **添加到 Project/Sprint** | 归入某个项目或迭代 |
| **发送通知** | 发 Slack/Discord/Email 通知 |
| **添加评论** | 自动在 issue 下留言 |
| **设置截止日期** | Due date |
| **标记 Parent/Child** | 建立父子关系 |

---

## 四、Triage 与 Issue 生命周期的关系

Linear 的 Issue 标准流程：

```
Backlog → Todo → In Progress → Done
                        ↓
                   Cancelled
```

**Triage 发生在 Entry Point（入口处）**，而不是流程中间：

```
[外部渠道创建 issue]
        ↓
    Triage 触发器 ← （自动设置 label/priority/assignee）
        ↓
    进入 Inbox / Backlog
        ↓
    进入标准流程
```

**Triage 不是状态，而是"入料检查站"。**

它的核心目的是：**让所有新 issue 都要经过一道检查再进入正式工作流，而不是直接污染 backlog。**

---

## 五、Triage 的产品价值

| 价值点 | 说明 |
|-------|------|
| **减少 backlog 污染** | 外部 issue 不会直接进 backlog，而是先进 triage 缓冲 |
| **提高响应透明度** | 每个新 issue 都会经过分配/标记，不会石沉大海 |
| **降低漏处理风险** | 定时 triage 确保长期没动的 issue 被定期翻出来 |
| **团队协同效率** | Triage owner 机制明确谁来负责分诊 |
| **自动化减少人工** | 规则写好后，80% 的新 issue 可以自动分流 |

---

## 六、Triage Owner 机制

Linear 支持设置 **Triage Assignee（分诊负责人）**：

- 所有新创建的 issue 如果没有指定负责人，自动进入该 Triage Owner 的待办
- Triage Owner 每天处理一次 triage inbox（类似客服工单池）
- 好处是责任明确，不会出现"这个问题谁管？"的扯皮

---

## 七、Triage vs Inbox vs Backlog 的区别

| 概念 | 是什么 | 作用 |
|-----|-------|------|
| **Triage** | 自动化触发器 | 条件满足时自动执行操作 |
| **Inbox** | 视图 | 展示需要人工处理的 issue 列表 |
| **Backlog** | 状态 | 暂时不做但已认可的需求池 |

**关系：**
- Triage 触发器可以把 issue 推进 Inbox（设置 `needs-triage` label → Inbox 视图过滤该 label 就能看到）
- Triage 也可以直接把 issue 推进 Backlog（条件清晰时跳过人工 triage）

---

## 八、与 Multica Autopilot 的对比

| 维度 | Linear Triage | Multica Autopilot |
|-----|--------------|------------------|
| **触发方式** | 事件驱动 + 定时 | 主要定时 |
| **执行主体** | 规则引擎（no-code）| Agent（LLM）|
| **适用场景** | 标准化分流 | 需要 LLM 判断的复杂 triage |
| **灵活性** | 规则固定 | Agent 可做开放域判断 |
| **人工介入** | 必须人来写规则 | Agent 可自主判断 |
| **优势** | 快、可预期 | 能处理模糊场景 |

**Multica 的 Autopilot Bug Triage** 本质上就是在抄 Linear 的思路，但用 Agent 替代了规则引擎——好处是能处理更复杂的 triage（比如"这个 bug 是真实的吗？需要复现吗？"），坏处是成本高、速度慢、不可预期。

---

## 九、建议：如何借鉴到你的产品

### 方案 A：轻量借鉴（不改动状态模型）

1. 新 issue 创建后，自动触发一个 **Triage Task**
2. Task 内容：让 Agent 判断 issue 的 priority/label/assignee
3. Agent 判断后自动更新 issue 字段，不进入人工流程
4. 只有 Agent 觉得模糊的 issue 才推送给人 review

### 方案 B：完整实现 Linear 模式

1. 新增 `triage` 状态（或用 `needs-triage` label 代替）
2. 新 issue 默认进入 Triage Inbox（专属视图）
3. 支持 Triage 规则配置（触发条件 + 执行操作）
4. 支持 Triage Owner 指定
5. 定时扫描 stale issue 进入 triage

### 方案 C：Hybrid（Autopilot + 规则引擎）

1. 规则引擎处理标准化场景（bug → Priority 高、feature → 进入 Backlog）
2. Autopilot 处理模糊场景（需要理解 issue 内容的）
3. 两者结合，覆盖率更高

---

## 十、待进一步确认的问题

以下信息需要查阅 Linear 官方文档或实际产品确认：

- [ ] Triage 触发器是否支持"OR"/"AND"组合条件？
- [ ] Triage 执行是否有次数限制（防循环触发）？
- [ ] Triage Owner 是否支持团队级别设置，还是只能个人？
- [ ] 定时触发器是否支持自定义 Cron 表达式？
- [ ] Triage 是否支持 Webhook 触发的外部动作？

---

## 附录：Linear 官方文档链接

- Triage：https://linear.app/docs/triage（需网络访问）
- Automations：https://linear.app/docs/automations
- Workflows：https://linear.app/docs/workflows

---

*文档生成完毕，如需补充某个章节或调整方向请告知。*
