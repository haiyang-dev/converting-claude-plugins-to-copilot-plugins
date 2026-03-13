---
name: converting-claude-plugins-to-copilot-plugins
description: "Use when: converting a Claude Code plugin into a GitHub Copilot CLI plugin, especially when plugin manifests, skills directories, marketplace files, and path-sensitive references must be translated safely."
---

# Converting Claude Plugins To Copilot Plugins

## Overview

Convert a Claude Code plugin directory into a GitHub Copilot CLI plugin directory.

The output is a sibling directory named by appending `-copilot` to the source directory name.

**Primary goal:** preserve the plugin structure while translating Claude-specific file locations into Copilot CLI plugin locations safely.

Use the standard Copilot plugin layout with `plugin.json` at the target root. Copilot can also discover manifests in `.github/plugin/plugin.json` or `.claude-plugin/plugin.json`, but this skill emits the root manifest because that is the documented default plugin structure.

## When to Use

Use this skill when:

- A user gives you a Claude Code plugin folder and wants a GitHub Copilot CLI plugin as output.
- The source plugin uses Claude plugin conventions such as an optional `.claude-plugin/plugin.json`, an optional `.claude-plugin/marketplace.json`, and root-level component directories such as `skills/`, `agents/`, or `hooks/`.
- Path handling is the risky part: manifest locations, marketplace files, mixed separators, relative references, or Windows path normalization.
- The user wants the final output in a sibling directory named `<source>-copilot`.

Do not use this skill when:

- The user only wants a project-level Copilot skill under `.github/skills/`.
- The source directory does not look like a Claude plugin.
- The task is unrelated to plugin conversion.

## Procedure

1. Ask the user for the Claude Code plugin root folder.
2. Normalize the source path and derive the sibling `-copilot` output path.
3. Validate the Claude plugin structure before writing anything.
4. Create a Copilot plugin root with a root-level `plugin.json` manifest, translating source metadata when present and deriving only the minimum required metadata when it is not.
5. Copy and translate plugin components using the official Copilot plugin file locations.
6. Rewrite path-sensitive references conservatively.
7. Generate a conversion report that lists copied, translated, skipped, and ambiguous items.

## Claude Source Plugin Structure

### Claude Input Structure

Per the Claude plugin documentation, expect a source tree like this:

```text
my-plugin/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── agents/
├── commands/
├── hooks/
│   └── hooks.json
├── skills/
│   └── <skill>/
│       └── SKILL.md
├── .mcp.json
├── .lsp.json
└── settings.json
```

The `.claude-plugin/` directory contains metadata such as an optional `plugin.json` and, optionally, `marketplace.json`. Component directories such as `skills/`, `agents/`, `commands/`, and `hooks/` still belong at the plugin root.

Do not require the Claude manifest as the sole indicator that the source is a Claude plugin. A source tree can still be a valid Claude-style plugin package if it lacks `.claude-plugin/plugin.json` but contains recognizable plugin components in the documented root locations.

## GitHub Copilot Plugin Structure

### Copilot Output Structure

Per the GitHub Copilot CLI plugin docs, produce a plugin tree like this:

```text
my-plugin-copilot/
├── plugin.json
├── agents/
├── commands/
├── hooks/
│   └── hooks.json
├── skills/
│   └── <skill>/
│       └── SKILL.md
├── .mcp.json
├── lsp.json
├── README.md            # optional copied docs
├── LICENSE              # optional copied docs
└── .github/
    └── plugin/
        └── marketplace.json   # optional
```

The required Copilot plugin manifest is `plugin.json` at the plugin root.

`marketplace.json` is not required for a valid Copilot plugin. It is only needed when the user wants marketplace distribution metadata or when the source Claude plugin already includes marketplace metadata that should be translated.

Use `hooks/hooks.json` as the default file-based hooks location. Only emit a root `hooks.json` if the source already uses a root hooks file or the user explicitly asks for the flat layout.

## Shared Path Rules

### Normalize Input First

When the user gives you a source path:

