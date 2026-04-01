# Agent Loop 与 Task Framework

## 真正的主循环在哪里

如果只看名字，很多人会先去看 `QueryEngine`。但真正的 agentic loop 主体在 [`../src/query.ts`](../src/query.ts)。

[`../src/QueryEngine.ts`](../src/QueryEngine.ts) 更像外层会话控制器，它负责：

- 维护 `mutableMessages`
- 处理用户输入
- 组装 system prompt / user context
- 持久化 transcript
- 把内部消息映射成 SDK 消息流

真正反复执行 `model -> tool_use -> tool_result -> next turn` 的逻辑，在 `query.ts` 的 `queryLoop()`。

## query / tool / task 三者关系

这三者不是一层东西。

### 1. query loop

`queryLoop()` 每轮大致做这些事：

1. 基于当前 messages 组装请求上下文
2. 在请求前做 compact / collapse / budget 处理
3. 发起模型流式请求
4. 收集 assistant message 和 `tool_use`
5. 执行工具
6. 把 `tool_result` 和附加消息拼回 messages
7. 若需要继续，进入下一轮

### 2. 工具执行

工具执行是 query loop 的一个阶段，不是独立主循环。

tool 由统一 `Tool` 抽象执行，结果再回灌到 query 上下文。

### 3. task framework

task framework 是后台任务状态机，不直接驱动主模型采样。

[`../src/Task.ts`](../src/Task.ts) 定义了：

- `TaskType`
- `TaskStatus`
- task id 规则

[`../src/utils/task/framework.ts`](../src/utils/task/framework.ts) 提供：

- `registerTask`
- `updateTaskState`
- `pollTasks`
- `evictTerminalTask`

它负责的是后台 agent / shell / workflow 任务的状态与通知，而不是 query 的 while 循环。

## 计划模式怎么插入循环

plan mode 不是另一套 agent loop，而是权限模式和工具可用面的切换。

相关入口有：

- [`../src/commands/plan/plan.tsx`](../src/commands/plan/plan.tsx)
- [`../src/tools/EnterPlanModeTool/EnterPlanModeTool.ts`](../src/tools/EnterPlanModeTool/EnterPlanModeTool.ts)
- [`../src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`](../src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts)

运行时效果是：

- 当前 `toolPermissionContext.mode` 变为 `plan`
- 工具与权限行为改变
- 退出 plan 后把审批过的结果作为 tool result 回到同一条 query 链

所以 plan mode 是“外层上下文插桩”，不是“内层循环替换”。

## compact 怎么插入循环

compact 插在模型请求之前。

相关目录是 [`../src/services/compact`](../src/services/compact)。

常见插点包括：

- auto compact
- session memory compact
- snip / projection
- context collapse

它们的共同目标是：在不破坏主会话语义的前提下，控制上下文长度和内存占用。

[`../src/QueryEngine.ts`](../src/QueryEngine.ts) 还会在收到 compact boundary 后裁剪自己的 `mutableMessages`，避免 headless 会话无限增长。

## background / interrupt / resume

### 1. interrupt

通过 `AbortController` 贯穿模型请求和工具执行。

### 2. background

主会话可以转成后台任务，典型实现见：

- [`../src/tasks/LocalMainSessionTask.ts`](../src/tasks/LocalMainSessionTask.ts)

### 3. resume

resume 不是恢复某个 while 循环栈帧，而是恢复：

- messages
- file history
- session metadata
- context collapse snapshot
- plan 相关引用

然后再从新的会话状态继续进入 `queryLoop()`。

## 关键文件

- [`../src/query.ts`](../src/query.ts)
- [`../src/QueryEngine.ts`](../src/QueryEngine.ts)
- [`../src/query/deps.ts`](../src/query/deps.ts)
- [`../src/Task.ts`](../src/Task.ts)
- [`../src/utils/task/framework.ts`](../src/utils/task/framework.ts)
- [`../src/tasks/LocalMainSessionTask.ts`](../src/tasks/LocalMainSessionTask.ts)
- [`../src/commands/plan/plan.tsx`](../src/commands/plan/plan.tsx)
- [`../src/services/compact`](../src/services/compact)

## 设计结论

- 真正的 agent loop 在 `query.ts`，`QueryEngine` 是包装层。
- task framework 管的是后台执行，不是模型采样本身。
- plan mode、compact、resume 都是围绕同一套 query loop 打补丁，而不是复制一套新循环。
