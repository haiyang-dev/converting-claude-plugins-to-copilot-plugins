# Core Differences Between GitHub Copilot Plugins and Claude Code Plugins

This document summarizes the key differences between GitHub Copilot CLI plugins and Claude Code plugins based on the official documentation. The goal is not general product comparison. It is to help you decide which parts can be mapped directly, which parts must be rewritten, and which parts require manual review.

## One-Sentence Conclusion

The two systems use a similar plugin concept: both package skills, agents, hooks, MCP, and LSP capabilities into a distributable unit. But they are not structurally equivalent. The most error-prone differences are these four:

- manifest and marketplace file locations differ
- the hooks capability model differs substantially
- marketplace relative paths are resolved from different roots
- Claude runtime path variables cannot be assumed to exist in Copilot

That is why converting a Claude plugin to a Copilot plugin cannot be done safely with simple file renaming.

## High-Level Comparison

| Topic | GitHub Copilot CLI plugin | Claude Code plugin | Migration impact |
| --- | --- | --- | --- |
| Default manifest location | `plugin.json` at the plugin root, and this is the documented standard structure | `.claude-plugin/plugin.json` | Claude -> Copilot requires moving the manifest to the plugin root |
| Compatible manifest discovery locations | CLI can also discover `.github/plugin/plugin.json` and `.claude-plugin/plugin.json` | Native layout uses `.claude-plugin/plugin.json` | Copilot compatibility does not mean converted output should keep Claude layout |
| Marketplace file location | Standard documented location is `.github/plugin/marketplace.json` | `.claude-plugin/marketplace.json` | Marketplace metadata must be relocated |
| Skills and agents directories | Default `skills/`, `agents/` | Default `skills/`, `agents/` | Directory names are similar, but manifest and loading rules differ |
| Default hooks file | `hooks.json` or `hooks/hooks.json` | `hooks/hooks.json` | Claude hooks files often stay in `hooks/hooks.json`, but their contents usually cannot be reused unchanged |
| LSP file | Default `lsp.json` or `.github/lsp.json` | Default `.lsp.json` | Requires renaming |
| MCP file | Commonly `.mcp.json`, with other documented locations also supported | Default `.mcp.json` | File location is closer, but command paths often still need rewriting |
| Settings support | The referenced docs do not describe plugin `settings.json` as a standard plugin component | Supports plugin-root `settings.json`, currently mainly for agent settings | Claude `settings.json` should not be assumed to have a Copilot equivalent |
| Hook types | Referenced docs describe shell command hooks configured with `bash` and `powershell` fields | Supports `command`, `http`, `prompt`, and `agent` hooks | Advanced Claude hooks usually require downgrade, restructuring, or manual rewrite |
| Hook event model | Fewer events, centered on CLI and coding agent lifecycle | Much broader event set covering subagents, tasks, worktrees, config changes, and more | Many Claude hook events have no direct Copilot equivalent |
| Naming and collision behavior | Skills and agents are deduplicated by precedence and name, first-found-wins | Plugin skills are explicitly namespaced, for example `/plugin-name:hello` | Claude plugin-prefixed invocation does not map directly to Copilot |
| Marketplace relative path semantics | `source` is relative to repository root | `source` is relative to marketplace root, meaning the directory that contains `.claude-plugin/` | The same relative path string may point somewhere different after migration |

## 1. Plugin Structure Differences

### GitHub Copilot Standard Structure

GitHub Copilot CLI documents a plugin as a plugin directory that contains, at minimum, a root-level `plugin.json`. The default layout looks like this:

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

Copilot CLI can also discover `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`, but that is a compatibility behavior, not the recommended output structure.

### Claude Standard Structure

Claude Code documents this standard layout:

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

Claude documentation explicitly notes that only `plugin.json` belongs inside `.claude-plugin/`. Other component directories remain at plugin root.

### Migration Takeaway

- `plugin.json` should not remain in `.claude-plugin/`
- `marketplace.json` should not remain in `.claude-plugin/`
- `.lsp.json` should become `lsp.json`
- `skills/`, `agents/`, `commands/`, and `hooks/` can usually remain at plugin root
- Claude `settings.json` needs separate review and usually should not be carried over directly

## 2. Manifest Model Differences

### Copilot: Manifest Is the Standard Entry Point

In Copilot CLI docs, `plugin.json` is the standard plugin entry point, and `name` is the only required field. Other metadata such as `description`, `version`, `author`, `homepage`, `repository`, `license`, `keywords`, `category`, and `tags` are optional.