1. Remove wrapping single or double quotes.
2. Expand `.` and `..`.
3. Convert the path to an absolute path.
4. Normalize separators for internal computation.
5. On Windows, compare normalized paths case-insensitively.

When writing rewritten documentation paths back out, prefer `/` separators for consistency.

Keep two explicit variables in your reasoning:

- `SOURCE_ROOT`: normalized Claude plugin root
- `TARGET_ROOT`: `dirname(SOURCE_ROOT) / (basename(SOURCE_ROOT) + "-copilot")`

For every file you copy, derive the target path from that file's normalized path relative to `SOURCE_ROOT`. Do not build target paths from ad hoc string concatenation or partially normalized fragments.

### Script, Command, And Documentation Environment Variable Rules

Handle script paths, command strings, and copied documentation separately from generic prose rewrites.

Claude hooks explicitly document path-sensitive environment variables such as:

- `$CLAUDE_PROJECT_DIR` for project-root-relative scripts
- `${CLAUDE_PLUGIN_ROOT}` for plugin-root-relative scripts bundled with a plugin
- `CLAUDE_ENV_FILE` for `SessionStart` hooks that persist environment variables for later Bash commands

GitHub Copilot hooks configuration, in the referenced hooks docs, documents per-platform script fields such as `bash` and `powershell`, plus an optional hook `cwd`. It does not document Claude-equivalent plugin-root or project-root environment variables for path resolution.

Do not assume this issue is limited to hooks. Any copied text or executable surface in the plugin may embed Claude-specific environment-variable paths, including:

- hook commands
- command implementations
- agent or skill assets that shell out to bundled scripts
- helper scripts under `scripts/`
- MCP or LSP launcher commands
- `README.md`, `CHANGELOG.md`, setup guides, troubleshooting docs, and embedded command examples

If a copied file or manifest field contains `CLAUDE_*` environment-variable references, treat that as a path-sensitive conversion point even if it is not inside `hooks/hooks.json`.

Because of that mismatch, use these rules:

1. If a Claude command string or referenced script path uses `${CLAUDE_PLUGIN_ROOT}` or `$CLAUDE_PROJECT_DIR` only to locate a bundled file that will also be copied into the Copilot plugin, rewrite it to a concrete target-relative path instead of inventing a new environment variable.
2. Prefer keeping target script paths relative and stable, using documented Copilot hook `cwd` only when that makes the resulting path simpler and still unambiguous on both Bash and PowerShell.
3. If Claude logic relies on `CLAUDE_ENV_FILE`, do not translate it blindly. Copilot hooks configuration does not document an equivalent persistence file in the referenced docs, so preserve the logic only if it can be rewritten without that variable; otherwise report it as manual follow-up.
4. If a copied script, manifest field, or command string uses other `CLAUDE_*` variables for path resolution, treat them as Claude-specific unless the Copilot docs explicitly document the same variable. Do not guess.

Apply these concrete rewrites when they are structurally valid:

- `${CLAUDE_PLUGIN_ROOT}/scripts/format.sh` -> a copied target script path such as `./scripts/format.sh` in the Copilot hook definition, or `format.sh` with hook `cwd: "scripts"`
- `"$CLAUDE_PROJECT_DIR"/.claude/hooks/check-style.sh` -> a copied target script path inside the Copilot plugin, not a preserved Claude variable reference

When converting hook definitions, script-bearing configuration, or copied text files:

- Normalize every referenced script path before rewriting it.
- Resolve the referenced source file against the correct Claude base: plugin root for `${CLAUDE_PLUGIN_ROOT}`, project root for `$CLAUDE_PROJECT_DIR`, and the source hook file location for ordinary relative paths.
- Copy the referenced script into the target only if the resolved file stays inside `SOURCE_ROOT`.
- If the referenced script resolves outside `SOURCE_ROOT`, stop and report it.
- Preserve quoting for paths that may contain spaces.
- For Copilot hooks, emit both `bash` and `powershell` entries only when you can point each one at a real copied script. Do not fabricate a PowerShell variant from a Bash-only source script.

When scanning copied documentation and Markdown:

