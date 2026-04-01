# 总览与入口

## 仓库定位

这个仓库是 `@anthropic-ai/claude-code` `v2.1.88` 的源码还原版本，构建说明在 [`../README.md`](../README.md)，项目配置在 [`../package.json`](../package.json)。

从代码组织看，它不是简单的 CLI 外壳，而是一个把以下能力合并在一起的终端 agent 宿主：

- 命令系统
- 工具系统
- 多轮 query/模型循环
- 任务与后台执行
- skills
- MCP 客户端
- 远程会话
- 多代理协作
- 终端 UI

## 启动链

最短调用链是：

`package.json -> src/entrypoints/cli.tsx -> src/main.tsx -> REPL 或 QueryEngine`

具体分层如下：

1. [`../src/entrypoints/cli.tsx`](../src/entrypoints/cli.tsx)
   负责超薄启动层和 fast-path。
   这里会优先处理 `--version`、MCP/native-host、bridge、daemon、background session 等特殊路径，只有命中普通主流程时才动态导入 `main.tsx`。
2. [`../src/main.tsx`](../src/main.tsx)
   负责真正的系统编排。
   它会做 settings 预热、keychain 预取、policy/remote settings 加载、commands 与 tools 装配、MCP 初始化、resume/continue 处理、interactive 与 non-interactive 分流。
3. 交互式主界面走 [`../src/screens/REPL.tsx`](../src/screens/REPL.tsx)。
4. 非交互或 SDK 会话走 [`../src/QueryEngine.ts`](../src/QueryEngine.ts)。

## 初始化阶段装配了什么

[`../src/main.tsx`](../src/main.tsx) 是这个仓库最重的编排枢纽。它在初始化阶段把这些系统接到一起：

- `bootstrap/state` 进程级全局状态
- `commands` 命令注册
- `tools` 内置工具池
- `skills/bundled` 内置 skill 注册
- `services/mcp` MCP server 配置、连接、工具/资源注入
- `services/policyLimits` 与 `services/remoteManagedSettings`
- `state/AppStateStore` UI 侧状态
- `plugins`、`telemetry`、`settings`、`session restore`

这意味着后续很多“功能入口”都不会直接出现在某个子目录里，而是先在 `main.tsx` 被装配，再通过 `AppState`、`ToolUseContext`、`QueryEngine` 或 `REPL` 传播。

## 两层状态模型

这个仓库最容易混淆的是状态分层。

### 1. 进程级状态

[`../src/bootstrap/state.ts`](../src/bootstrap/state.ts) 管的是全局单例状态，例如：

- session id
- cwd 与 original cwd
- main loop model
- session source / client type
- skills 调用记录
- remote / oauth 相关会话级状态

它更像“运行时环境寄存器”。

### 2. UI / 会话状态

[`../src/state/AppStateStore.ts`](../src/state/AppStateStore.ts) 管的是交互界面和执行态，例如：

- messages
- tasks
- teamContext
- toolPermissionContext
- MCP 连接状态
- remote 会话状态
- thinking / loading / notifications

它更像“当前会话的可渲染状态树”。

## task 有两套含义

这个仓库里 `task` 不是一个概念。

- [`../src/tasks`](../src/tasks) 下的是运行时后台任务实现，例如 `local_agent`、`remote_agent`、`local_bash`
- [`../src/utils/tasks.ts`](../src/utils/tasks.ts) 是持久化 task list / todo 体系，给计划、协作、team 使用

如果把这两个混为一谈，很容易误判调用链。

## 最值得先读的文件

- [`../src/entrypoints/cli.tsx`](../src/entrypoints/cli.tsx)
- [`../src/main.tsx`](../src/main.tsx)
- [`../src/bootstrap/state.ts`](../src/bootstrap/state.ts)
- [`../src/query.ts`](../src/query.ts)
- [`../src/QueryEngine.ts`](../src/QueryEngine.ts)
- [`../src/screens/REPL.tsx`](../src/screens/REPL.tsx)
- [`../src/tools.ts`](../src/tools.ts)

## 设计特点

- 启动层做了大量 fast-path 和动态导入，目标很明确：尽量减少无关模块求值。
- `main.tsx` 是一个总装配中心，优点是入口集中，缺点是职责非常重。
- 整个系统偏“宿主型架构”：不是某个单独 agent 模块驱动一切，而是多个子系统通过统一状态与上下文拼在一起。
