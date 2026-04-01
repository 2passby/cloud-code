# Remote、SDK 与 Coordinator

## 高阶编排方式有哪些

除了本地 REPL 和普通 subagent 之外，这个仓库还预留了几类更高阶的编排模式：

- remote session
- SDK message adapter
- coordinator mode
- 通过 build feature flag 控制的内部模式

这些能力并不总在外部构建里开启，但代码骨架已经存在。

## remote session 消息流

核心文件是 [`../src/remote/RemoteSessionManager.ts`](../src/remote/RemoteSessionManager.ts)。

它做了两件事：

- 用 WebSocket 订阅远程 session 消息
- 用 HTTP POST 发送用户消息到远程 session

其中控制消息和普通 SDK 消息被明确拆开了。

### 控制消息

`control_request`
主要用于远程侧向本地发起权限请求。

`RemoteSessionManager` 会：

1. 收到 `control_request`
2. 识别 `can_use_tool`
3. 保存 pending permission request
4. 回调本地 UI/宿主处理
5. 再通过 `respondToPermissionRequest()` 发回远端

### 普通 SDK 消息

普通 SDK 消息会继续流向展示层。

[`../src/remote/sdkMessageAdapter.ts`](../src/remote/sdkMessageAdapter.ts) 负责把 SDK 消息转换成 REPL 可显示消息。

它会刻意忽略一些非展示型事件，例如：

- `auth_status`
- `tool_use_summary`
- `rate_limit_event`

这说明 SDK 协议比 REPL 展示协议更丰富，adapter 负责做裁剪。

## coordinator mode 是什么

核心文件是 [`../src/coordinator/coordinatorMode.ts`](../src/coordinator/coordinatorMode.ts)。

这个模式的本质不是“另一种 UI”，而是把主 agent 的 system prompt 和 worker 可用工具面改造成协调者模式。

它做的关键事情包括：

- 通过 `CLAUDE_CODE_COORDINATOR_MODE` 判定当前是否进入 coordinator
- 在 resume 时用 `matchSessionMode()` 纠正会话模式
- 通过 `getCoordinatorUserContext()` 告诉 leader：
  - worker 能用哪些工具
  - 有哪些 MCP server
  - 是否有 scratchpad
- 通过 `getCoordinatorSystemPrompt()` 强化 leader/worker 协作规则

这套设计说明 coordinator 模式更像“特化的 agent prompt + tool surface”，而不是全新 runtime。

## feature flag 的影响

[`../build.ts`](../build.ts) 定义了大量编译期开关。

与本篇直接相关的开关包括：

- `COORDINATOR_MODE`
- `BRIDGE_MODE`
- `DIRECT_CONNECT`
- `CHICAGO_MCP`
- `MCP_SKILLS`
- `KAIROS`

在当前这个还原仓库的外部默认构建里，很多高级模式默认是关闭的。
这意味着：

- 代码里能看到架构骨架
- 但并不代表外部构建一定把这些能力全部编译进最终行为

## 关键文件

- [`../src/remote/RemoteSessionManager.ts`](../src/remote/RemoteSessionManager.ts)
- [`../src/remote/sdkMessageAdapter.ts`](../src/remote/sdkMessageAdapter.ts)
- [`../src/coordinator/coordinatorMode.ts`](../src/coordinator/coordinatorMode.ts)
- [`../src/entrypoints/sdk`](../src/entrypoints/sdk)
- [`../build.ts`](../build.ts)

## 设计结论

- remote session 把“会话执行”与“本地展示/审批”拆开了。
- sdkMessageAdapter 明确承担协议裁剪层，不让 REPL 直接吞原始 SDK 事件。
- coordinator mode 不是第二套 executor，而是基于 prompt、tool surface、worker protocol 的模式化改写。