- Inspect copied text files for `CLAUDE_*`, `.claude/`, and Claude-only install or runtime examples.
- Rewrite environment-variable-backed paths only when the target location is concrete and documented.
- Rewrite ordinary documentation examples such as `${CLAUDE_PLUGIN_ROOT}/scripts/...` to the concrete target-relative path that will exist after conversion.
- Preserve ambiguous examples or prose if you cannot prove the intended runtime context.
- If the text documents Claude-only operational behavior rather than a path, preserve it only when it remains useful; otherwise report it as Claude-specific residue for manual cleanup.

If a Claude script or command mixes path variables with unrelated shell logic and you cannot isolate a safe path rewrite, preserve the command text in the report and flag it for manual review.

### Reject Unsafe Paths

Fail immediately if any of these are true:

- `SOURCE_ROOT` does not exist.
- `SOURCE_ROOT` is not a directory.
- `TARGET_ROOT` already exists.
- Any resolved source or target path escapes its root after normalization.

Treat the source as invalid only if it has neither a Claude manifest nor any recognizable Claude plugin components in documented locations such as `skills/`, `agents/`, `commands/`, `hooks/hooks.json`, `.mcp.json`, `.lsp.json`, or `.claude-plugin/marketplace.json`.

If `skills/` exists, every skill directory under it must contain `SKILL.md`.

If symlinks exist, resolve them carefully. If the resolved content stays within `SOURCE_ROOT`, copy the resolved content into the target tree. If a symlink resolves outside `SOURCE_ROOT`, stop and report it instead of copying external content into the plugin.

Do not guess. Stop and explain the exact failing path or condition.

### Mapping Rules

Use these target translations as the Copilot output contract for version 1:

| Claude source | Copilot output |
|---|---|
| `.claude-plugin/plugin.json` | `plugin.json` |
| `.claude-plugin/marketplace.json` | `.github/plugin/marketplace.json` |
| `skills/` | `skills/` |
| `agents/` | `agents/` |
| `commands/` | `commands/` |
| `hooks/hooks.json` | `hooks/hooks.json` |
| `.mcp.json` | `.mcp.json` |
| `.lsp.json` | `lsp.json` |
| `settings.json` | unsupported by default; report and skip unless you have a documented Copilot equivalent |
| `outputStyles` manifest field or related assets | unsupported by default; report and skip unless the target Copilot docs explicitly document an equivalent |

For other top-level files and directories:

- Copy general plugin documentation and assets such as `README.md`, `LICENSE`, `CHANGELOG.md`, and `scripts/` when they are part of the plugin package or are referenced by copied components.
- Skip unrelated platform packaging directories such as `.cursor-plugin/`, `.codex/`, and `.opencode/`.
- Preserve relative layout for copied non-component assets.

### Manifest Translation

If `.claude-plugin/plugin.json` exists, translate the Claude manifest into a Copilot `plugin.json` at the target root.

If the source has no Claude manifest, still emit a Copilot `plugin.json` because it is the standard Copilot plugin entry point. In that case:

- derive `name` conservatively from the source directory name if no better documented source exists
- copy only metadata you can prove from source files
- do not invent optional fields

Preserve metadata fields when available and valid:

- `name`
- `description`
- `version`
- `author`
- `homepage`
- `repository`
- `license`
- `keywords`

Set component path fields explicitly only when needed:

- `skills`: `skills/`
- `agents`: `agents/`
- `commands`: `commands/`
- `hooks`: `hooks/hooks.json` or `hooks.json`
- `mcpServers`: `.mcp.json`
- `lspServers`: `lsp.json`

If the target uses only Copilot default locations, omit these component path fields entirely. Do not add redundant custom paths unless they are needed for non-default layout.

Do not copy the Claude manifest location itself. The Copilot manifest belongs at the target root.

Do not copy Claude-only manifest fields such as `outputStyles` unless the target Copilot documentation explicitly documents an equivalent field.

## Claude Marketplace Semantics

When reading a source marketplace:

- Treat `.claude-plugin/marketplace.json` as marketplace metadata, not as part of the plugin's runtime component tree.
- Treat relative `source` paths as marketplace-root-relative Claude paths.
- Reject or warn on entries that depend on `..` traversal or Claude-only behavior you cannot map safely.

