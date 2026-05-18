# Cursor

!!! info "Harness compatibility"
    **Minimum version**: 3.0  
    **Last verified**: 2026-04-30  
    **Docs**: [cursor.com/docs](https://cursor.com/docs)  
    Stale? [File an issue](https://github.com/tamarillo-ai/theta-spec/issues/new?labels=stale-harness&title=stale:+cursor).

## Config surface

| Surface | File | Format | theta section | Docs |
|---|---|---|---|---|
| System prompt | `.cursor/rules/system.md` | Markdown | `[agent]` + `[instructions].system` | [Rules](https://cursor.com/docs/rules) |
| Rules | `.cursor/rules/*.{md,mdc}` | mdc frontmatter (`description`, `globs`, `alwaysApply`) | `[instructions.rules]` | [Rules](https://cursor.com/docs/rules) |
| MCP servers | `.cursor/mcp.json` (`"mcpServers"` key) | JSON | `[tools]` | [MCP](https://cursor.com/docs/mcp) |
| Subagents | `.cursor/agents/*.md` | YAML frontmatter + Markdown | `[[subagents]]` | [Subagents](https://cursor.com/docs/subagents) |
| Skills | `.cursor/skills/*/SKILL.md` | YAML frontmatter + Markdown | `[skills]` | [Skills](https://cursor.com/docs/skills) |
| Skills (cross-agent) | `.agents/skills/*/SKILL.md` | Same as above | `[skills]` | [Skills](https://cursor.com/docs/skills) |
| Hooks | `.cursor/hooks.json` | JSON | `[harness.cursor].hooks` | [Hooks](https://cursor.com/docs/hooks) |
| Plugins | `.cursor-plugin/plugin.json` | JSON | *Not modeled* | [Plugins](https://cursor.com/docs/plugins) |
| Permissions | `~/.cursor/permissions.json` | JSON | *User-level, not modeled* | [Permissions](https://cursor.com/docs/reference/permissions) |
| CLI config | `.cursor/cli.json` | JSON | *Not modeled* | [CLI](https://cursor.com/docs/cli/reference/permissions) |
| Ignore files | `.cursorignore`, `.cursorindexingignore` | gitignore syntax | *Not modeled* | [Ignore](https://cursor.com/docs/reference/ignore-file) |
| Commands | `.cursor/commands/*.md` | Markdown | *Deprecated — replaced by skills* | [Skills](https://cursor.com/docs/skills#migrating-rules-and-commands-to-skills) |

!!! note "Alternative file locations"
    Cursor also discovers agents and skills at `.claude/agents/`, `.codex/agents/`, `.agents/skills/`, `.claude/skills/`, `.codex/skills/`, and `~/` global variants. `theta cast from cursor` reads `.cursor/` and `.agents/` paths by default.

!!! info "`theta` implementation notes — cross-read"
    `theta cast from cursor --cross-read` also imports files from other harness locations that Cursor discovers at runtime:

    - `AGENTS.md` — concatenated into the system prompt
    - `.claude/agents/*.md` — imported as subagents (same frontmatter format as `.cursor/agents/`)
    - `.codex/agents/*.toml` — imported as subagents (`name`, `description`, `developer_instructions`, `model`)

    Native `.cursor/agents/` files take precedence on name collisions. Each cross-read file emits a diagnostic hint.

## Output files

| theta operation | Output | Format |
|---|---|---|
| Cast instructions | `.cursor/rules/system.md` | mdc frontmatter + Markdown |
| Cast rules | `.cursor/rules/<name>.mdc` | mdc frontmatter + Markdown |
| Cast MCP servers | `.cursor/mcp.json` | JSON (`{"mcpServers": {...}}`) |
| Cast subagents | `.cursor/agents/<name>.md` | YAML frontmatter + Markdown |
| Cast skills | `.cursor/skills/<name>/SKILL.md` (+ `scripts/`, `references/`, `assets/`) | Markdown |
| Cast hooks | `.cursor/hooks.json` | JSON (`{"version": 1, "hooks": {...}}`) |

## Casting notes

- Rule frontmatter is a line-based `key: rest-of-line` format, not YAML. theta uses a dedicated mdc parser for import and emits raw lines on cast. Agent frontmatter IS valid YAML. [ref](https://cursor.com/docs/rules#rule-anatomy)
- `globs` values are split on commas. [ref](https://cursor.com/docs/rules#glob-pattern-examples) Brace expansion (`**/*.{ts,tsx}`) is broken in cursor itself — commas inside braces are consumed by the splitter. theta matches this behavior.
- MCP uses `"mcpServers"` root key (not `"servers"`). [ref](https://cursor.com/docs/mcp)
- Hooks use a single file, not a directory. The entire JSON blob round-trips through `[harness.cursor].hooks`. [ref](https://cursor.com/docs/hooks)
- Subagent frontmatter has exactly 5 fields: `name`, `description`, `model`, `readonly`, `is_background`. [ref](https://cursor.com/docs/subagents#configuration-fields) `name` is derived from the filename — theta does not emit it on cast.

### Known round-trip normalizations

- **mdc frontmatter key order** — key order **MAY** change
- **Glob format** — JSON array `["*.ts", "*.tsx"]` normalizes to comma-separated `*.ts, *.tsx`
- **Empty fields stripped** — `description: ` (trailing space / YAML null) is dropped
- **JSON key order** — `hooks.json` and `mcp.json` keys **MAY** reorder alphabetically
- **Trailing newline** — cast output always ends with `\n` (POSIX)
- **JSONC comments stripped** — `//` comments in `.json` files are discarded during JSONC parsing
- **CRLF** — normalizes to LF

!!! info "`theta` CLI"
    `theta cast to cursor --notes` and `theta cast from cursor --notes` print the full list of known cast limitations.

### MCP fields not in `[tools]`

Keys outside the theta-typed set (`command`, `args`, `env`, `url`, `headers`) round-trip through `[harness.cursor.mcp_extras.<name>]`. On cast, harness extras are merged with the theta-typed overlay — theta-typed keys win on conflict. Cursor-specific fields like `auth`, `envFile`, `type` preserve verbatim via this path. [ref](https://cursor.com/docs/mcp)

!!! note "No `mcp_input_variables`"
    Cursor does not support `${input:VAR}` prompts. Use `${env:NAME}` or `envFile` instead. [ref](https://cursor.com/docs/mcp)

## Harness-specific config

| `[harness.cursor]` field | Cast target | Notes |
|---|---|---|
| `version` | — | Semver range constraint. Cast warns if outside range. |
| `hooks` | `.cursor/hooks.json` | 21 events. [ref](https://cursor.com/docs/hooks) |
| `mcp_extras.<name>` | `.cursor/mcp.json` (merged) | Per-server extras not in `[tools]`. [ref](https://cursor.com/docs/mcp) |

!!! warning "Commands are deprecated"
    `.cursor/commands/` is deprecated. Cursor replaced them with [skills](https://cursor.com/docs/skills). theta does not import commands.

## Known limitations

- Plugin manifests (`.cursor-plugin/plugin.json`) are a distribution wrapper — theta imports components from canonical paths, not plugin manifests.
- Team/enterprise rules are cloud-managed via the Cursor dashboard — no file representation in the project.
- `loop_limit: null` means unlimited in Cursor, but TOML has no null type — key is omitted (defaults to 5). Use an explicit high number instead.
- Brace expansion in globs (`{a,b}`) is broken in Cursor — commas inside braces are consumed by the comma splitter.
- `.cursorignore` / `.cursorindexingignore` are gitignore-syntax files — not modeled.
