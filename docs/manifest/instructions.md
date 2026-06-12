# Instructions

The `[instructions]` section declares the agent's system prompt and rule files. For normative field constraints, see the [manifest spec](../spec/manifest.md#instructions).

---

## Fields

```toml
[instructions]
system = "instructions/system.md"

[instructions.rules.safety]
src = "instructions/rules/safety.md"
summary = "Input validation guardrails"
apply = "always"

[instructions.rules.style]
src = "instructions/rules/style.md"
description = "Code style preferences - apply when editing source files"
apply = "model-decision"

[instructions.rules.rust]
src = "instructions/rules/rust.md"
apply = "glob"
apply_to = ["*.rs"]
```

### System

| Field | Type | Description |
|---|---|---|
| `system` | `str` | Relative path to system prompt markdown file — **MUST** end in `.md` |

### Rules

Rules are named tables under `[instructions.rules.<name>]`. Each rule points to a markdown file and carries metadata controlling when the harness injects it into model context.

A rule is a markdown file containing natural language instructions that the harness injects into the model's context window before inference. The injection point varies by harness — most inject into the system prompt. The rule file **MUST** be pure content — activation metadata (`apply`, `apply_to`, `description`) lives in `theta.toml`, and `theta cast` injects harness-specific frontmatter when the target expects it.

