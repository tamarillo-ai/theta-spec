# Claude Code

!!! info "Harness compatibility"
    **Last verified**: 2026-04-13  
    **Docs**: [code.claude.com/docs/en](https://code.claude.com/docs/en)  
    Stale? [File an issue](https://github.com/tamarillo-ai/theta-spec/issues/new?labels=stale-harness&title=stale:+claude-code).

## Config surface

3 config scopes: user (`~/.claude/settings.json`), project (`.claude/settings.json`), local (`.claude/settings.local.json`). Project-level is the cast target.

Sources: [Settings](https://code.claude.com/docs/en/settings) · [Hooks](https://code.claude.com/docs/en/hooks) · [Permissions](https://code.claude.com/docs/en/permissions) · [Env vars](https://code.claude.com/docs/en/env-vars) · [Sub-agents](https://code.claude.com/docs/en/sub-agents) · [Skills](https://code.claude.com/docs/en/skills) · [Memory](https://code.claude.com/docs/en/memory)

## Output files

| theta operation | Output | Format |
|---|---|---|
| Cast agent config | `.claude/settings.json` | JSON |
| Cast instructions | `CLAUDE.md` | Markdown |
| Cast rules | `.claude/rules/*.md` | Markdown |
| Cast MCP servers | `.mcp.json` | JSON |
| Cast subagents | `.claude/agents/<name>.md` | YAML frontmatter + Markdown |
| Cast harness config | `.claude/settings.json` (merged) | JSON |

!!! info "Skills"
    Skills are cast as full directories (SKILL.md + supporting files). Both `theta cast to claude-code` and `theta cast from claude-code` handle `.claude/skills/`.

## Casting notes

- Instructions cast to `CLAUDE.md`, not `AGENTS.md`. Casting MUST rename.
- Rules cast to `.claude/rules/*.md`. Also appended to `CLAUDE.md` when a single-file output is needed.
- No native agent description field. Agent identity is embedded in the `CLAUDE.md` header.

## Harness-specific config


| `[harness.claude_code]` field | Cast target | Notes |
|---|---|---|
| `sandbox` | `settings.json --> sandbox` | `enabled`, `filesystem` (`allowWrite`/`denyRead`/`denyWrite`/`allowRead`), `network` (`allowedDomains`, `allowUnixSockets`) |
| `hooks` | `settings.json --> hooks` | 26 event types — 4 handler types: `command`, `http`, `prompt`, `agent` |
| `permissions` | `settings.json --> permissions` | `allow`/`ask`/`deny` arrays — format: `Tool(specifier)` — deny --> ask --> allow evaluation order |
| `auto_mode` | `settings.json --> autoMode` | `environment`, `allow`, `soft_deny` arrays |
| `view_mode` | `settings.json --> viewMode` | `"default"` \| `"verbose"` \| `"focus"` |
| `attribution` | `settings.json --> attribution` | Git commit/PR attribution text |
| `worktree` | `settings.json --> worktree` | `symlinkDirectories`, `sparsePaths` |
| `enabled_plugins` | `settings.json --> enabledPlugins` | Plugin enable/disable map |
| `version` | — | Semver range constraint — cast warns if installed version is outside range |

~60 additional `settings.json` keys pass through without validation.

## Known limitations

- No configurable compaction — always auto.
- `ignore` relies on `.gitignore` — no dedicated context ignore mechanism.
- No native agent description field.
- Global config (`~/.claude.json`) is user-scoped and not project-castable.

!!! info "`theta` CLI"
    `theta cast to claude-code --notes` and `theta cast from claude-code --notes` print the full list of known cast limitations.
