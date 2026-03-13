# GitHub Copilot Plugin 与 Claude Code Plugin 的核心区别

本文基于官方文档，总结 GitHub Copilot CLI plugin 与 Claude Code plugin 在结构、能力边界、hooks、marketplace 和迁移语义上的关键差异。它的目标不是做功能宣传，而是帮助你判断：哪些内容可以直接映射，哪些内容必须重写，哪些内容只能人工复核。

## 一句话结论

两套系统的“plugin”概念相似，都是把 skills、agents、hooks、MCP、LSP 等组件打包成可分发单元，但它们不是同构系统。最容易出错的地方不是组件名称，而是以下四类差异：

- manifest 和 marketplace 文件位置不同
- hooks 能力模型差异很大
- marketplace 中相对路径的解析根不同
- Claude 的插件运行时路径变量不能直接假定在 Copilot 中存在

这也是为什么 Claude plugin 到 Copilot plugin 的转换不能靠简单重命名文件完成。

## 总览对比

| 主题 | GitHub Copilot CLI plugin | Claude Code plugin | 迁移影响 |
| --- | --- | --- | --- |
| 插件 manifest 默认位置 | `plugin.json` 位于插件根目录，且这是文档中的标准结构 | `.claude-plugin/plugin.json` | Claude -> Copilot 时，manifest 需要上移到插件根目录 |
| manifest 兼容发现位置 | CLI 也会查找 `.github/plugin/plugin.json` 和 `.claude-plugin/plugin.json` | 原生就是 `.claude-plugin/plugin.json` | Copilot 有兼容读取，不代表转换产物应继续保留 Claude 布局 |
| marketplace 文件位置 | 文档标准位置是 `.github/plugin/marketplace.json` | `.claude-plugin/marketplace.json` | marketplace 文件需要迁移位置 |
| 技能与 agent 目录 | 默认 `skills/`、`agents/` | 默认 `skills/`、`agents/` | 目录名相近，但 manifest 和加载规则不同 |
| hooks 默认文件 | `hooks.json` 或 `hooks/hooks.json` | `hooks/hooks.json` | Claude 的 hooks 文件通常能保留在 `hooks/hooks.json`，但内容常常不能原样复用 |
| LSP 文件 | 默认 `lsp.json` 或 `.github/lsp.json` | 默认 `.lsp.json` | 需要改名 |
| MCP 文件 | 常见为 `.mcp.json`，也支持其他文档列出的路径 | 默认 `.mcp.json` | MCP 文件位置更接近，但命令中的路径变量通常仍需重写 |
| settings 支持 | 参考文档未把插件 `settings.json` 作为标准 plugin 组件 | 支持插件根目录 `settings.json`，当前主要是 agent 设置 | Claude 的 `settings.json` 不能假定有 Copilot 等价物 |
| hooks 类型 | 参考文档中是 shell command hooks，按 `bash`/`powershell` 字段配置 | 支持 `command`、`http`、`prompt`、`agent` 四类 hook | Claude 的高级 hook 需要降级、拆分，或人工改写 |
| hooks 事件模型 | 事件较少，偏向 CLI/coding agent 生命周期 | 事件非常丰富，覆盖 subagent、task、worktree、config 变化等 | Claude hook 不一定有 Copilot 对应事件 |
| 插件命名与冲突 | skills/agents 按加载优先级和名称去重，first-found-wins | 插件技能显式命名空间化，如 `/plugin-name:hello` | Claude 的“插件前缀调用方式”不能直接映射到 Copilot |
| marketplace 相对路径语义 | `source` 相对仓库根目录 | `source` 相对 marketplace 根，也就是包含 `.claude-plugin/` 的目录 | 同样的相对路径文本，迁移后可能指向不同目录 |

## 1. 插件结构的差异

### GitHub Copilot 的标准结构

GitHub Copilot CLI 文档把插件定义为“至少包含一个根目录 `plugin.json` 的插件目录”。默认组件布局大致如下：

```text
my-plugin/
├── plugin.json
├── agents/
├── commands/
├── skills/
├── hooks/
│   └── hooks.json
├── .mcp.json
├── lsp.json
└── .github/
    └── plugin/
        └── marketplace.json
```

虽然 Copilot CLI 也兼容从 `.claude-plugin/plugin.json` 和 `.claude-plugin/marketplace.json` 读取，但这属于兼容发现机制，不是推荐输出结构。