| Field | Type | Description |
|---|---|---|
| `src` | [`LocalOrGitRef`](../spec/manifest.md#localorgitref) | **REQUIRED** — rule source |
| `summary` | `str` | Short label for display purposes (e.g. `theta list rules`) |
| `description` | `str` | Model-facing text — **REQUIRED** when `apply = "model-decision"` |
| `apply` | [`ApplyMode`](../spec/manifest.md#applymode) | Activation mode — defaults to `"always"` |
| `apply_to` | `arr[str]` | File glob patterns (`*`, `**`, `?`, `[...]`) — **SHOULD** be present when `apply = "glob"` |

Rule names **MUST** match `^[a-z0-9]+(-[a-z0-9]+)*(/[a-z0-9]+(-[a-z0-9]+)*)*$` — kebab-case segments optionally separated by `/`.

Path-qualified names (e.g. `backend/typescript`, `review/pr-review`) group related rules into logical categories. The `/` separator maps to subdirectories in the theta store (`rules/backend/typescript.md`) and in harnesses that support recursive scanning ([Copilot](../harnesses/copilot.md)). Harnesses with flat rule directories (Claude Code, Cursor) receive flattened names: `backend/typescript` --> `backend-typescript`.

System store names remain simple kebab-case (no `/`). `theta add rule backend/typescript --system typescript` registers a store rule under a path-qualified local name.

!!! info "`theta` CLI — removal"
    `theta rm rule <name> --delete` removes the rule from the manifest and deletes the source `.md` file. `theta rm system --delete` does the same for the system prompt. Omitting `--delete` removes the manifest entry only and leaves the source file in place. The `--delete` pattern is consistent across `theta rm rule|system|skill|subagent`; `theta rm tool` takes no `--delete` since tools have no source files.

### Apply modes

| Value | Meaning | When to use |
|---|---|---|
| `"always"` | Always injected into context | General rules (safety, coding standards) |
| `"model-decision"` | Model reads `description`, decides whether to apply | Contextual rules the model should selectively apply |
| `"glob"` | Applied when active file matches `apply_to` patterns | Language-specific or directory-scoped rules |
| `"manual"` | Only applied on explicit user invocation | Rarely-needed reference rules |

!!! info "Harness support for activation modes"
    `"always"` and `"glob"` are supported by all harnesses (Codex CLI approximates glob via concatenation). `"model-decision"` and `"manual"` are only natively supported by **Cursor** — casting to other harnesses is lossy (see [apply mode casting](#apply-mode-casting)).

### Validation

`theta check` validates rules:

- Rule names **MUST** match `[a-z0-9]+(-[a-z0-9]+)*(/[a-z0-9]+(-[a-z0-9]+)*)*`
- ≥3 path segments --> warning (deeply nested rules are an anti-pattern)
- `src` (when a local path string) **MUST** end in `.md`
- `apply = "model-decision"` without `description` --> warning (model has no signal to decide)
- `apply = "glob"` without `apply_to` (or empty) --> warning (rule never activates)
- `apply_to` set but `apply` is not `"glob"` --> warning (dead config)
- Rule file does not exist --> error
- Rule file is empty --> warning
- Rule file still matches scaffold template --> warning

### Writing effective rules

Recommendations synthesized from official harness documentation:

- Be **concrete and verifiable** — "use 2-space indentation" not "format code properly" ([Claude Code](https://code.claude.com/docs/en/memory), [Copilot](https://code.visualstudio.com/docs/copilot/customization/custom-instructions))
- **Include reasoning** behind rules — "use `date-fns` instead of `moment.js` because moment.js is deprecated" ([Copilot](https://code.visualstudio.com/docs/copilot/customization/custom-instructions))
- **Reference files** instead of copying their contents — prevents staleness ([Cursor](https://cursor.com/docs/rules))
- Skip rules the model already knows (common tools, obvious practices) ([Cursor](https://cursor.com/docs/rules))
- Only add rules when you notice the agent making the **same mistake repeatedly** ([Cursor](https://cursor.com/docs/rules))
- Prefer **bullet points and markdown structure** over long paragraphs ([Claude Code](https://code.claude.com/docs/en/memory))

### Conventional directory

```
project/
├── theta.toml
├── instructions/
│   ├── system.md              <-- [instructions].system
│   └── rules/
│       ├── safety.md          <-- [instructions.rules.safety]
│       ├── style.md           <-- [instructions.rules.style]
│       └── backend/
│           └── typescript.md  <-- [instructions.rules."backend/typescript"]
```

Default directories are configurable via `theta-settings`:

- `THETA_INSTRUCTIONS_DIR` (default: `instructions`)
- `THETA_RULES_DIR` (default: `rules`)

Resolution order: CLI flag > env var > built-in default.

`THETA_OUT_DIR` redirects where `theta sync` writes `.theta/` and `theta.lock`. It does not affect where source files are read from. See [conformance](../protocol/conformance.md#environment-variables).

These settings are also documented in the [protocol guide](../protocol/index.md).

## Casting

| Field | Claude Code | Codex CLI | Cursor | Copilot |
|---|---|---|---|---|
| `system` | `CLAUDE.md` | `AGENTS.md` | `.cursor/rules/system.md` | `.github/copilot-instructions.md` |
| Rules | `.claude/rules/<name>.md` | Concatenated into `AGENTS.md` | `.cursor/rules/<name>.mdc` | `.github/instructions/<name>.instructions.md` |

!!! note "Path-qualified names and flat harnesses"
    Path-qualified rule names (`backend/typescript`) cast to subdirectories in Copilot (`.github/instructions/backend/typescript.instructions.md`). Flat harnesses receive a flattened filename: `.claude/rules/backend-typescript.md`, `.cursor/rules/backend-typescript.mdc`.

!!! note "Claude Code: file rename"
    Claude Code reads `CLAUDE.md`, not `AGENTS.md`. Casting **MUST** rename or generate the file. See [Claude Code harness reference](../harnesses/claude-code.md).

### Apply mode casting

How each `apply` value maps to harness-native frontmatter:

| `apply` | Cursor (`.mdc`) | Copilot | Claude Code | Codex CLI |
|---|---|---|---|---|
| `"always"` | `alwaysApply: true` | No frontmatter | No frontmatter | Concatenated |
| `"model-decision"` | `alwaysApply: false` + `description` | Lossy (warn) | Lossy (warn) | Lossy (warn) |
| `"glob"` | `alwaysApply: false` + `globs` | `applyTo` | `paths` | Lossy (warn) |
| `"manual"` | `alwaysApply: false` | Lossy (warn) | Lossy (warn) | Lossy (warn) |

### Rules frontmatter by harness

Native frontmatter keys each caster emits for rules:

| Harness | Frontmatter keys |
|---|---|
| Cursor | `description`, `globs`, `alwaysApply` (YAML in `.mdc`) |
| Copilot | `applyTo` (YAML in `.instructions.md`) |
| Claude Code | `paths` (YAML in `.md` under `.claude/rules/`) |
| Codex CLI | None — rules concatenated into `AGENTS.md` |

### Output size constraints

Hard and soft limits imposed by harnesses on cast output:

| Harness | Constraint | Type | Value | Source |
|---|---|---|---|---|
| Codex CLI | Combined `AGENTS.md` size | Hard | 32 KiB | [developers.openai.com](https://developers.openai.com/codex/guides/agents-md) |
| Cursor | Per-rule line count | Soft | 500 lines | [cursor.com/docs/rules](https://cursor.com/docs/rules) |
| Claude Code | Per-file line count | Soft | 200 lines | [code.claude.com](https://code.claude.com/docs/en/memory) |
| Copilot | — | None | — | — |

`theta cast` emits warnings (`Diagnostic::warn`) for hard limits and hints (`Diagnostic::hint`) for soft limits.

## Metadata: theta.toml vs frontmatter

Rule configuration metadata (description, globs, activation mode) lives in theta.toml, not in any file frontmatter. The rule markdown file is pure content. `theta cast` injects frontmatter when the target harness expects it (e.g., **Cursor** `.mdc`).

The guiding principle: if a resource has **config metadata + free-text content**, the config metadata belongs in theta.toml.

## Global instructions

User-level instructions (e.g., `~/.claude/CLAUDE.md`, user rules in Cursor settings) are a runtime concern owned by each harness. They are not project-scoped and not part of `theta.toml`.
