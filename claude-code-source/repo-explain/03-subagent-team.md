# Subagent、Team 与 Worker 设计

## 这里有哪些子代理形态

从运行时实现看，仓库里至少有 4 类 agent / worker 执行形态：

- `local_agent`
  本地后台 agent task
- `in_process_teammate`
  同进程 teammate
- `remote_agent`
  远程会话型 agent
- pane teammate
  通过 `tmux` / `iTerm2` 启动的独立 Claude 进程

这些形态不是平行暴露给用户的概念，而是被 `AgentTool`、`TeamCreateTool`、`SendMessageTool`、swarm backend 等能力组合出来的。

## team 是怎么建起来的

建队入口在 [`../src/tools/TeamCreateTool/TeamCreateTool.ts`](../src/tools/TeamCreateTool/TeamCreateTool.ts)。

它做了 4 件关键事：

1. 生成 team name 和 lead agent id
2. 通过 [`../src/utils/swarm/teamHelpers.ts`](../src/utils/swarm/teamHelpers.ts) 落盘 team file
3. 创建对应的 task list 目录
4. 在 `AppState.teamContext` 里注册 leader 与 teammate 视图

所以 team 既有磁盘态，也有内存态。

## subagent 创建链路

### 1. in-process teammate

最关键的链路是：

`registry -> InProcessBackend -> spawnInProcessTeammate -> registerTask -> startInProcessTeammate -> runWithTeammateContext -> runAgent`

关键文件：

- [`../src/utils/swarm/backends/registry.ts`](../src/utils/swarm/backends/registry.ts)
- [`../src/utils/swarm/backends/InProcessBackend.ts`](../src/utils/swarm/backends/InProcessBackend.ts)
- [`../src/utils/swarm/spawnInProcess.ts`](../src/utils/swarm/spawnInProcess.ts)
- [`../src/utils/swarm/inProcessRunner.ts`](../src/utils/swarm/inProcessRunner.ts)
- [`../src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`](../src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx)

### 2. pane teammate

这类 worker 走进程外 backend：

`registry -> detect backend -> PaneBackendExecutor.spawn -> create pane -> send Claude command to pane`

它依赖：

- `tmux`
- `iTerm2`
- team file
- pane id / mailbox

关键文件：

- [`../src/utils/swarm/backends/PaneBackendExecutor.ts`](../src/utils/swarm/backends/PaneBackendExecutor.ts)
- [`../src/utils/swarm/backends/TmuxBackend.ts`](../src/utils/swarm/backends/TmuxBackend.ts)
- [`../src/utils/swarm/backends/ITermBackend.ts`](../src/utils/swarm/backends/ITermBackend.ts)

### 3. remote_agent

`remote_agent` 是本地 task + 远程执行的组合。
本地任务只负责轮询、状态同步和结束归档，真正执行发生在远端。

关键文件：

- [`../src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`](../src/tasks/RemoteAgentTask/RemoteAgentTask.tsx)

### 4. local_agent

这是最“普通”的后台 agent 形态。
它通过任务系统挂到 `AppState.tasks`，用 `pendingMessages`、background signal、resume 机制推进。

关键文件：

- [`../src/tasks/LocalAgentTask/LocalAgentTask.tsx`](../src/tasks/LocalAgentTask/LocalAgentTask.tsx)

## 通信、等待与关闭

### 消息

统一消息入口是 [`../src/tools/SendMessageTool/SendMessageTool.ts`](../src/tools/SendMessageTool/SendMessageTool.ts)。

它不只负责“给 agent 发一句话”，还承载了：

- 普通消息
- broadcast
- shutdown request / response
- plan approval response
- 对 `uds:` / `bridge:` 目标的转发

一个重要设计点是：即便 `in_process_teammate` 在同一进程里，也尽量复用 mailbox / message 通道，而不是偷偷走函数调用。

### 等待

这里没有一个完全统一的 `wait` 抽象，不同 agent 形态等待方式不同：

- `remote_agent` 依赖 poll
- `in_process_teammate` 依赖 mailbox、idle 状态和回调
- `local_agent` 依赖 task state 与 background resume

这也是这个架构后期继续演化时最容易出现重复封装的地方。

### 关闭

关闭策略按形态区分：

- `local_agent`
  abort + cleanup
- `remote_agent`
  标记 killed 并归档远程会话
- `in_process_teammate`
  优先发 shutdown request，必要时强制 abort
- pane teammate
  可优雅 terminate，也可直接 kill pane

## 隔离机制

这部分是 agent 设计里最值得注意的。

### 1. 同进程隔离

`in_process_teammate` 通过 `AsyncLocalStorage` 承载 `TeammateContext`，再辅以独立 `AbortController` 做隔离。

这说明它不是简单地“在同一线程里跑函数”，而是显式模拟了 worker 身份边界。

### 2. 进程隔离

pane teammate 通过独立终端 pane 启动新的 Claude 进程，天然隔离。

### 3. 远程隔离

`remote_agent` 用远程 session 作为边界。

## 关键文件

- [`../src/tools/TeamCreateTool/TeamCreateTool.ts`](../src/tools/TeamCreateTool/TeamCreateTool.ts)
- [`../src/tools/SendMessageTool/SendMessageTool.ts`](../src/tools/SendMessageTool/SendMessageTool.ts)
- [`../src/tasks/LocalAgentTask/LocalAgentTask.tsx`](../src/tasks/LocalAgentTask/LocalAgentTask.tsx)
- [`../src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`](../src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx)
- [`../src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`](../src/tasks/RemoteAgentTask/RemoteAgentTask.tsx)
- [`../src/utils/swarm/backends/registry.ts`](../src/utils/swarm/backends/registry.ts)
- [`../src/utils/swarm/spawnInProcess.ts`](../src/utils/swarm/spawnInProcess.ts)
- [`../src/utils/swarm/teamHelpers.ts`](../src/utils/swarm/teamHelpers.ts)
- [`../src/utils/teamDiscovery.ts`](../src/utils/teamDiscovery.ts)

## 设计结论

- 这个仓库不是只有一种 subagent 实现，而是本地、同进程、远程、终端 pane 并存。
- team 是一层更高的协作壳，底下可以套不同执行 backend。
- 它最强的设计点在于“统一消息协议 + 多 backend”，最弱的点在于“等待与生命周期抽象还不完全统一”。
