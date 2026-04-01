# Skill 系统

## skill 从哪里来

这个仓库里的 skill 不是单一来源，而是多源汇总后的统一 `Command`。

核心入口在 [`../src/skills/loadSkillsDir.ts`](../src/skills/loadSkillsDir.ts)。

主要来源有：

- bundled skills
  由 [`../src/skills/bundled/index.ts`](../src/skills/bundled/index.ts) 在启动时注册
- 文件型 skills
  从 `managed`、`user`、`project`、`--add-dir` 指向的 `.claude/skills` 中加载
- legacy commands
  从旧的 `.claude/commands` 兼容加载
- MCP skills
  `SkillTool` 会把 `AppState.mcp.commands` 中 `loadedFrom === 'mcp'` 的 prompt 合并进来
- plugin / managed skill
  仍然会落到统一的 `Command` 抽象上

## 加载、缓存、去重

### 1. 目录扫描

[`../src/skills/loadSkillsDir.ts`](../src/skills/loadSkillsDir.ts) 的 `getSkillDirCommands()` 会并行加载：

- managed skills
- user skills
- project skills
- additional dirs
- legacy commands

### 2. frontmatter 解析

同一个文件里会解析 skill frontmatter，提取这些元信息：

- `description`
- `when_to_use`
- `allowed-tools`
- `arguments`
- `model`
- `effort`
- `hooks`
- `context`
- `paths`

其中 `context: fork` 会直接影响执行方式。

### 3. 去重

skill 去重不是按名字，而是按真实文件路径 `realpath` 去重。
这样可以避免同一 skill 通过符号链接或重叠父目录被重复加载。

### 4. 缓存

`getSkillDirCommands()` 自身是 `memoize` 的。
[`../src/skills/loadSkillsDir.ts`](../src/skills/loadSkillsDir.ts) 还提供了 `clearSkillCaches()`，会同时清空目录加载缓存、条件 skill 状态等。

## 动态发现与热更新

### 1. 动态发现

同一个文件里维护了两类 session 内动态状态：

- `dynamicSkills`
- `conditionalSkills`

`conditionalSkills` 是带 `paths` frontmatter 的 skill。它们不会立即暴露，而是等当前操作文件命中路径规则后，通过 `activateConditionalSkillsForPaths()` 激活。

### 2. 额外目录

`--add-dir` 指向的目录也会被纳入 `.claude/skills` 搜索范围，这使得 skill 可以跨目录挂载，而不一定只来自当前 repo。

### 3. 热更新

[`../src/utils/skills/skillChangeDetector.ts`](../src/utils/skills/skillChangeDetector.ts) 用 `chokidar` 监视：

- `~/.claude/skills`
- `~/.claude/commands`
- `./.claude/skills`
- `./.claude/commands`
- `--add-dir` 对应的 `.claude/skills`

变更后会触发：

- `clearSkillCaches()`
- command 相关缓存清理
- `skillsChanged` 信号通知

在 Bun 下它还专门切到了轮询模式，规避 `fs.watch()` 死锁问题。

## skill 怎么执行

skill 的执行入口在 [`../src/tools/SkillTool/SkillTool.ts`](../src/tools/SkillTool/SkillTool.ts)。

它有两种主路径。

### 1. inline 执行

默认路径是：

`SkillTool -> processPromptSlashCommand -> 生成 newMessages/contextModifier -> 回到主 query`

这类 skill 更像“展开一段 prompt/命令模板”，不会独立起 agent。

它还可以通过 `contextModifier()` 动态修改：

- `allowedTools`
- `mainLoopModel`
- `effort`

### 2. fork 执行

当 skill 的 `context === 'fork'` 时，`SkillTool` 会走 `executeForkedSkill()`。

这条链路是：

`SkillTool -> prepareForkedCommandContext -> runAgent()`

也就是说，fork skill 本质上是“在隔离 sub-agent 里执行 skill prompt”。

这也是 repo 里 skill 和 subagent 直接接触的关键点。

## 与会话状态的耦合点

skill 不是完全无状态的，它至少耦合了这些系统：

- [`../src/bootstrap/state.ts`](../src/bootstrap/state.ts)
  记录 invoked skills，用于 compaction 后保留上下文
- `discoveredSkillNames`
  由 [`../src/QueryEngine.ts`](../src/QueryEngine.ts) 维护，给 skill discovery / telemetry 用
- hooks
  skill frontmatter 可以带 hooks
- model / effort
  skill 可以局部覆盖主会话配置

## 关键文件

- [`../src/skills/loadSkillsDir.ts`](../src/skills/loadSkillsDir.ts)
- [`../src/skills/bundled/index.ts`](../src/skills/bundled/index.ts)
- [`../src/tools/SkillTool/SkillTool.ts`](../src/tools/SkillTool/SkillTool.ts)
- [`../src/utils/skills/skillChangeDetector.ts`](../src/utils/skills/skillChangeDetector.ts)
- [`../src/bootstrap/state.ts`](../src/bootstrap/state.ts)

## 设计要点

- skill 在实现上更接近“带 frontmatter 的 prompt command”，而不是单独 DSL。
- inline 与 fork 两种执行模式，让它既能当 prompt 宏，也能当轻量 agent 模板。
- 动态 skill、条件 skill、MCP skill 被统一到了同一套 `Command` 表面，这是这个系统的关键抽象收益。
