# Source resolution

---

See the [protocol spec](../spec/protocol.md#lock) for normative requirements.

!!! info "`theta` CLI"
    `theta lock` resolves and pins all sources. `theta lock --force` re-resolves unconditionally.

## Source types

Source types vary by resource kind:

### Rules (`LocalOrGitRef`)

| Source | Syntax | Resolution |
|---|---|---|
| **Local** | `"rules/my-rule.md"` (bare string) | Relative path to a `.md` file |
| **Git** | `{ git = "https://...", tag = "v1.0", file = "rule.md" }` | Single file from a git repo |
| **System** | `{ system = "name" }` | Named rule in the system store |

### Skills (`SourceRef`)

| Source | Syntax | Resolution |
|---|---|---|
| **Path** | `{ path = "./local/dir" }` | Local directory containing `SKILL.md` |
| **Git** | `{ git = "https://...", tag = "v1.0", subdirectory = "skills/x" }` | Directory (or subdirectory) from a git repo |
| **System** | `{ system = "name" }` | Named skill in the system store |

### Subagents

Subagents use `agent_ref` — a path to a sibling `theta.toml`. Subagent sources are resolved recursively during locking.

### System store

The `system` source references resources registered in the user's local system store. Resolved during locking by content hash, same as `path` sources.

The store location **MUST** follow [XDG Base Directory](https://specifications.freedesktop.org/basedir-spec/latest/) conventions. The default resolves to `$XDG_DATA_HOME/theta/store/` (Linux) or the platform equivalent on macOS / Windows. Implementations **SHOULD** respect a `THETA_DATA_DIR` environment variable override that points to the parent data directory (the store lives at `<THETA_DATA_DIR>/store/`).

!!! info "`theta` CLI"
    `theta register skill|rule|agent` adds resources to the store. `theta list store` lists registered entries. `theta rm store <kind> <name>` unregisters.

    **Template / placeholder rejection.** `theta register` rejects skills, rules, and agents whose content is the unedited scaffold template (e.g. `SKILL.md` containing the default `name`/`description` placeholders, or `theta.toml` with the placeholder `description`). The store is for finished, reusable resources — templates must be edited first.

    **`theta init --from <name>`** seeds a new agent from a registered system agent. The CLI copies the registered manifest, system prompt, rules, and skill directories into the new project and rewrites local rule `src` paths to the project-relative locations. Git-sourced rules and skills are copied verbatim (still resolved from their remote on the next `theta lock`).

## Lockfile

`theta.lock` pins the full dependency graph:

```toml title="theta.lock"
[meta]
schema = "2026-04"
manifest_hash = "sha256:..."

[instructions.system]
source = { path = "system.md" }
content_hash = "sha256:..."

[instructions.rules.my-rule]
source = { path = "rules/my-rule.md" }
content_hash = "sha256:..."

[skills.my-skill]
source = { path = "skills/my-skill" }
content_hash = "sha256:..."
```

The lockfile **MUST** be deterministic: same `theta.toml` produces the same `theta.lock`. The lockfile **MUST** be committed to version control.

!!! note "Forward-declared lock variants"
    The lockfile schema reserves `url = "..."` and `{ registry, name, version }` source shapes for future use. They are not produced by any current source type and implementations **MAY** parse but **MUST NOT** rely on them. The serialized lock schema may add support without bumping the manifest schema version.

### Staleness detection

The lock is stale when:

- `manifest_hash` does not match the current `theta.toml`
- Any local-path or system-sourced dependency has changed on disk

Git sources are never re-checked locally — they are locked by commit SHA.

`theta sync` re-locks automatically when stale. `theta lock --force` re-locks unconditionally.

### Atomic writes

!!! info "`theta` CLI"
    Lockfile writes use a write-to-temp + rename pattern for atomicity.

## Local cache

Resolved remote dependencies **MUST** be cached following [XDG Base Directory](https://specifications.freedesktop.org/basedir-spec/latest/) conventions:

- Default: `$XDG_CACHE_HOME/theta/` (resolves to `~/.cache/theta/` on Linux without `XDG_CACHE_HOME` set; platform-equivalent locations on macOS and Windows via [`etcetera`](https://docs.rs/etcetera))
- Git fetches are stored under `<cache>/git/`
- MCP registry responses under `<cache>/registry/`

Caching **SHOULD** be content-addressed where possible.
