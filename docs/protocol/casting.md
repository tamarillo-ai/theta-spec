# Casting

---

## Cast to

```
cast to(manifest, .theta/, target) --> {files}
```

Given a manifest, a materialized `.theta/` directory, and a target harness name, casting produces the set of native config files that harness expects. See the [protocol spec](../spec/protocol.md#cast-to) for normative requirements.

!!! info "`theta` CLI"
    `theta cast to <harness>` with `--output` to override the output directory and `--force` to overwrite conflicts.

### Pipeline

Implementations **SHOULD** ensure `.theta/` is up to date before casting (implicit sync).

- **Config validation** — per-harness checks on `[harness.<name>]` (version constraints, unsupported fields)
- **File generation** — deterministic conversion of manifest + materialized resources into harness-specific files
- **Output validation** — harness-specific size and structural checks

!!! info "`theta` CLI"
    `theta cast to` runs an implicit `theta sync` before casting. Output files are staged to a temporary directory and renamed into place (atomic write).


## Supported targets

| Target | `[harness.*]` key | Output files |
|---|---|---|
| [Claude Code](../harnesses/claude-code.md) | `claude_code` | `CLAUDE.md`, `.claude/rules/*.md`, `.claude/settings.json`, `.mcp.json`, `.claude/skills/*/SKILL.md`, `.claude/agents/*.md` |
| [Copilot](../harnesses/copilot.md) | `github_copilot` | `.github/copilot-instructions.md`, `.github/instructions/*.instructions.md`, `.vscode/mcp.json`, `.github/skills/*/SKILL.md`, `.github/agents/*.agent.md`, `.vscode/settings.json` |
| [Cursor](../harnesses/cursor.md) | `cursor` | `.cursor/rules/system.md`, `.cursor/rules/*.mdc`, `.cursor/mcp.json`, `.cursor/hooks.json`, `.cursor/skills/*/SKILL.md` |
| [Codex CLI](../harnesses/codex-cli.md) | `codex` | `AGENTS.md`, `.codex/config.toml`, `.codex/agents/*.toml`, `.agents/skills/*/SKILL.md` (default; `.codex/skills/*/SKILL.md` when `[harness.codex].codex_specific_skills = true`) |

Harness config formats change independently of this spec — see [`harnesses.toml`](https://github.com/tamarillo-ai/theta-spec/blob/main/harnesses.toml) for the current registry. Per-harness file details are on each [harness reference page](../harnesses/index.md).

## Lossy casting

Not every harness supports every feature. Casting **MAY** be lossy — but **MUST NOT** be silently lossy.

When a field cannot be represented in the target, implementations **MUST** emit a warning:

```
warning: casting to cursor: dropped context.compaction (cursor manages compaction internally)
warning: casting to codex: dropped context.compaction_reserved (codex has no configurable compaction)
warning: casting to github_copilot: mapped permissions.default to tool allowlist (copilot uses allowlist model)
```

Lossy casting **MUST NOT** cause failure. When the target has less granularity, casting **SHOULD** err toward the more restrictive interpretation.

## Output validation

Implementations **SHOULD** validate cast output against known harness constraints:

- Hard limits (harness will reject or truncate) produce warnings
- Soft limits (harness recommends but does not enforce) produce hints

Known constraints are documented per-harness in the [instructions guide](../manifest/instructions.md#output-size-constraints) and [harness reference](../harnesses/index.md).

## Harness version validation

Implementations **SHOULD** validate that the installed harness version satisfies any constraint declared in `[harness.<name>].version`:

| Condition | Diagnostic level |
|---|---|
| No `version` key declared | Hint — "consider pinning a version constraint" |
| Version declared, detection fails | Hint — "installed but undetectable" |
| Version detected, constraint not satisfied | Warn |

!!! info "`theta` CLI"
    Version detection methods: `claude --version`, `codex --version`, `cursor --version`, `~/.cursor/version`. Copilot version is read from `~/.vscode/extensions/github.copilot-*/package.json`.

## Cast notes

Each harness has known limitations, lossy transforms, and format normalization behaviors that may affect round-trip fidelity. Implementations **SHOULD** make these discoverable.

!!! info "`theta` CLI"
    `theta cast to <harness> --notes` and `theta cast from <harness> --notes` print known limitations and clarifying notes for the target harness, then exit. See each [harness reference page](../harnesses/index.md) for a summary.

## Multi-harness config

A single manifest **MAY** contain `[harness.<name>]` sections for multiple targets. When casting to harness X:

- Read common sections (`[agent]`, `[tools.*]`, etc.)
- Read `[harness.X]` for target-specific overrides
- Ignore all other `[harness.*]` sections
- Produce output files