### Claude 的标准结构

Claude Code 文档的标准布局则是：

```text
my-plugin/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── agents/
├── commands/
├── skills/
├── hooks/
│   └── hooks.json
├── .mcp.json
├── .lsp.json
└── settings.json
```

Claude 文档明确强调：只有 `plugin.json` 放在 `.claude-plugin/` 下，其他组件目录仍然在插件根目录。

### 迁移结论

- `plugin.json` 不能继续留在 `.claude-plugin/`
- `marketplace.json` 不能继续留在 `.claude-plugin/`
- `.lsp.json` 需要改成 `lsp.json`
- `skills/`、`agents/`、`commands/`、`hooks/` 这些目录通常可以保留在根目录
- Claude 的 `settings.json` 需要单独评估，通常不能直接带过去

## 2. manifest 模型的差异

### Copilot: manifest 是标准入口

Copilot CLI 参考文档把 `plugin.json` 作为插件的标准入口文件，`name` 是唯一必填字段。其余元数据如 `description`、`version`、`author`、`homepage`、`repository`、`license`、`keywords`、`category`、`tags` 都是可选的。

它还支持通过 `agents`、`skills`、`commands`、`hooks`、`mcpServers`、`lspServers` 这些字段声明自定义路径或内联对象，但如果你使用默认目录，文档建议可以省略这些字段。

### Claude: manifest 可选，且路径规则更严格

Claude 文档说明 `.claude-plugin/plugin.json` 是可选的；如果没有 manifest，Claude Code 仍可通过默认目录自动发现组件。Claude 的 manifest 也允许声明 `commands`、`agents`、`skills`、`hooks`、`mcpServers`、`outputStyles`、`lspServers` 等组件路径。

Claude 对自定义路径有一条非常重要的约束：这些路径都应相对插件根目录，并且要以 `./` 开头；并且“自定义路径是补充默认目录，而不是替代默认目录”。

### 迁移结论

- Copilot 转换产物应以根目录 `plugin.json` 为主入口
- Claude 的 `outputStyles` 没有在本次 Copilot 参考文档里找到对应 plugin 字段，不能假定可迁移
- 如果 Claude manifest 里写了很多自定义路径，迁移时要重新判断这些字段在 Copilot 中是否还需要显式保留
- 不要因为 Copilot 能兼容读取 `.claude-plugin/plugin.json`，就把它当成标准目标结构

## 3. hooks 能力模型差异最大

这是两者最容易“看起来像，实际上差很多”的地方。

### Copilot hooks: 文档里是命令型 hooks

GitHub Copilot 参考文档中的 hooks 配置是命令驱动的，核心字段包括：

- `type: "command"`
- `bash`
- `powershell`
- `cwd`
- `env`
- `timeoutSec`

参考文档中列出的事件主要包括：

- `sessionStart`
- `sessionEnd`
- `userPromptSubmitted`
- `preToolUse`
- `postToolUse`
- `errorOccurred`

其中 `preToolUse` 支持返回 `permissionDecision`，可用于 deny 工具调用；其余大部分事件更偏向日志、审计、通知、清理等场景。

### Claude hooks: 是一个更完整的自动化框架

Claude 的 hooks 远不只是“运行 shell 脚本”。它支持四类 hook handler：

- `command`
- `http`
- `prompt`
- `agent`

并且事件覆盖范围显著更广，包括：

- `SessionStart`
- `UserPromptSubmit`
- `PreToolUse`
- `PermissionRequest`
- `PostToolUse`
- `PostToolUseFailure`
- `Notification`
- `SubagentStart`
- `SubagentStop`
- `Stop`
- `TeammateIdle`
- `TaskCompleted`
- `InstructionsLoaded`
- `ConfigChange`
- `WorktreeCreate`
- `WorktreeRemove`
- `PreCompact`
- `SessionEnd`

Claude hooks 还支持：

- matcher 正则过滤
- 通过 JSON 输出进行 allow/deny/ask 或 stop/block 控制
- `updatedInput` 修改工具输入
- `additionalContext` 给模型补充上下文
- `async` 异步运行 command hooks
- 在 skills 和 agents frontmatter 中内联定义 hooks

### 迁移结论

如果 Claude plugin 里的 hooks 只是在固定时机运行脚本，那么可能能迁移。

