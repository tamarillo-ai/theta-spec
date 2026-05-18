# Skills

The `[skills]` section declares composable capability packages following the [Agent Skills spec](https://agentskills.io/specification). For normative field constraints, see the [manifest spec](../spec/manifest.md#skillsname).

---

## Fields

```toml
[skills.code-review]
source = { path = "skills/code-review" }
tags = ["code-quality", "review"]
goal = "review code changes for correctness, style, and security issues"

[skills.web-design]
source = { git = "https://github.com/vercel-labs/agent-skills", subdirectory = "skills/web-design-guidelines" }
tags = ["frontend", "design"]

[skills.deploy]
source = { system = "deploy" }
```

| Field | Type | Description |
|---|---|---|
| `source` | [`SourceRef`](../spec/manifest.md#sourceref) | **REQUIRED** — where the skill comes from |
| `tags` | array[string] | Kebab-case labels for discovery and categorization — each <= 64 chars |
| `goal` | string | Machine-facing purpose statement — <= 512 chars |

Skill names **MUST** match `^[a-z0-9]+(-[a-z0-9]+)*$`, <= 64 chars, no consecutive hyphens (`--`).

### Source types

| Type | Syntax | Resolution |
|---|---|---|
| `path` | `{ path = "skills/name" }` | Relative to manifest |
| `git` | `{ git = "https://...", branch = "...", subdirectory = "..." }` | Cached at `~/.cache/theta/git/` |

Git sources accept one optional ref field: `branch`, `tag`, or `rev` (at most one). See [`SourceRef`](../spec/manifest.md#sourceref) for details.
| `system` | `{ system = "name" }` | [System store](../protocol/sources.md#system) |

After materialization, skills live in `.theta/skills/<name>/`.

!!! note "`theta` implementation notes, `git` source resolution"
    Git-sourced skills are fetched and pinned during `theta lock`. The resolved commit SHA is recorded in `theta.lock`. `theta sync` clones into the local git cache and `theta cast` reads from the materialized `.theta/skills/<name>/` directory. See [source resolution](../protocol/sources.md) for details.

---

## Skill directory structure

A skill is a directory with a `SKILL.md` at its root:

```
skill-name/
  SKILL.md          # YAML frontmatter + markdown body
  scripts/          # optional - executable code
  references/       # optional - docs the agent reads on demand
  assets/           # optional - templates, data files
```

The `SKILL.md` frontmatter format is defined by the [Agent Skills spec](https://agentskills.io/specification#frontmatter).

---

## Validation

`theta check` validates skills. Checks are grouped by source type.

**All sources:**

- Skill name
    - **MUST** match `^[a-z0-9]+(-[a-z0-9]+)*$`
    - **MUST** be <= 64 chars

**Path sources:**

- `source.path` **MUST NOT** be empty
- `source.path` **MUST NOT** reference `.theta/` — use the source directory, not materialized output

**Git sources:**

- `source.git` **MUST** have a recognized URL scheme (`https://`, `http://`, `git://`, `ssh://`)

**System sources:**

- `source.system` name **MUST** be a valid skill name (same pattern)

---

## Casting

The SKILL.md content is read from `.theta/skills/<name>/SKILL.md` (materialized) and written to the harness's specific location.

| Harness | Output path | Format |
|---|---|---|
| [Claude Code](../harnesses/claude-code.md) | `.claude/skills/<name>/SKILL.md` | [Agent Skills](https://agentskills.io/specification) |
| [Codex CLI](../harnesses/codex-cli.md) | `.agents/skills/<name>/SKILL.md` (default) or `.codex/skills/<name>/SKILL.md` (opt-in via `[harness.codex].codex_specific_skills = true`) | [Agent Skills](https://agentskills.io/specification) |
| [Cursor](../harnesses/cursor.md) | `.cursor/skills/<name>/SKILL.md` | [Plugin skills](https://cursor.com/docs/skills) |
| [Copilot](../harnesses/copilot.md) | `.github/skills/<name>/SKILL.md` | [Agent Skills](https://agentskills.io/specification) |

Codex CLI natively discovers `.agents/skills/` ([ref](https://developers.openai.com/codex/skills)), so it is the canonical cast output. Set `[harness.codex].codex_specific_skills = true` to write into `.codex/skills/` instead.

### Import

When importing (`theta cast from`), skill directories found in the harness's native location are registered as local path sources. Harnesses with cross-agent skill directories (`.agents/skills/`) are also scanned — harness-native location takes precedence on name collisions.

---

!!! info "`theta` implementation notes"
    `theta add skill <name>` scaffolds the skill directory with `SKILL.md` + optional subdirectories. `theta add skill --path <dir>` registers an existing directory. `theta add skill --git <url>` adds a git-sourced skill resolved during `theta sync`. `theta add skill --system` references a skill from the system store. `theta register skill <name>` copies a local skill into the system store for reuse across projects. `theta rm skill <name> --delete` removes the skill from the manifest and deletes the skill directory,omit `--delete` to keep the source on disk.

    **GitHub shorthand** — `theta add skill owner/repo[/subdirectory][@ref]` expands to a git source with `https://github.com/owner/repo`. The `@ref` part maps to `rev` since the shorthand cannot distinguish branches from tags. Examples:

    - `theta add skill vercel-labs/agent-skills/skills/web-design-guidelines@main`
    - `theta add skill tamarillo/skills/osint`

    `theta register skill` supports the same shorthand syntax.
