# Converting Claude Plugins To Copilot Plugins

This directory contains a GitHub Copilot custom skill for converting a Claude Code plugin into a GitHub Copilot CLI plugin.

The skill is designed for path-sensitive migrations. It focuses on the parts that usually break during translation:

- manifest location changes
- marketplace file relocation
- hooks layout differences
- relative path rewrites
- Windows path normalization
- Claude-specific environment variables that appear in scripts, commands, and documentation

## What This Skill Is

This is a Copilot skill, not a standalone converter binary.

It gives GitHub Copilot a structured procedure for taking a Claude plugin folder as input and producing a sibling output folder with the `-copilot` suffix.

Example:

```text
C:\work\superpowers
->
C:\work\superpowers-copilot
```

## What It Does

The skill tells Copilot to:

1. validate the Claude plugin root
2. derive a sibling target directory with the `-copilot` suffix
3. translate `.claude-plugin/plugin.json` to root `plugin.json`
4. translate `.claude-plugin/marketplace.json` to `.github/plugin/marketplace.json` when needed
5. preserve supported component directories such as `skills/`, `agents/`, `commands/`, and hooks
6. rewrite path-sensitive references conservatively
7. generate a conversion report listing copied, translated, skipped, and ambiguous items

## Why This Skill Exists

Claude Code plugins and GitHub Copilot CLI plugins are similar enough to invite naive search-and-replace, but different enough that this usually produces broken results.

The hardest failures are rarely the obvious ones. They usually come from:

- paths resolved from different roots
- marketplace `source` semantics changing between systems
- hooks script references that rely on Claude runtime variables
- documentation that still contains Claude-only commands or `.claude/` paths
- Windows separator and case-normalization edge cases

This skill exists to make those conversions deterministic and reviewable.

## Supported Conversion Scope

The skill covers these source-to-target mappings:

| Claude source | Copilot target |
| --- | --- |
| `.claude-plugin/plugin.json` | `plugin.json` |
| `.claude-plugin/marketplace.json` | `.github/plugin/marketplace.json` |
| `skills/` | `skills/` |
| `agents/` | `agents/` |
| `commands/` | `commands/` |
| `hooks/hooks.json` | `hooks/hooks.json` |
| `.mcp.json` | `.mcp.json` |
| `.lsp.json` | `lsp.json` |

The skill also tells Copilot to inspect copied text files and scripts for Claude-specific residue such as:

- `CLAUDE_*` environment variables
- `.claude/` paths
- Claude-only install commands
- Claude marketplace assumptions

## Path And Environment Variable Handling

This skill treats path handling as the primary risk area.

It requires Copilot to normalize input paths before doing any conversion work:

- strip wrapping quotes
- resolve `.` and `..`
- convert to an absolute path
- normalize separators for internal computation
- compare case-insensitively on Windows

It also treats Claude-specific environment variables as conversion hazards, not as safe portable abstractions.

Examples include:

- `$CLAUDE_PROJECT_DIR`
- `${CLAUDE_PLUGIN_ROOT}`
- `CLAUDE_ENV_FILE`

The rule is simple:

- if the target path can be proven, rewrite to a concrete Copilot-relative path
- if the target path cannot be proven, preserve and report it for manual review
- do not invent undocumented Copilot equivalents

This applies to:

- hooks
- command definitions
- helper scripts
- MCP or LSP launch commands
- copied Markdown and README content

## What The Skill Does Not Do

This skill does not claim that every Claude plugin can be converted automatically without review.

In particular, it does not:

- guess undocumented Copilot behavior
- overwrite an existing target directory
- preserve unsafe `..` traversal
- silently copy symlink targets outside the plugin root
- fabricate PowerShell variants for Bash-only scripts
- treat Claude-only runtime variables as if Copilot documented the same runtime

If a source plugin depends on Claude-only behavior that cannot be mapped safely, the skill instructs Copilot to preserve the original text where appropriate and report the issue instead of guessing.

## Expected Output

The expected result is a real Copilot plugin directory, not a repo-local `.github/skills/` customization.

By default, the output uses:

- root `plugin.json`
- `hooks/hooks.json` for file-based hooks
- `.github/plugin/marketplace.json` only when marketplace metadata exists or is explicitly requested

The skill also expects a human-readable `conversion-report.md` in the target directory.

## How To Use

Place this skill in a repository under:

```text
.github/skills/converting-claude-plugins-to-copilot-plugins/
```

Then ask GitHub Copilot for a conversion such as:

```text
Convert this Claude Code plugin into a GitHub Copilot CLI plugin:
C:\path\to\my-plugin
```

The skill is intended to guide Copilot through the conversion process, including validation and conservative rewrites.

## Recommended Review Checklist

After conversion, review at least these areas:

1. `plugin.json` is at the target root
2. `lsp.json` was renamed correctly from `.lsp.json`
3. hooks script paths still point at real copied files
4. copied docs do not contain stale Claude-only commands or `CLAUDE_*` paths
5. marketplace `source` values are correct for Copilot repository-root semantics
6. the conversion report contains no unresolved items you care about

## Source References

This skill is intentionally based on the official Claude and GitHub Copilot documentation.

GitHub Copilot:

- Customization overview: `https://docs.github.com/api/article/body?pathname=/en/copilot/how-tos/copilot-cli/customize-copilot/overview`
- CLI plugin reference: `https://docs.github.com/api/article/body?pathname=/en/copilot/reference/copilot-cli-reference/cli-plugin-reference`
- Hooks configuration: `https://docs.github.com/api/article/body?pathname=/en/copilot/reference/hooks-configuration`
- Marketplace guide: `https://docs.github.com/api/article/body?pathname=/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-marketplace`

Claude Code:

- Plugins guide: `https://code.claude.com/docs/en/plugins.md`
- Plugins reference: `https://code.claude.com/docs/en/plugins-reference.md`
- Hooks reference: `https://code.claude.com/docs/en/hooks.md`
- Plugin marketplaces: `https://code.claude.com/docs/en/plugin-marketplaces.md`

## License And Reuse

If you publish this skill as an open-source project, keep the README aligned with the actual behavior of `SKILL.md`.

The important contract is not marketing language. It is the conversion behavior:

- target is a Copilot CLI plugin
- output path is a sibling `-copilot` directory
- path rewrites must be provable
- ambiguous Claude-specific behavior must be reported, not guessed