## GitHub Copilot Marketplace Semantics

### Marketplace Translation

If the source includes `.claude-plugin/marketplace.json`, translate it to `.github/plugin/marketplace.json` because that is the standard documented Copilot marketplace location. Copilot can also discover marketplace manifests in `.claude-plugin/marketplace.json`, but this skill emits only `.github/plugin/marketplace.json` to keep the output unambiguous.

If the user explicitly wants marketplace output and the source does not include a marketplace file, generate a minimal Copilot marketplace manifest from the translated plugin metadata. Otherwise do not create a marketplace file.

Treat the marketplace as a separate catalog from the plugin itself. A plugin repository does not need a marketplace file unless the user wants marketplace distribution.

When translating marketplace entries:

- Preserve `name` and `owner`.
- Preserve `metadata.description` and `metadata.version` when present.
- Preserve `metadata.pluginRoot` only if it still matches the translated repository layout.
- Preserve plugin entry metadata such as `description`, `version`, `author`, `homepage`, `repository`, `license`, `keywords`, `category`, and `tags` when valid.
- Preserve plugin entry component override fields only when they still point to valid Copilot plugin content.

Rewrite each plugin entry `source` so it remains correct for Copilot marketplace semantics:

- For repository-local relative sources, make the path relative to the repository root, because Copilot resolves marketplace relative paths from the repository root.
- `./plugins/my-plugin` and `plugins/my-plugin` are both valid in Copilot; prefer preserving `./` when translating from Claude to minimize churn.
- Do not emit `../` in marketplace relative paths.
- When the plugin lives at the repository root, use `.` or `./` consistently.
- If the Claude marketplace uses external source objects such as GitHub, URL, git-subdir, npm, or pip, preserve them unless a field is Claude-specific and undocumented in Copilot.

If the source marketplace file depends on Claude-only semantics you cannot map safely, preserve the entry and flag it in the conversion report.

Do not map Claude `strict` semantics directly to Copilot `strict`. In Claude marketplace docs, `strict: false` changes whether the marketplace entry or `plugin.json` is authoritative for component definitions. In the referenced Copilot docs, `strict` controls validation strictness instead. Because the field name matches but the semantics do not, use these rules:

- do not preserve Claude `strict: false` just because it appeared in the source marketplace
- keep Copilot's default strict behavior unless you have a documented Copilot-specific reason to emit `strict: false`
- if the Claude marketplace depended on `strict: false` authority semantics, report that semantic mismatch explicitly in the conversion report

## Rewrite Rules

Do not do blind global search-and-replace. Interpret the reference type first.

### Keep These Unchanged

- Relative references that stay valid after relocation because the local relationship is preserved
- External URLs
- Ambiguous prose that only looks path-like

Example:

- Source file: `skills/a/SKILL.md`
- Source reference: `./helper.md`
- Target file: `skills/a/SKILL.md`
- Target reference: still `./helper.md`

### Rewrite These

- References to `.claude-plugin/plugin.json` -> `plugin.json`
- References to `.claude-plugin/marketplace.json` -> `.github/plugin/marketplace.json`
- References to Claude plugin install commands -> Copilot plugin install commands when the mapping is direct
- References to Claude-specific environment-variable paths -> concrete target-relative paths when the destination file is known

Examples:

- `claude --plugin-dir ./my-plugin` -> `copilot plugin install ./my-plugin-copilot`
- `/plugin install superpowers@claude-plugins-official` -> `copilot plugin install SPECIFICATION` only when you know the real Copilot specification; otherwise preserve and warn
- `${CLAUDE_PLUGIN_ROOT}/scripts/format.sh` in copied docs -> `./scripts/format.sh` only when that file will exist at that relative location in the Copilot output

### Warn Instead Of Guessing

If a relative reference escapes the source plugin tree, preserve the text and add it to the conversion report for manual review.

If a manifest field or command has no documented Copilot equivalent, preserve the original text and report it.

