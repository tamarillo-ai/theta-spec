# GitHub Copilot

!!! info "Harness compatibility"
    **Last verified**: 2026-04-19 against VS Code 1.99+ docs (pages dated 2026-04-15)  
    **Docs**: [code.visualstudio.com/docs/copilot](https://code.visualstudio.com/docs/copilot/reference/copilot-settings)  
    Stale? [File an issue](https://github.com/tamarillo-ai/theta-spec/issues/new?labels=stale-harness&title=stale:+copilot).

## Config surface

| Surface | File | Format | theta section | Docs |
|---|---|---|---|---|
| System prompt | `.github/copilot-instructions.md` | Markdown | `[agent]` + `[instructions].system` | [Custom instructions](https://code.visualstudio.com/docs/copilot/customization/custom-instructions#_use-a-githubcopilotinstructionsmd-file) |
| Rules | `.github/instructions/**/*.instructions.md` | Markdown + YAML fm (`name`, `description`, `applyTo`) | `[instructions.rules]` | [Custom instructions](https://code.visualstudio.com/docs/copilot/customization/custom-instructions#_use-instructionsmd-files) |
| Skills | `.github/skills/<name>/SKILL.md` | Markdown + YAML fm (`name`, `description`, `argument-hint`, `user-invocable`, `disable-model-invocation`) | `[skills]` | [Agent skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills) |
| MCP servers | `.vscode/mcp.json` (`"servers"` key) | JSON | `[tools]` | [MCP configuration](https://code.visualstudio.com/docs/copilot/reference/mcp-configuration) |
| Custom agents | `.github/agents/<name>.agent.md` | Markdown + YAML fm (`description`, `name`, `model`, `tools`, `agents`, `handoffs`, `hooks`, etc.) | `[[subagents]]` | [Custom agents](https://code.visualstudio.com/docs/copilot/customization/custom-agents) |
| VS Code settings | `.vscode/settings.json` | JSON | `[harness.github_copilot]` | [Settings](https://code.visualstudio.com/docs/copilot/reference/copilot-settings) |
| Hooks | `.github/hooks/*.json` | JSON (`"hooks"` key with lifecycle event arrays) | `[harness.github_copilot].hooks` | [Hooks](https://code.visualstudio.com/docs/copilot/customization/hooks) |
| Prompt files | `.github/prompts/*.prompt.md` | Markdown + optional YAML fm | *Not modeled* | [Prompt files](https://code.visualstudio.com/docs/copilot/customization/prompt-files) |
| Always-on (cross-agent) | `AGENTS.md`, `CLAUDE.md` | Markdown | *Not modeled* | [AGENTS.md](https://code.visualstudio.com/docs/copilot/customization/custom-instructions#_use-an-agentsmd-file), [CLAUDE.md](https://code.visualstudio.com/docs/copilot/customization/custom-instructions#_use-a-claudemd-file) |

!!! note "Alternative file locations"
    Copilot also discovers files at `.claude/rules/`, `.claude/agents/`, `.claude/skills/`, `~/.copilot/*/`, and `~/.claude/*/`. `theta cast from copilot` reads only the `.github/` and `.vscode/` paths. Locations are configurable via `chat.instructionsFilesLocations`, `chat.agentFilesLocations`, `chat.agentSkillsLocations`, `chat.hookFilesLocations`, `chat.promptFilesLocations`.

## Output files

| theta operation | Output | Format |
|---|---|---|
| Cast instructions | `.github/copilot-instructions.md` | Markdown |
| Cast rules | `.github/instructions/**/*.instructions.md` | Markdown (with `applyTo` frontmatter) |
| Cast subagents | `.github/agents/<name>.agent.md` | YAML frontmatter + Markdown |
| Cast skills | `.github/skills/*/SKILL.md` | YAML frontmatter + Markdown |
| Cast MCP servers | `.vscode/mcp.json` | JSON |
| Cast harness config | VS Code `settings.json` (merged) | JSON |
| Cast hooks | `.github/hooks/theta-hooks.json` | JSON (wrapped under `"hooks"` key) |

## Casting notes

- VS Code scans `.github/instructions/` **recursively** for `*.instructions.md` files. [ref](https://code.visualstudio.com/docs/copilot/customization/custom-instructions) `theta cast from copilot` preserves subdirectory structure as path-qualified rule names (`review/pr-review`). `theta cast to copilot` writes to matching subdirectories. Copilot is the only harness that natively supports rule subdirectories.
- Secrets use `${input:VAR}` (user-prompted at runtime) instead of `${env:VAR}`. Casting SHOULD map `${env:VAR}` to `${input:VAR}` and warn that the user will be prompted. [ref](https://code.visualstudio.com/docs/copilot/reference/mcp-configuration#_input-variables-for-sensitive-data)
- Agent files use `.agent.md` extension with YAML frontmatter. VS Code also detects plain `.md` files in `.github/agents/`. [ref](https://code.visualstudio.com/docs/copilot/customization/custom-agents#_custom-agent-file-structure)
- MCP uses `"servers"` key (NOT `"mcpServers"` which is Claude Code's format). [ref](https://code.visualstudio.com/docs/copilot/reference/mcp-configuration#_configuration-structure)

### Known round-trip limitations

YAML frontmatter in `.instructions.md` and `.agent.md` files is parsed and re-serialized. The following differences are expected and functionally equivalent:

- **Quote style** — `applyTo: "**"` MAY become `applyTo: '**'`. `serde_norway` controls quoting; no Rust YAML crate supports preserving original quote style.
- **Inline --> block sequences** — `tools: ['read', 'search']` MAY become a block list. `serde_norway` always emits block-style sequences.
- **Frontmatter key ordering** — key order MAY change. `serde_norway` does not guarantee insertion-order serialization.
- **YAML comments stripped** — `# comment` lines inside frontmatter are discarded by all YAML parsers (per YAML spec, comments are not part of the data model).
- **CRLF --> LF** — Windows line endings normalize to LF.
- **JSONC comments stripped** — `//` comments in `settings.json` and `mcp.json` are discarded during JSONC --> JSON parsing.
- **Agent `name:` added** — cast always emits `name:` in agent frontmatter for round-trip fidelity. Agents that originally had no `name:` field will have it added. VS Code derives name from the filename so this is functionally harmless.
- **MCP `type` added** — cast MAY add `"type": "stdio"` to MCP server configs when inferred from the `command` field.

These limitations apply to all harnesses that use YAML frontmatter, not just Copilot.

!!! info "`theta` CLI"
    `theta cast to copilot --notes` and `theta cast from copilot --notes` print the full list of known cast limitations.

### MCP fields not in `[tools]`

Keys outside the theta-typed set (`type`, `command`, `args`, `env`, `url`, `headers`) round-trip through `[harness.github_copilot.tool.<name>]`. On cast, the harness extras are merged with the theta-typed overlay — the theta-typed value wins on conflict and `validate_config` emits a warning. VS Code-specific runtime fields like `sandboxEnabled`, `sandbox`, `envFile`, `dev` preserve verbatim via this path. The top-level `inputs` array round-trips through `[harness.github_copilot.mcp_input_variables]`.

### Subagent fields not in `[[subagents]]`

Keys outside the theta-typed set (`name`, `description`, `model`, `tools`) round-trip through `[harness.github_copilot.subagent.<name>]`. On cast, harness extras merge into the `.agent.md` frontmatter with theta-typed keys winning. Fields handled this way include `agents`, `argument-hint`, `user-invocable`, `disable-model-invocation`, `target`, `mcp-servers`, `handoffs`, `hooks`, `infer` (deprecated). [ref](https://code.visualstudio.com/docs/copilot/customization/custom-agents#_header-optional) `name` in the extras is redundant (implicit via the filename / `[[subagents]].name`) and `validate_config` emits a hint.

## Harness-specific config

`[harness.github_copilot]` stores Copilot-specific VS Code settings using their native key names. On import, only keys in the `github.copilot.*` and `chat.*` namespaces are captured. On cast, they are written back to `.vscode/settings.json` verbatim.

```toml
[harness.github_copilot]
"chat.agent.maxRequests" = 50
"chat.tools.terminal.enableAutoApprove" = true
"chat.tools.global.autoApprove" = false
"github.copilot.chat.codeGeneration.useInstructionFiles" = true
```

All keys pass through without transformation — no aliases, no renaming. Full enumeration: [settings reference](https://code.visualstudio.com/docs/copilot/reference/copilot-settings).

!!! note
    Non-Copilot settings (`editor.*`, `files.*`, `search.*`, etc.) are NOT imported. They belong to VS Code, not to the agent manifest. On `cast to copilot`, existing non-Copilot keys in `.vscode/settings.json` are preserved by the merge.

!!! info "Imported key prefixes"
    Import grabs keys matching `github.copilot.*` and `chat.*`. The `chat.*` prefix includes both agent-specific settings (`chat.agent.*`, `chat.tools.*`, `chat.mcp.*`) and general chat UI settings (`chat.editor.*`, `chat.fontSize`, etc.). theta does not distinguish between them — all are stored in `[harness.github_copilot]` and round-trip verbatim. This avoids data loss as VS Code continues adding new `chat.*` keys.

!!! warning "Discovery path settings are not followed"
    VS Code settings like `chat.agentFilesLocations`, `chat.instructionsFilesLocations`, `chat.hookFilesLocations`, `chat.agentSkillsLocations`, and `chat.promptFilesLocations` override where VS Code looks for config files. theta does NOT follow these paths — import always reads from the fixed well-known locations (`.github/agents/`, `.github/instructions/`, `.github/skills/`, `.github/hooks/`). The discovery settings round-trip as opaque data in `[harness.github_copilot]` but do not change theta's import behavior. If a project uses custom discovery paths, files at those locations will not be imported.

!!! warning
    Hooks are in **preview** in VS Code. The format and behavior may change. [ref](https://code.visualstudio.com/docs/copilot/customization/hooks)

Copilot supports 8 lifecycle hook events: `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PreCompact`, `SubagentStart`, `SubagentStop`, `Stop`. VS Code scans every `*.json` file in `.github/hooks/` and merges the results. [ref](https://code.visualstudio.com/docs/copilot/customization/hooks#_hook-lifecycle-events)

theta imports the hook **configuration** (event --> command mapping) and stores it in `[harness.github_copilot].hooks`. theta does NOT manage the **scripts** those commands reference — if a hook command points to `.github/hooks/scripts/policy.sh`, that script must exist in the project independently. This is the same model as `[tools]` MCP servers: theta stores `"command": "npx"` but doesn't manage the `npx` binary.

**Import.** Every `*.json` file under `.github/hooks/` is parsed and the per-event arrays are union-merged into a single event map. Both wrapped (`{"hooks": {...}}`) and bare (`{"PreToolUse": [...]}`) shapes are accepted. Multiple files are consolidated — original file grouping is not preserved.

**Cast.** All theta-managed hooks are written to `.github/hooks/theta-hooks.json`. VS Code picks this up alongside any user-authored sibling `*.json` files.

!!! warning "Duplication risk"
    VS Code hooks are still a preview feature; it scans multiple `*.json` files in a directory and merges them ([ref](https://code.visualstudio.com/docs/copilot/customization/hooks#_hook-file-locations)). After `theta cast to copilot`, if the original hook files (e.g. `memory-capture.json`, `policy.json`) still exist alongside `theta-hooks.json`, VS Code will merge all of them and **hooks may fire twice**. `theta cast to copilot` emits a warning when this is detected. Remove or rename the originals after importing into theta.

!!! note "Scripts are the user's responsibility"
    Hook commands are free-form strings (`bash ./script.sh`, `npx tool`, `python -c '...'`). theta stores and round-trips these strings verbatim. If the referenced script or binary doesn't exist at runtime, the harness will fail — theta does not validate command availability.

!!! note "Future consideration"
    If hooks stabilize across harnesses and converge on a common event model, a first-class `[hooks]` manifest section MAY be introduced. The declarative part (event --> command mapping) is portable; the imperative part (script files) remains outside theta's managed surface, like MCP server binaries today.

## Prompt files

`.github/prompts/*.prompt.md` are manually-invoked slash commands in VS Code. [ref](https://code.visualstudio.com/docs/copilot/customization/prompt-files) theta does not model prompt files — they overlap with [skills](../manifest/skills.md) (both are slash-command invocable), and skills are the cross-harness portable format via [agentskills.io](https://agentskills.io/).

## Cross-harness file reading

Copilot discovers files from other harness locations: `.claude/rules/`, `.claude/agents/`, `.claude/skills/`, `AGENTS.md`, `CLAUDE.md`. [ref](https://code.visualstudio.com/docs/copilot/customization/custom-instructions#_use-a-claudemd-file) `theta cast from copilot` reads only `.github/` and `.vscode/` paths by default.

!!! info "`theta` implementation notes — cross-read"
    `theta cast from copilot --cross-read` also imports files from other harness locations that Copilot discovers at runtime:

    - `AGENTS.md`, `CLAUDE.md`, `.claude/CLAUDE.md`, `CLAUDE.local.md` — concatenated into the system prompt
    - `.claude/rules/*.md` — imported as rules (with `paths` frontmatter mapped to `apply_to` globs)

    Each cross-read file emits a diagnostic hint warning about duplication risk on subsequent round-trips. Cross-read is opt-in to avoid silent content duplication.

## Known limitations

- Permissions use an allowlist model, not `allow`/`ask`/`deny`. Casting maps `deny` to exclusion from the allowlist, `ask` to `PreToolUse` hooks. [ref](https://code.visualstudio.com/docs/copilot/customization/hooks#_pretooluse)
- No configurable compaction. Copilot supports `PreCompact` hooks but no compaction settings. [ref](https://code.visualstudio.com/docs/copilot/customization/hooks#_precompact)
- `chat.agent.maxRequests` defaults to 25 (request limit per session, not a file count limit). [ref](https://code.visualstudio.com/docs/copilot/reference/copilot-settings#_agent-settings)
- `cast to copilot` MERGES `.vscode/settings.json` and `.vscode/mcp.json` with existing files at the output directory. theta-owned keys overwrite same-named entries; unrelated user keys are preserved. For `mcp.json`, user-authored server entries not in `[tools]` and the top-level `inputs` array survive when theta does not produce one. theta-owned Markdown files (`.github/copilot-instructions.md`, `.github/instructions/*`, `.github/agents/*`, `.github/skills/*`, `.github/hooks/theta-hooks.json`) are fully owned by theta and overwritten on each cast.
- System prompt identity header parsing assumes `# heading\n\ndescription\n\nbody` format. Files without a blank line between description and body emit a hint and fall back to treating the entire block as description.
