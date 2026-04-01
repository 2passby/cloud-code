# Repo Explain

这组文档按 7 个视角拆解这个仓库，重点覆盖 agent 相关设计，而不是泛泛介绍目录。

建议阅读顺序：

1. [01-overview-entry.md](./01-overview-entry.md)
2. [02-skill-system.md](./02-skill-system.md)
3. [03-subagent-team.md](./03-subagent-team.md)
4. [04-agent-loop-and-task-framework.md](./04-agent-loop-and-task-framework.md)
5. [05-auth-permission-sandbox.md](./05-auth-permission-sandbox.md)
6. [06-tools-and-mcp.md](./06-tools-and-mcp.md)
7. [07-remote-sdk-and-coordinator.md](./07-remote-sdk-and-coordinator.md)

这套代码本质上是一个 Bun 构建的 Claude Code CLI 源码还原仓库，主入口在 [`../src/entrypoints/cli.tsx`](../src/entrypoints/cli.tsx)，真正的启动编排在 [`../src/main.tsx`](../src/main.tsx)。它不是单一的“聊天循环”，而是把命令系统、工具系统、技能系统、任务系统、MCP 客户端、权限系统、远程会话和多代理协作都塞进了一个终端应用里。

先抓住 4 条主线：

- 入口主线：`cli.tsx -> main.tsx -> REPL / QueryEngine`
- 执行主线：`query.ts -> tool_use -> tool_result -> 下一轮 query`
- 协作主线：`AgentTool / TeamCreate / SendMessage -> local_agent / teammate / remote_agent`
- 安全主线：`auth -> permission rules -> sandbox adapter -> tool filtering`

如果你只想快速定位代码：

- 启动与总装配看 [`../src/main.tsx`](../src/main.tsx)
- 会话/模型循环看 [`../src/query.ts`](../src/query.ts) 和 [`../src/QueryEngine.ts`](../src/QueryEngine.ts)
- 工具池与 MCP 看 [`../src/tools.ts`](../src/tools.ts) 和 [`../src/services/mcp/client.ts`](../src/services/mcp/client.ts)
- skill 看 [`../src/skills/loadSkillsDir.ts`](../src/skills/loadSkillsDir.ts) 和 [`../src/tools/SkillTool/SkillTool.ts`](../src/tools/SkillTool/SkillTool.ts)
- subagent/team 看 [`../src/tasks`](../src/tasks) 和 [`../src/utils/swarm`](../src/utils/swarm)
- 权限与沙盒看 [`../src/utils/permissions`](../src/utils/permissions) 和 [`../src/utils/sandbox/sandbox-adapter.ts`](../src/utils/sandbox/sandbox-adapter.ts)