Copilot also supports `agents`, `skills`, `commands`, `hooks`, `mcpServers`, and `lspServers` fields for custom paths or inline objects, but when you use default locations these fields can usually be omitted.

### Claude: Manifest Is Optional and Path Rules Are Stricter

Claude docs state that `.claude-plugin/plugin.json` is optional. Without a manifest, Claude Code can still auto-discover components from default locations. Claude manifest files may also declare `commands`, `agents`, `skills`, `hooks`, `mcpServers`, `outputStyles`, and `lspServers`.

Claude also applies an important path rule: custom paths are relative to plugin root, must start with `./`, and supplement default directories rather than replacing them.

### Migration Takeaway

- Copilot output should use root-level `plugin.json` as the primary entry point
- Claude `outputStyles` has no documented equivalent in the referenced Copilot plugin docs and should not be assumed portable
- If the Claude manifest uses many custom paths, you must re-evaluate whether those fields still need to exist explicitly in Copilot
- Copilot compatibility with `.claude-plugin/plugin.json` should not be treated as the target structure

## 3. Hooks Are the Biggest Capability Mismatch

This is where the two systems look similar at a glance but differ materially.

### Copilot Hooks: Command-Oriented in the Referenced Docs

The GitHub Copilot hooks docs in scope describe command-driven hooks with fields such as:

- `type: "command"`
- `bash`
- `powershell`
- `cwd`
- `env`
- `timeoutSec`

The main documented events are:

- `sessionStart`
- `sessionEnd`
- `userPromptSubmitted`
- `preToolUse`
- `postToolUse`
- `errorOccurred`

Among these, `preToolUse` can return `permissionDecision` to deny tool execution. Most of the others are better suited to logging, auditing, notification, or cleanup.

### Claude Hooks: A Broader Automation Framework

Claude hooks are not limited to shell commands. They support four handler types:

- `command`
- `http`
- `prompt`
- `agent`

The event surface is also much broader, including:

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

Claude hooks also support:

- regex matchers
- JSON-based allow, deny, ask, stop, and block control
- `updatedInput` to modify tool input
- `additionalContext` to feed context back to the model
- `async` command hooks
- hooks embedded in skill and agent frontmatter

### Migration Takeaway

If a Claude plugin only uses simple scripts triggered at fixed lifecycle points, migration may be possible.

But the following should not be assumed to have direct Copilot equivalents:

- HTTP hooks
- prompt hooks
- agent hooks
- `PermissionRequest`
- `SubagentStart` and `SubagentStop`
- `TaskCompleted`
- `InstructionsLoaded`
- `ConfigChange`
- `WorktreeCreate` and `WorktreeRemove`
- `PreCompact`
- hooks embedded in skill or agent frontmatter

In practice, Claude hooks usually need one of three treatments when converting to Copilot:

- downgrade to a simpler command hook
- move control logic into scripts
- preserve and flag for manual review if no safe mapping exists

## 4. Hook Path Variables Are Not Compatible

### Claude Explicitly Documents Plugin and Project Root Variables

Claude hooks docs explicitly provide these variables:

- `$CLAUDE_PROJECT_DIR`
- `${CLAUDE_PLUGIN_ROOT}`

In addition, `SessionStart` hooks can use `CLAUDE_ENV_FILE` to persist environment variables for later Bash commands.

These capabilities appear frequently in plugin code, especially in:

- hook script references
- MCP server startup commands
- README examples
- references to plugin-local scripts, config files, and resource directories

### Copilot Docs Do Not Document Claude-Style Plugin Root Variables

Within the Copilot docs you provided, hooks are primarily defined through:

- `bash`
- `powershell`
- `cwd`
- direct script paths

There is no documented equivalent to `${CLAUDE_PLUGIN_ROOT}` and no documented mechanism like `CLAUDE_ENV_FILE`.

### Migration Takeaway

- any `CLAUDE_*` variable should be treated as a migration risk point by default
- if the variable only points to a known bundled file, rewrite it to a concrete relative path
- if the logic depends on session persistence via `CLAUDE_ENV_FILE`, do not assume Copilot has a native equivalent
- scan `CLAUDE_*` across README files, scripts, command fields, and MCP or LSP launch config, not just `hooks.json`

## 5. Marketplace Design Is Similar, but Path Semantics Differ

### Copilot Marketplace

The standard Copilot marketplace location is:

```text
.github/plugin/marketplace.json
```

Each plugin entry includes at least:

- `name`
- `source`

The critical difference is how relative `source` values are resolved. Copilot docs explicitly state that:

