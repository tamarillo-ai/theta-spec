# Agent

The `[agent]` section. For normative field constraints, see the [manifest spec](../spec/manifest.md#agent).

## Fields

```toml
[agent]
name = "osint-investigator"
description = "OSINT people intelligence agent"
version = "0.3.1"
authors = ["ivan <ivan@tamarillo.ai>"]
model = "claude-sonnet-4-20250514"
tags = ["osint", "people", "research"]
```

| Field | Type | Description |
|---|---|---|
| `name` | string | REQUIRED — MUST match `^[a-z0-9]+(-[a-z0-9]+)*$` |
| `description` | string | REQUIRED — <= 1024 chars |
| `version` | string | [Semver](https://semver.org/) `major.minor.patch` only — no pre-release, no build metadata |
| `authors` | array[string] | `"name"` or `"name <email>"` format — email MUST contain `@` |
| `model` | string | Model this agent was designed for — metadata, not a runtime directive |
| `tags` | array[string] | Kebab-case labels for discovery and categorization — each <= 64 chars |

`version`, `authors`, and `model` are theta-spec metadata for identification. `model` signals which model the author had in mind when building this agent — harnesses select their own model at runtime.

Runtime model settings (reasoning effort, multi-model, provider format) belong in `[harness.<name>]`:

```toml
[harness.codex]
model_reasoning_effort = "high"
```

---

!!! info "`theta` implementation notes"
    The [theta CLI](https://theta.tamarillo.ai/) derives `name` from the directory name, auto-detects `authors` from `git config`, and scaffolds `version` as `"0.1.0"`. `model` is not scaffolded — set it if you want to signal which model the agent was designed for. See [theta CLI docs](https://theta.tamarillo.ai/) for details.

    **Name normalization fallback** — when the source directory name normalizes to an empty string (e.g. the directory is `---` or contains no ASCII alphanumerics), the CLI scaffolds `name = "my-agent"`. Override with `theta init --name <name>` when scaffolding from such a directory.

### Casting

`name` and `description` are embedded as markdown in each harness's system prompt file — `name` becomes a `# heading`, `description` becomes the first paragraph. `version`, `authors`, and `model` are not cast.

| Harness | System prompt file |
|---|---|
| Claude Code | `CLAUDE.md` |
| Codex CLI | `AGENTS.md` |
| Cursor | `.cursor/rules/system.md` |
| GitHub Copilot | `.github/copilot-instructions.md` |

For ref subagents, the child agent's own `model` is used in subagent frontmatter (see [subagents](subagents.md)).
