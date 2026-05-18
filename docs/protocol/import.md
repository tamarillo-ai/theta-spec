# Import

---

## Cast from

```
cast from(target, directory) --> theta.toml + source files
```

Given a harness name and a directory containing its native config, import produces a best-effort `theta.toml` and extracted source files. See the [protocol spec](../spec/protocol.md#cast-from) for normative requirements.

!!! info "`theta` CLI"
    `theta cast from <harness>` — symmetric with `theta cast to <harness>`. Use `--input` to override the source directory and `--force` to overwrite an existing `theta.toml`. `--cross-read` also imports files from other harness locations that the source harness can discover at runtime. `--notes` prints known import limitations for the target harness, then exits.

## Pipeline

- Check for existing `theta.toml` — error unless `--force`
- Read harness-specific config files from the project directory
- If `--cross-read` is set, also read files from other harness locations the source harness discovers (e.g. `AGENTS.md`, `CLAUDE.md`, `.claude/rules/`, `.claude/agents/`, `.codex/agents/`)
- Produce `theta.toml` + extracted source files
- Emit diagnostics for anything that could not be cleanly mapped
- Emit hints for each cross-read file warning about duplication risk
- Conflict-check extracted files, then write

## Identity extraction

Some harnesses store identity, description, and system prompt in a single markdown file (`CLAUDE.md`, `AGENTS.md`). Import attempts to parse this structure:

```markdown
# agent name

agent description paragraph.

(remaining body becomes the system prompt)
```

| Parsed element | Maps to | Fallback when missing |
|---|---|---|
| `# heading` | `[agent].name` | Directory name |
| First paragraph after heading | `[agent].description` | `"imported from <harness>"` |
| Remaining body | System prompt source file | Empty |

This is best-effort heuristic parsing on freeform markdown. Implementations **MAY** emit diagnostics when the expected structure is not found, but silent fallback to defaults is acceptable.

## Information preservation

Import **MUST NOT** silently discard information:

| Source config | Destination |
|---|---|
| Maps to a common theta-spec field | Common section (`[agent]`, `[tools.*]`, etc.) |
| Harness-specific, no common equivalent | `[harness.<name>]` |

## Extracted files

Import writes source files alongside `theta.toml`:

| Extracted file | Content |
|---|---|
| `system.md` | System prompt body |
| `rules/<name>.md` | Per-rule content |
| `skills/<name>/SKILL.md` | Skill packages |

The generated `theta.toml` references these files via `path` sources.

## Import sources

Each harness reads a different set of files. See the [harness reference pages](../harnesses/index.md) for the full list per harness.

## Merge behavior

If a `theta.toml` already exists, `theta cast from` **MUST** warn and refuse to overwrite unless `--force` is passed.
