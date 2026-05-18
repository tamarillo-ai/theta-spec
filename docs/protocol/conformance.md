# Conformance

---

See the [protocol spec](../spec/protocol.md) for normative requirements. This page describes what conformance looks like in practice.

## Operations

A conformant implementation **MUST** support all six protocol operations: [validate](../spec/protocol.md#validate), [lock](../spec/protocol.md#lock), [materialize](../spec/protocol.md#materialize), [cast to](../spec/protocol.md#cast-to), [cast from](../spec/protocol.md#cast-from), and [check](../spec/protocol.md#check).

### Auxiliary CLI surface

Implementations **MAY** expose additional CLI verbs beyond the six protocol operations. The `theta` CLI ships the following:

| Verb | Purpose |
|---|---|
| `theta init` | Scaffold a new `theta.toml` (and optionally seed from a registered system agent via `--from`) |
| `theta schema` | Print the JSON Schema for the manifest |
| `theta add <kind>` | Add a rule, system prompt, tool, skill, or subagent to the manifest |
| `theta rm <kind> [--delete]` | Remove a manifest entry; `--delete` also deletes the source file or directory (supported for rule, system, skill, subagent) |
| `theta list <kind>` | List manifest entries or system store contents |
| `theta register <kind>` | Copy a local skill, rule, or agent into the system store (`--no-lock` skips the post-register lock step for `register agent`) |
| `theta tree` | Print the subagent dependency graph |

These verbs are not normative — they are useful affordances on top of the six core operations and other implementations **MAY** expose equivalent surfaces under different names.

### Global flags

The following global flags are accepted on every `theta` subcommand:

| Flag | Purpose |
|---|---|
| `--directory <path>` | Treat `<path>` as the project root (defaults to the current directory) |
| `--manifest <file>` | Use `<file>` instead of `<directory>/theta.toml` |
| `--instructions-dir <dir>` | Override the default `instructions/` directory (env: `THETA_INSTRUCTIONS_DIR`) |
| `--rules-dir <dir>` | Override the default `rules/` subdirectory (env: `THETA_RULES_DIR`) |

Resolution order: CLI flag → environment variable → built-in default.

## Validation

Implementations **MUST** validate `theta.toml` against the published [JSON Schema](../manifest/format.md#versions-serialized-schema) for the declared schema version before any operation that consumes the manifest.

Invalid manifests **MUST** cause immediate failure with actionable diagnostics.

## Check

Full-pipeline validation combining structural, reachability, and content quality checks.

!!! info "`theta` CLI"
    `theta check` runs the full pipeline. `theta check --schema-only` runs structural validation only (TOML parse + manifest deserialization + field constraints; no filesystem reads). `theta check --skip-materialization` runs everything except the materialization phase and downgrades missing-`.theta/` errors to warnings — useful for validating a freshly cloned repository where `theta.lock` is present but `theta sync` has not yet run. `theta check --output-format json` produces machine-readable output. `--schema-only` and `--skip-materialization` are mutually exclusive.

### Phases

| Phase | What it checks |
|---|---|
| **Structural** | TOML parse, schema version, field constraints (name format, source validity, conflicting options) |
| **Reachability** | Local paths exist on disk, system store entries present, git refs resolvable |
| **Content quality** | Empty or placeholder system prompts and rules |
| **Lock consistency** | Manifest hash matches lock, no orphaned lock entries, no source drift |
| **Materialization consistency** | `.theta/` exists, expected files present, no unexpected files |
| **Harness config** | Version constraints for all known harnesses |

### Diagnostics

Three severity levels:

| Level | Meaning | Example |
|---|---|---|
| `error` | Invalid state — **MUST** be fixed | Rule name not kebab-case, tool with neither `command` nor `url` |
| `warn` | Likely problem — **SHOULD** be addressed | Stale lockfile, placeholder system prompt, lossy cast |
| `hint` | Informational — **MAY** be relevant | No lockfile yet, consider pinning harness version |

Implementations **SHOULD** support both human-readable and machine-readable output formats.

!!! info "`theta` CLI — output format"
    Human-readable output uses colored `error:`, `warn:`, `hint:` prefixes. JSON output (`--output-format json`) uses this structure:

    ```json
    {
      "valid": false,
      "errors": 1,
      "warnings": 2,
      "hints": 3,
      "diagnostics": [
        { "level": "error", "path": "tools.my-tool", "message": "must specify command or url" }
      ]
    }
    ```

### Exit codes

Mutating operations (lock, sync, cast, add, rm, register) **MUST** exit with status `0` on success and a non-zero status on any error.

`theta check` and other validation surfaces **MUST**:

- Exit `0` when no diagnostics of level `error` were emitted (warnings and hints alone do not fail)
- Exit non-zero when any `error` diagnostic was emitted

The specific non-zero value is not normative; the `theta` CLI uses the standard convention `1` for diagnostic failures and `2` for invalid argument combinations (from clap).

## Global config

Implementations **MAY** manage user-level defaults at `~/.config/theta/config.toml` ([XDG](https://specifications.freedesktop.org/basedir-spec/latest/)).

!!! info "`theta` CLI"
    `config.toml` is not yet implemented. User-level defaults are not currently supported.

If a non-theta-managed config already exists for a target harness (e.g., `~/.claude/CLAUDE.md`), implementations **MUST** warn and refuse to overwrite. They **SHOULD** offer to merge or skip.