If copied docs or examples rely on Claude plugin namespacing such as `/plugin-name:skill`, do not invent a Copilot command or invocation form unless the target interaction is documented and provable. Preserve and report ambiguous examples.

If a path-like string could be either prose or a real path, preserve it and warn.

## Unsupported Or Conditional Items

Copilot CLI plugins support agents, skills, hooks, MCP, LSP, and commands, but not every Claude plugin file maps one-to-one.

- Convert `skills/`, `agents/`, `commands/`, hooks config, `.mcp.json`, and `.lsp.json` when present
- Translate `.claude-plugin/plugin.json` into root `plugin.json` when present, or derive a minimal root `plugin.json` when the source uses only default Claude component locations
- Translate `.claude-plugin/marketplace.json` into `.github/plugin/marketplace.json` when present
- Generate `.github/plugin/marketplace.json` only when explicitly requested and absent from the source
- Keep marketplace `source` entries repository-root-relative and avoid `..`
- Rewrite Claude script-path environment variables to concrete target paths only when the target file location is provable; otherwise preserve and report
- Report `settings.json` as unsupported unless the destination behavior is explicitly documented
- Report Claude-only hook types such as `http`, `prompt`, and `agent`, and Claude-only hook events without documented Copilot equivalents, unless you can safely reduce them to supported Copilot command-hook behavior
- Report Claude marketplace `strict` authority semantics and Claude manifest `outputStyles` as unsupported semantic mismatches unless the target Copilot docs explicitly document an equivalent

## Conversion Report

Generate a human-readable report in the target directory, for example `conversion-report.md`.

The report should include:

- Source path
- Target path
- Manifest translations performed
- Component directories copied
- Marketplace file translations performed
- Path references rewritten
- Unsupported files skipped
- Ambiguous references preserved for manual review

## Common Failure Modes

- Treating Claude root components as if they lived inside `.claude-plugin/`
- Rejecting a valid Claude plugin package only because `.claude-plugin/plugin.json` is absent
- Converting to `.github/skills/` when the user asked for a Copilot plugin
- Leaving the Copilot manifest in `.claude-plugin/` instead of the target root
- Forgetting to rename `.lsp.json` to `lsp.json`
- Adding custom manifest component paths even though the target uses default Copilot locations
- Rewriting marketplace `source` paths relative to the wrong root
- Treating the marketplace file as mandatory for every plugin instead of optional distribution metadata
- Treating Claude `strict` as if it had the same meaning in Copilot marketplace entries
- Silently dropping Claude-only fields such as `settings.json` or `outputStyles` instead of reporting them
- Rewriting text that is not actually a path
- Following `..` segments or symlinks outside the allowed root

## Bottom Line

Build a real Copilot plugin directory. Put `plugin.json` at the target root. Omit redundant component path fields when you use Copilot defaults. Default file-based hooks to `hooks/hooks.json`. Generate marketplace metadata only when it already exists in the source or the user asks for it. Rewrite only what you can prove.

## References

### GitHub Copilot Core

- Customization overview: `https://docs.github.com/api/article/body?pathname=/en/copilot/how-tos/copilot-cli/customize-copilot/overview`
- Plugin reference: `https://docs.github.com/api/article/body?pathname=/en/copilot/reference/copilot-cli-reference/cli-plugin-reference`
- Hooks configuration: `https://docs.github.com/api/article/body?pathname=/en/copilot/reference/hooks-configuration` and `https://docs.github.com/api/article/body?pathname=/en/copilot/how-tos/use-copilot-agents/coding-agent/use-hooks`

### GitHub Copilot Marketplace

- Marketplace guide: `https://docs.github.com/api/article/body?pathname=/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-marketplace`

### Claude Source Plugin Docs

- Plugins guide: `https://code.claude.com/docs/en/plugins.md`
- Plugins reference: `https://code.claude.com/docs/en/plugins-reference.md`
- Hooks reference: `https://code.claude.com/docs/en/hooks.md`

### Claude Marketplace Docs

- Plugin marketplaces guide: `https://code.claude.com/docs/en/plugin-marketplaces.md`