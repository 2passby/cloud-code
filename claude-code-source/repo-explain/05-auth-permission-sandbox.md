# Auth、权限与 Sandbox

## auth 来源与能力判定

主入口在 [`../src/utils/auth.ts`](../src/utils/auth.ts)。

它至少解决 3 类问题：

1. 当前会话是否允许走 Anthropic 1P OAuth
2. token / api key 到底来自哪里
3. 当前用户具有什么套餐或能力

典型判定函数包括：

- `isAnthropicAuthEnabled()`
- `getAuthTokenSource()`
- `getAnthropicApiKeyWithSource()`
- `getClaudeAIOAuthTokens()`
- `getSubscriptionType()`

从实现看，auth 来源会综合：

- `CLAUDE_CODE_OAUTH_TOKEN`
- file descriptor 传入 token
- `ANTHROPIC_AUTH_TOKEN`
- `ANTHROPIC_API_KEY`
- `apiKeyHelper`
- 本地持久化的 `claude.ai` OAuth
- Bedrock / Vertex / Foundry 等第三方 provider

## permission 决策流

权限系统的核心目录是 [`../src/utils/permissions`](../src/utils/permissions)。

### 1. 规则来源

权限规则会从多种 source 汇总到 `ToolPermissionContext`：

- policy settings
- user settings
- project settings
- command / session / cli 临时规则

### 2. 规则行为

持久化规则层的主行为是：

- `allow`
- `deny`
- `ask`

运行时还会出现 `passthrough` 这种决策态，但它更像中间状态，不是主要落盘规则。

### 3. 匹配与预过滤

[`../src/utils/permissions/permissions.ts`](../src/utils/permissions/permissions.ts) 负责规则匹配。

[`../src/tools.ts`](../src/tools.ts) 会在“模型看到工具之前”先根据 deny rule 过滤工具。
这点很关键，因为它说明权限不只是调用时拦截，而是会改变模型可见工具面。

### 4. 交互式审批

交互式审批 UI 在 [`../src/components/permissions`](../src/components/permissions)。
这里按工具类型拆了很多专用组件，例如文件写入、编辑、沙盒、computer-use 等。

## auto / plan mode 的关系

[`../src/utils/permissions/permissionSetup.ts`](../src/utils/permissions/permissionSetup.ts) 把权限系统与运行模式绑在一起。

它处理：

- 默认 permission mode
- auto mode 门控
- 危险 bash / powershell / agent delegation 规则检查
- mode 切换后的上下文修正

一个很重要的约束是：如果某些 allow 规则过于宽泛，会直接破坏 auto mode 的安全边界，因此这里专门识别危险规则。

## sandbox 怎么叠加进来

sandbox 不是权限系统的替代物，而是另一层约束。

核心文件是 [`../src/utils/sandbox/sandbox-adapter.ts`](../src/utils/sandbox/sandbox-adapter.ts)。

它的职责是把 Claude 自己的 permission/settings 规则翻译成 sandbox runtime 可执行配置。

转换内容包括：

- 文件读写允许范围
- 网络 allow / deny
- managed settings 的附加限制
- 默认禁止对某些敏感路径写入

从语义上看：

- permission rule 决定“产品层面是否允许这个工具/命令”
- sandbox 决定“即便允许执行，OS / FS / Network 层还能做什么”

这两层是叠加关系。

## 关键文件

- [`../src/utils/auth.ts`](../src/utils/auth.ts)
- [`../src/utils/permissions/permissions.ts`](../src/utils/permissions/permissions.ts)
- [`../src/utils/permissions/permissionSetup.ts`](../src/utils/permissions/permissionSetup.ts)
- [`../src/utils/permissions/permissionsLoader.ts`](../src/utils/permissions/permissionsLoader.ts)
- [`../src/utils/sandbox/sandbox-adapter.ts`](../src/utils/sandbox/sandbox-adapter.ts)
- [`../src/tools.ts`](../src/tools.ts)
- [`../src/components/permissions`](../src/components/permissions)

## 设计结论

- auth、permission、sandbox 被明确拆成 3 层，而不是揉成一个大 if/else。
- 权限系统不仅管“允不允许执行”，还管“模型能不能看见这个工具”。
- sandbox 是执行层硬约束，permission 是产品层审批约束，两者缺一不可。