- relative `source` paths are resolved from repository root
- both `./plugins/foo` and `plugins/foo` are valid
- even though the marketplace file lives at `.github/plugin/marketplace.json`, `source` is not resolved relative to that directory

### Claude Marketplace

The standard Claude marketplace location is:

```text
.claude-plugin/marketplace.json
```

Claude docs state that:

- relative `source` values are resolved from marketplace root
- that root is the directory that contains `.claude-plugin/`, which is effectively repository root in the common layout
- relative paths must start with `./`
- `../` is not allowed for escaping upward from `.claude-plugin/`
- `metadata.pluginRoot` can act as a base prefix for relative source paths

### Similar on the Surface, Risky in Practice

Many Claude marketplace examples use:

```json
{
  "source": "./plugins/quality-review-plugin"
}
```

That same string often still works after migration to Copilot, but for a different reason:

- Claude resolves it from marketplace root
- Copilot resolves it from repository root

In a simple repository these may coincide. Once `pluginRoot`, custom layout, or incorrect assumptions about the base path are involved, migration becomes error-prone.

### `strict` Exists in Both, but Means Different Things

Claude marketplace supports `strict` with this behavior:

- `true`: `plugin.json` is the authoritative source of component definitions
- `false`: the marketplace entry becomes the complete definition, and conflicts occur if the plugin also declares components

Copilot marketplace also supports `strict`, but in the referenced docs it controls validation strictness. `false` means relaxed validation for legacy or direct-install scenarios.

So although both systems use the same field name, you cannot assume the same semantics. Migration must follow the target platform's documented meaning.

## 6. Distribution and Installation Differ

### Copilot

Copilot CLI uses:

- `copilot plugin install SPECIFICATION`
- `copilot plugin marketplace add SPECIFICATION`

Install sources can include:

- marketplace entries
- GitHub repositories
- repository subdirectories
- Git URLs
- local paths

### Claude

Claude Code uses:

- `claude plugin install <plugin>`
- `/plugin install <plugin>`
- `claude --plugin-dir ./my-plugin` for local development and testing

Claude also distinguishes installation scopes:

- `user`
- `project`
- `local`
- `managed`

### Migration Takeaway

- `/plugin ...` examples in Claude docs cannot be copied directly to Copilot and must be rewritten into valid Copilot CLI syntax
- the Claude `--plugin-dir` development flow is not documented with the same name in the Copilot references you provided and should not be copied blindly

## 7. Loading and Naming Collision Behavior Differ

### Copilot Emphasizes Precedence Rules

Copilot docs explicitly describe loading order and conflict handling:

- agents and skills use first-found-wins precedence
- project-level or personal customizations can take priority over plugin-provided components
- plugins cannot override built-in tools or built-in agents

That makes Copilot plugins behave more like an additional source of components than a forced namespace container.

### Claude Explicitly Namespaces Plugin Skills

Claude docs explicitly namespace plugin skills with the plugin name. For example:

- plugin name `my-first-plugin`
- skill directory `skills/hello/`
- invocation form `/my-first-plugin:hello`

This avoids some naming collisions by design.

### Migration Takeaway

- Claude's namespaced invocation model does not map directly to Copilot
- when converting Claude plugins, pay attention to skill-name collisions and override behavior
- if the original Claude plugin relied on plugin-name-prefixed isolation, use more conservative naming on the Copilot side to reduce collisions

## 8. Claude Has Plugin Capabilities Not Shown in the Copilot Docs in Scope

Within the documentation set you provided, Claude plugins clearly support features that the referenced Copilot plugin docs do not document as equivalent capabilities or standard locations, including:

- `settings.json` as plugin default settings
- `outputStyles`
- hooks of type `http`, `prompt`, and `agent`
- hooks defined directly in skill or agent frontmatter
- a much more fine-grained hook lifecycle

These should default to “needs separate confirmation” during migration rather than “safe to convert.”

## 9. The Most Useful Rules for Real Migration Work

If you are converting a Claude plugin into a Copilot plugin, the safest approach is:

1. Migrate structure first, then migrate capabilities.
2. Fix manifest, marketplace, and LSP file locations before dealing with hooks.
3. Treat all `CLAUDE_*`, `.claude/`, `/plugin ...` commands, `settings.json`, and `outputStyles` as review items.
4. Do not keep Claude-style output layout just because Copilot can compatibly discover `.claude-plugin/*`.
5. Do not copy Claude hook event names, JSON output formats, or control semantics into Copilot unchanged.
6. Re-evaluate marketplace `source` paths from the target platform's path root semantics.
7. If you cannot prove a Copilot equivalent exists, preserve the original text and mark it for manual review rather than guessing.

## References

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