但下面这些能力不能假定在 Copilot 里有对等支持：

- HTTP hooks
- prompt hooks
- agent hooks
- `PermissionRequest`
- `SubagentStart` / `SubagentStop`
- `TaskCompleted`
- `InstructionsLoaded`
- `ConfigChange`
- `WorktreeCreate` / `WorktreeRemove`
- `PreCompact`
- hooks 嵌入 skill/agent frontmatter

所以 Claude hooks 到 Copilot hooks 的转换通常不是“字段改名”，而是三选一：

- 降级成简单命令 hook
- 移到脚本内部自己实现控制逻辑
- 无法安全映射时保留并标记为人工处理

## 4. hooks 路径变量不兼容

### Claude 明确提供插件和项目根变量

Claude hooks 文档明确提供两类常用变量：

- `$CLAUDE_PROJECT_DIR`
- `${CLAUDE_PLUGIN_ROOT}`

此外，`SessionStart` hooks 还可以使用 `CLAUDE_ENV_FILE`，把环境变量持久化给后续 Bash 命令使用。

这三个能力在插件中非常常见，尤其是：

- hook 脚本引用
- MCP server 启动命令
- README 里的示例命令
- 指向插件内脚本、配置文件、资源目录的相对引用

### Copilot 文档没有给出对应的 Claude 风格插件根变量

在你提供的 Copilot 文档范围内，hooks 主要通过：

- `bash`
- `powershell`
- `cwd`
- 直接脚本路径

来定义执行方式，但没有文档化的 `${CLAUDE_PLUGIN_ROOT}` 对等变量，也没有 `CLAUDE_ENV_FILE` 一类的机制。

### 迁移结论

- 看到 `CLAUDE_*` 变量时，默认应视为迁移风险点
- 如果变量只是为了引用插件内一个已知文件，可以改写成确定的相对路径
- 如果逻辑依赖 `CLAUDE_ENV_FILE` 这类会话持久化能力，就不能假定 Copilot 有原生等价物
- README、脚本、命令字段、MCP/LSP 启动参数中的 `CLAUDE_*` 都要一起扫，不只是 hooks.json

## 5. marketplace 设计相似，但路径语义不同

### Copilot marketplace

Copilot marketplace 的标准文件位置是：

```text
.github/plugin/marketplace.json
```

每个 plugin entry 至少包含：

- `name`
- `source`

关键差异在于 `source` 的相对路径语义。Copilot 文档明确说明：

- 相对 `source` 是相对于仓库根目录解析的
- `./plugins/foo` 和 `plugins/foo` 都可以
- marketplace 文件虽然位于 `.github/plugin/marketplace.json`，但 `source` 不是相对这个目录解析

### Claude marketplace

Claude marketplace 的标准文件位置是：

```text
.claude-plugin/marketplace.json
```

Claude 文档说明：

- marketplace 中的相对 `source` 从 marketplace root 解析
- 这个 root 是包含 `.claude-plugin/` 的目录，也就是仓库根
- 相对路径必须以 `./` 开头
- 不允许用 `../` 从 `.claude-plugin/` 向上跳
- `metadata.pluginRoot` 可以作为相对 source 的前缀基目录

### 表面相似，实际风险点

很多 Claude marketplace 例子会写：

```json
{
  "source": "./plugins/quality-review-plugin"
}
```

这个字符串迁移到 Copilot 后常常仍然成立，但原因不同：

- Claude 是“相对 marketplace root”
- Copilot 是“相对仓库根”

在简单仓库里两者可能刚好等价；一旦你依赖了 `pluginRoot`、自定义布局，或者原来理解错了解析基准，迁移就会出错。

### strict 语义两边都存在，但用途要谨慎

Claude marketplace 支持 `strict`：

- `true` 时，`plugin.json` 是组件定义主权威
- `false` 时，marketplace entry 自身成为完整定义，若插件自身又声明组件会冲突

Copilot marketplace 也支持 `strict`，含义是默认严格校验 schema，`false` 时允许更宽松或 legacy 的插件定义。

这说明两边虽然都叫 `strict`，但你不能只凭字段名认定含义完全一致。迁移时必须以目标平台的验证规则为准。

## 6. 插件分发和安装方式不同

### Copilot

Copilot CLI 使用：

- `copilot plugin install SPECIFICATION`
- `copilot plugin marketplace add SPECIFICATION`

插件安装源可以是：

