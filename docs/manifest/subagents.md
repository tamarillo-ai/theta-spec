# Subagents

`[[subagents]]` declares child agents that the parent can delegate to.

## Fields

| Field | Required | Type | Description |
|---|---|---|---|
| `name` | Yes | string | Kebab-case identifier (`^[a-z0-9][a-z0-9-]*$`) |
| `description` | Yes (non-ref) | string | When to delegate to this subagent |
| `ref` | No | string | Path to a child `theta.toml` — mutually exclusive with `prompt_path` |
| `prompt_path` | No | string | Relative path to a `.md` file containing the system prompt — mutually exclusive with `ref` |
| `model` | No | string | Model identifier; inherits parent if omitted |
| `tools` | No | string[] | Tool allow-list; inherits parent if omitted |
| `skills` | No | string[] | Skills to inject at startup |

## Modes

Each `[[subagents]]` entry MUST be exactly one of:

| Mode | Indicator | Semantics |
|---|---|---|
| **Ref** | `ref` is set | Full child agent with its own manifest, identity, rules, skills, nested subagents |
| **Inline** | `prompt_path` is set | System prompt in a local `.md` file; parent manifest owns `model`, `tools`, `skills` |
| **Description-only** | Neither `ref` nor `prompt_path` | Subagent runs with `description` alone |

`ref` and `prompt_path` MUST NOT both be set.

## Constraints

- `prompt_path` MUST end in `.md`
- `prompt_path` MUST NOT point under `.theta/`
- `ref` MUST NOT point under `.theta/`
- `ref` SHOULD end in `.toml`
- When `ref` is set, `model`, `tools`, and `skills` on the parent entry are ignored (the child manifest owns them)

## Examples

### Ref subagent

```toml
[[subagents]]
name = "analyst"
ref = "agents/analyst/theta.toml"
```

### Inline subagent with prompt

```toml
[[subagents]]
name = "scraper"
description = "scrapes github repos for harness config patterns"
prompt_path = "subagents/scraper.md"
model = "claude-sonnet-4-20250514"
tools = ["playwright"]
```

### Description-only subagent

```toml
[[subagents]]
name = "helper"
description = "a general-purpose helper that runs with the parent's configuration"
```

## Materialization

Both modes materialize to `.theta/subagents/<name>/system.md`. Ref subagents also materialize the child manifest and its resources.

```
.theta/subagents/
├── analyst/                    # ref subagent
│   ├── theta.toml
│   ├── system.md
│   └── rules/
└── scraper/                    # inline subagent
    └── system.md
```

## Casting

Casters read `.theta/subagents/<name>/system.md` regardless of mode. The resolution chain:

- Ref subagent's resolved `system_prompt` (from child manifest)
- Materialized `system.md`
- Fallback to description

## Importing

`theta cast from <harness>` externalizes subagent prompt bodies to `<project>/subagents/<name>.md` and writes `prompt_path` in the generated `theta.toml`. `--subagent-prompts <DIR>` overrides the default directory (also configurable via `THETA_SUBAGENTS_DIR`). `--force-overwrite` allows replacing existing files with different content.

---

## Definition styles

=== "Reference"

    A pointer to another `theta.toml`. Independently runnable.

    ```toml
    [[subagents]]
    name = "researcher"
    description = "deep research agent"
    ref = "./agents/researcher.theta.toml"
    ```

=== "Inline (prompt_path)"

    System prompt externalized to a `.md` file. Parent manifest owns the remaining fields.

    ```toml
    [[subagents]]
    name = "code-reviewer"
    description = "reviews code for style and correctness"
    model = "claude-sonnet-4-20250514"
    prompt_path = "subagents/code-review.md"
    tools = ["read", "grep", "glob"]
    skills = ["code-quality"]
    ```

=== "Description-only"

    No prompt file, no ref. Runs with `description` alone.

    ```toml
    [[subagents]]
    name = "helper"
    description = "a general-purpose helper"
    ```

---

## Validation

`theta check` validates subagents. Checks are grouped by mode.

**All subagents:**

- `name` **MUST NOT** be empty
- Duplicate names **MUST NOT** appear in the `[[subagents]]` array
- `tools` and `skills` lists **MUST NOT** contain empty strings

**Ref subagents:**

- `ref` path **MUST NOT** reference `.theta/`
- `ref` path **SHOULD** end in `.toml`
- `model`, `tools`, and `skills` on the parent entry produce warnings if present — the child manifest owns them

**Inline subagents (`prompt_path`):**

- `prompt_path` **MUST** end in `.md`
- `prompt_path` **MUST NOT** reference `.theta/`
- `description` **SHOULD** be present

**Description-only subagents:**

- `description` **SHOULD** be present

---

## Casting per harness

| Harness | Output path | Format |
|---|---|---|
| [Claude Code](../harnesses/claude-code.md) | `.claude/agents/<name>.md` | YAML frontmatter + markdown |
| [Codex CLI](../harnesses/codex-cli.md) | `.codex/agents/<name>.toml` | TOML |
| [Cursor](../harnesses/cursor.md) | `.cursor/agents/<name>.md` | YAML frontmatter + markdown |
| [GitHub Copilot](../harnesses/copilot.md) | `.github/agents/<name>.agent.md` | YAML frontmatter + markdown |

Frontmatter fields vary by harness:

| Field | Claude Code | Codex CLI | Cursor | Copilot |
|---|---|---|---|---|
| `description` | Yes | Yes | Yes | Yes |
| `model` | Yes | Yes | Yes | Yes |
| `tools` | Yes | — | — | Yes |
| `skills` | Yes | — | — | — |

!!! note "Claude `maxTurns`"
    Claude Code supports a `maxTurns` field in subagent frontmatter to cap agentic turns. This is harness-specific — set it via `[harness.claude_code.subagent.<name>].maxTurns` in `theta.toml`. The field round-trips through Claude's `.claude/agents/<name>.md` frontmatter via harness extras.

---

## Graph resolution

Ref subagents form a dependency graph — each `ref` points to a child `theta.toml` that **MAY** declare its own subagents. Implementations **MUST** resolve this graph recursively during locking and materialization.

- **Cycle detection** — implementations **MUST** detect cycles (A -> B -> A) and silently skip already-visited nodes
- **Name collisions** — if two subagents across the graph share a name, the first-encountered entry wins. Implementations **SHOULD** warn on collisions
- **Depth** — no depth limit is imposed by the spec

!!! info "`theta` implementation notes"
    `theta tree` displays the subagent dependency graph. `theta lock` resolves ref subagents recursively, locking each child's instructions and skills into `theta.lock`. `theta add subagent <name> --description "..."` scaffolds `subagents/<name>.md` + registers with `prompt_path`. `--prompt-path <file>` registers an existing `.md` file. `--agent-ref <path>` registers a ref subagent. `--description-only` registers with no prompt file. `theta rm subagent <name> --delete` removes from the manifest and deletes the source file (ref or prompt_path).

    `theta register agent <name>` copies the current project into the system store so it can be referenced via `[[subagents]] ref` from other projects. `--no-lock` skips the post-register `theta lock` step — useful when the consumer wants to amend more entries before re-locking.

    Cursor-specific subagent fields (`readonly`, `is_background`) are stored in `[harness.cursor.subagent.<name>]` and round-trip through Cursor's `.cursor/agents/<name>.md` frontmatter.
