# 工具注册与 MCP 集成

## Tool 抽象

统一工具抽象在 [`../src/Tool.ts`](../src/Tool.ts)。

这里定义了一个工具至少要提供哪些能力，例如：

- `name`
- `inputSchema`
- `call`
- `render`
- `isEnabled`
- `checkPermissions`

大部分内置工具都会通过 `buildTool()` 构造。

## 内置工具池如何装配

核心文件是 [`../src/tools.ts`](../src/tools.ts)。

它做了三层装配：

### 1. 声明层

`getAllBaseTools()` 枚举所有可能的 built-in tools。
这里会按 feature flag、环境变量、运行模式决定某些工具是否进入候选池。

### 2. 过滤层

`getTools(permissionContext)` 会继续处理：

- simple mode
- REPL mode
- deny rule 过滤
- `isEnabled()` 检查

### 3. 合并层

`assembleToolPool(permissionContext, mcpTools)` 把 built-in tools 和 MCP tools 合并成最终工具池。

它会：

- 先处理 built-in
- 再过滤 MCP deny rule
- 按名字排序
- 按名字去重

## MCP 接入链路

MCP 主入口在 [`../src/services/mcp/client.ts`](../src/services/mcp/client.ts)。

这个模块负责：

- 建立 MCP 连接
- 拉取 `tools/list`
- 拉取 `prompts/list`
- 拉取 `resources/list`
- 把它们包装成仓库内部统一抽象

命名规则相关文件：

- [`../src/services/mcp/normalization.ts`](../src/services/mcp/normalization.ts)
- [`../src/services/mcp/mcpStringUtils.ts`](../src/services/mcp/mcpStringUtils.ts)

最终工具名会变成：

`mcp__server__tool`

## MCP tool 如何变成内部 Tool

MCP 工具最终会被包装成 [`../src/tools/MCPTool/MCPTool.ts`](../src/tools/MCPTool/MCPTool.ts) 风格的内部工具。

包装后的工具仍然具备统一接口：

- schema
- permissions
- call
- render

这样 query loop 不需要知道它是 built-in 还是 MCP。

## 资源与认证补链

MCP 不只有 tools，还有 resource 和 auth 补链。

### 1. 资源

相关工具：

- [`../src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts`](../src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts)
- [`../src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts`](../src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts)

### 2. 认证

未认证或认证失效的 MCP server 会通过伪工具路径接入：

- [`../src/tools/McpAuthTool/McpAuthTool.ts`](../src/tools/McpAuthTool/McpAuthTool.ts)

## computer-use 与特殊 MCP server

computer-use 在这个仓库里不是普通外部 server，而是带专用渲染和权限桥接的“特殊 MCP server”。

关键文件：

- [`../src/utils/computerUse/setup.ts`](../src/utils/computerUse/setup.ts)
- [`../src/utils/computerUse/wrapper.tsx`](../src/utils/computerUse/wrapper.tsx)
- [`../src/utils/computerUse/common.ts`](../src/utils/computerUse/common.ts)
- [`../src/utils/computerUse/mcpServer.ts`](../src/utils/computerUse/mcpServer.ts)

它的特点是：

- 仍然以 `mcp__computer-use__*` 形式出现在工具层
- 但调用和渲染会被 wrapper 覆盖
- 会额外接入自己的审批与锁机制

Claude-in-Chrome 也是类似思路。

## 关键文件

- [`../src/Tool.ts`](../src/Tool.ts)
- [`../src/tools.ts`](../src/tools.ts)
- [`../src/services/mcp/client.ts`](../src/services/mcp/client.ts)
- [`../src/tools/MCPTool/MCPTool.ts`](../src/tools/MCPTool/MCPTool.ts)
- [`../src/utils/mcpValidation.ts`](../src/utils/mcpValidation.ts)
- [`../src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts`](../src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts)
- [`../src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts`](../src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts)
- [`../src/tools/McpAuthTool/McpAuthTool.ts`](../src/tools/McpAuthTool/McpAuthTool.ts)
- [`../src/utils/computerUse`](../src/utils/computerUse)

## 设计结论

- `Tool` 是统一执行抽象，MCP 只是另一种 Tool 来源。
- built-in、MCP、resource、auth pseudo-tool 被装配进同一工具面，这是这个系统能扩展的基础。
- computer-use 属于“特殊 MCP server + 本地 override”，不是完全独立于工具体系的私有功能。