- marketplace 名称
- GitHub 仓库
- 仓库子目录
- Git URL
- 本地路径

### Claude

Claude Code 使用：

- `claude plugin install <plugin>`
- `/plugin install <plugin>`
- `claude --plugin-dir ./my-plugin` 用于本地开发测试

Claude 还区分安装 scope：

- `user`
- `project`
- `local`
- `managed`

### 迁移结论

- Claude 文档中的 `/plugin ...` 命令示例不能直接替换成 Copilot 命令，必须按 Copilot CLI 语法重写
- Claude 的 `--plugin-dir` 开发体验在 Copilot 参考文档中没有对应的同名开发模式说明，不能直接照搬描述

## 7. 组件加载和命名冲突处理不同

### Copilot: 更强调加载优先级

Copilot 文档明确写了加载顺序和冲突处理：

- agents 和 skills 采用 first-found-wins
- 项目级或个人级自定义项会优先于插件内同名项
- 插件不能覆盖内置工具和内置 agents

这意味着 Copilot plugin 更像是“额外提供组件”，不是“强制成为命名空间容器”。

### Claude: 插件技能显式命名空间化

Claude 文档明确说明 plugin skills 会以插件名作为命名空间，例如：

- 插件名 `my-first-plugin`
- skill 目录 `skills/hello/`
- 调用形式 `/my-first-plugin:hello`

这使得 Claude 的插件组件天然避免部分命名冲突。

### 迁移结论

- Claude 的命名空间式调用体验不能直接映射到 Copilot
- 把 Claude plugin 转成 Copilot plugin 时，要特别注意 skill 名称冲突和覆盖关系
- 如果原 Claude plugin 依赖“插件名前缀隔离”，Copilot 侧需要用更保守的命名来减少碰撞

## 8. Claude 有一些 Copilot 文档里没有的插件能力

在你提供的文档范围内，Claude plugin 明确支持但 Copilot plugin 参考文档没有给出等价能力或标准位置的点包括：

- `settings.json` 作为插件默认设置文件
- `outputStyles`
- hooks 作为 `http` / `prompt` / `agent`
- 在 skill 或 agent frontmatter 里直接定义 hooks
- 更细粒度的 hook 生命周期事件

这些内容在迁移时应该默认视为“需要单独确认”的项，而不是默认可迁移项。

## 9. 对实际迁移最有价值的规则

如果你要把 Claude plugin 转成 Copilot plugin，最稳妥的做法是：

1. 先迁结构，再迁能力。
2. 先把 manifest、marketplace、LSP 文件位置改对，再处理 hooks。
3. 把所有 `CLAUDE_*`、`.claude/`、`/plugin ...` 命令、`settings.json`、`outputStyles` 都视为待审查项。
4. 不要因为 Copilot 能兼容读取 `.claude-plugin/*`，就继续输出 Claude 风格目录。
5. 不要把 Claude 的 hook 事件名、输出 JSON、控制语义原样复制到 Copilot。
6. 对 marketplace 的 `source`，必须按目标平台重新判断解析根。
7. 对无法证明有 Copilot 等价能力的内容，保留原文本并标注人工复核，比“猜一个等价实现”更安全。

## 参考文档

### GitHub Copilot Core

- Customization overview: `https://docs.github.com/api/article/body?pathname=/en/copilot/how-tos/copilot-cli/customize-copilot/overview`
- Plugin reference: `https://docs.github.com/api/article/body?pathname=/en/copilot/reference/copilot-cli-reference/cli-plugin-reference`
- Hooks configuration: `https://docs.github.com/api/article/body?pathname=/en/copilot/reference/hooks-configuration`
- Using hooks with GitHub Copilot agents: `https://docs.github.com/api/article/body?pathname=/en/copilot/how-tos/use-copilot-agents/coding-agent/use-hooks`

### GitHub Copilot Marketplace

- Marketplace guide: `https://docs.github.com/api/article/body?pathname=/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-marketplace`

### Claude Source Plugin Docs

- Plugins guide: `https://code.claude.com/docs/en/plugins.md`
- Plugins reference: `https://code.claude.com/docs/en/plugins-reference.md`
- Hooks reference: `https://code.claude.com/docs/en/hooks.md`

### Claude Marketplace Docs

- Plugin marketplaces guide: `https://code.claude.com/docs/en/plugin-marketplaces.md`
