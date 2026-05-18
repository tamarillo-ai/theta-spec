# Manifest spec

Normative field-level requirements for `theta.toml`. The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are used per [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

Machine-readable schema: [`theta.schema.json`](https://github.com/tamarillo-ai/theta-spec/blob/main/schemas/2026-04/theta.schema.json)

For guides with examples, casting tables, and implementation notes, see the [manifest guide](../manifest/index.md).

---

## Types

Custom types used throughout this document.

### `SourceRef`

A dependency source. Exactly one variant **MUST** be present:

| Variant | Shape | Description |
|---|---|---|
| path | `{ path = str }` | Local filesystem directory or file |
| git | `{ git = str }` | Remote git repository |
| system | `{ system = str }` | User's local system store |

All local `path` values **MUST** be relative to `theta.toml`. Absolute paths **MUST** be rejected. Paths referencing `.theta/` **MUST** be rejected.

Git sources accept the following optional fields. At most one of `branch`, `tag`, or `rev` **MAY** be set:

| Field | Type | Description |
|---|---|---|
| `branch` | `str` | Explicit branch name |
| `tag` | `str` | Explicit tag name |
| `rev` | `str` | Explicit commit SHA or rev-parse expression |
| `subdirectory` | `str` | Subdirectory within the repo |

`git` **MUST** have a recognized URL scheme (`https://`, `http://`, `git://`, `ssh://`). SCP-form URLs (`git@host:path`) are not valid.

### `LocalOrGitRef`

A rule source reference. Exactly one variant **MUST** be present:

| Variant | Shape | Description |
|---|---|---|
| local | `str` | Local path — the string value **MUST** end in `.md` |
| git | `{ git = str, file = str }` | File in a remote git repository |
| system | `{ system = str }` | User's system store — value **MUST** match `^[a-z0-9]+(-[a-z0-9]+)*$` |

Local paths **MUST** be relative to `theta.toml`. Absolute paths **MUST** be rejected. Paths referencing `.theta/` **MUST** be rejected.

Git sources accept the same optional ref fields as `SourceRef`: `branch`, `tag`, or `rev`. At most one **MAY** be set. `git` **MUST** have a recognized URL scheme.

### `ApplyMode`

Controls when a harness injects a rule into LLM context.

| Value | Meaning |
|---|---|
| `"always"` | Always injected (default) |
| `"model-decision"` | Model reads `description` and decides |
| `"glob"` | Injected when active file matches `apply_to` patterns |
| `"manual"` | Only on explicit user invocation |

---

## `[theta]`

**REQUIRED** section. **MUST** be present in every valid manifest.

### `schema`

- **REQUIRED**
- Type: `str`
- **MUST** be a calendar-based version identifier (e.g., `"2026-04"`)
- Implementations **MUST** reject manifests with incompatible schema versions
- Implementations **SHOULD** provide migration guidance on rejection

---

## `[agent]`

**REQUIRED** section. **MUST** be present in every valid manifest.

### `name`

- **REQUIRED**
- Type: `str`
- **MUST** match `^[a-z0-9]+(-[a-z0-9]+)*$`
- **MUST NOT** be empty

### `description`

- **REQUIRED**
- Type: `str`
- **MUST NOT** exceed 1024 characters

### `version`

- Type: `str`
- If present, **MUST** be strict [semver](https://semver.org/): `major.minor.patch`
- **MUST NOT** include pre-release suffixes
- **MUST NOT** include build metadata

### `authors`

- Type: `arr[str]`
- Each entry **MUST** match `"Name"` or `"Name <email>"`
    - When the `<email>` form is used, the email **MUST** contain `@`

### `model`

- Type: `str`
- Metadata field — signals which model the agent was designed for
- **MUST NOT** be consumed by [protocol operations](protocol.md)

### `tags`

- Type: `arr[str]`
- Each entry **MUST** match `^[a-z0-9]+(-[a-z0-9]+)*$`
- Each entry **MUST NOT** exceed 64 characters
- Metadata for discovery and categorization
- **MUST NOT** be consumed by [protocol operations](protocol.md)

---

## `[instructions]`

**MAY** be present.

### `system`

- Type: `str`
- The value **MUST** be a relative path ending in `.md`
- Absolute paths **MUST** be rejected
- Paths referencing `.theta/` **MUST** be rejected
- The referenced file **MUST** exist
- The referenced file **MUST** be a markdown document
- Implementations **SHOULD** warn when `[instructions.rules]` has entries but `system` is not set

### `[instructions.rules.<name>]`

Named rule tables. The key `<name>` is the rule identifier.

- `<name>` **MUST** match `^[a-z0-9]+(-[a-z0-9]+)*(/[a-z0-9]+(-[a-z0-9]+)*)*$` — simple kebab-case or path-qualified segments separated by `/`
- Leading, trailing, or consecutive `/` are invalid

A rule is a markdown file containing natural-language instructions that the harness injects into the model's context window before inference. The injection point varies by harness but in most cases it gets injected into the system prompt. The rule file **MUST** be pure content — activation metadata (`apply`, `apply_to`, `description`) is declared in `theta.toml` and casting injects harness-native frontmatter as needed.

#### `src`

- **REQUIRED**
- Type: [`LocalOrGitRef`](#localorgitref)

#### `summary`

- Type: `str`
- **MAY** be present
- Short label for display purposes

#### `description`

- Type: `str`
- **REQUIRED** when `apply` = `"model-decision"`
- Provides the text the model reads to decide whether to activate the rule

#### `apply`

- Type: [`ApplyMode`](#applymode)
- If omitted, defaults to `"always"`

#### `apply_to`

- Type: `arr[str]`
- File glob patterns (shell glob: `*`, `**`, `?`, `[...]`)
- **SHOULD** be present when `apply` = `"glob"`
- **SHOULD NOT** be present when `apply` ≠ `"glob"`
    - If present when `apply` ≠ `"glob"`, the patterns have no effect

---

## `[tools.<name>]`

**MAY** be present. Each tool is a named [MCP](https://modelcontextprotocol.io/) server declaration.

- `<name>` **MUST** match `^[a-z0-9]+(-[a-z0-9]+)*$`
- Each tool **MUST** have exactly one of `command` or `url`
    - Having both is invalid
    - Having neither is invalid

### `command`

- Type: `arr[str]`
- Command to start a stdio MCP server
- **REQUIRED** if `url` is absent

### `url`

- Type: `str`
- URL of a remote MCP server (streamable HTTP)
- **REQUIRED** if `command` is absent

### `env`

- Type: `table` (`str` -> `str`)
- Environment variables for the server process
- Keys **MUST** match POSIX format `[A-Za-z_][A-Za-z0-9_]*`
- Values are opaque strings passed through to harness-native config
- Values **MAY** use `${env:VAR_NAME}` as a convention to document host environment variable references — theta does not resolve this syntax

### `args`

- Type: `arr[str]`
- Additional command arguments

### `headers`

- Type: `table` (`str` --> `str`)
- HTTP headers for remote MCP servers
- Only meaningful when `url` is present — implementations **SHOULD** warn when set on a `stdio` tool

### `enabled`

- Type: `bool`
- Defaults to `true`
- When `false`, the tool is registered but not active

---

## `[skills.<name>]`

**MAY** be present. Each skill is a composable capability package following the [Agent Skills spec](https://agentskills.io/specification).

- `<name>` **MUST** match `^[a-z0-9]+(-[a-z0-9]+)*$`
- `<name>` **MUST NOT** exceed 64 characters
- `<name>` **MUST NOT** contain consecutive hyphens (`--`)

### `source`

- **REQUIRED**
- Type: [`SourceRef`](#sourceref)

### `tags`

- Type: `arr[str]`
- Each entry **MUST** match `^[a-z0-9]+(-[a-z0-9]+)*$`
- Each entry **MUST NOT** exceed 64 characters
- Metadata for discovery and categorization

### `goal`

- Type: `str`
- **MUST NOT** exceed 512 characters
- Machine-facing purpose statement for the skill
- Complements the `description` in `SKILL.md` frontmatter, which is human-facing

### Materialized skill requirements

After [materialization](protocol.md#materialize), the resolved skill directory **MUST** satisfy:

- A `SKILL.md` file **MUST** be present at the root of the skill directory
- The `SKILL.md` **MUST** contain YAML frontmatter with at least:
    - `name` — **MUST** match the skill's `<name>` key
    - `description` — **MUST NOT** be empty

These requirements follow the [Agent Skills spec](https://agentskills.io/specification).

---

## `[[subagents]]`

**MAY** be present. TOML array of tables — each entry defines a child agent.

A subagent **MUST** be in exactly one of three modes:

- **Ref mode**: `ref` is present — all inline fields (`prompt_path`, `model`, `tools`, `skills`) **MUST** be absent
- **Inline mode**: `ref` is absent and `prompt_path` is present — the subagent is defined by inline fields
- **Description-only mode**: `ref` and `prompt_path` are both absent — the subagent is defined by `description` alone, with no prompt file

### `name`

- **REQUIRED**
- Type: `str`
- **MUST** match `^[a-z0-9]+(-[a-z0-9]+)*$`

### `description`

- **REQUIRED**
- Type: `str`
- Non-ref subagents **SHOULD** have a non-empty description

### `ref`

- Type: `str`
- Relative path to a sibling `theta.toml` manifest
- **MUST** end in `.toml`
- Absolute paths **MUST** be rejected
- Paths referencing `.theta/` **MUST** be rejected
- Only local file paths are supported in this version of the spec — git URLs and system store references are not accepted in `ref`
- Mutually exclusive with inline fields (see above)

### `prompt_path`

- Type: `str`
- Relative path to a `.md` file containing the subagent's system prompt
- **MUST** end in `.md`

### `model`

- Type: `str`
- Model identifier
- If omitted, inherits from the parent agent

### `tools`

- Type: `arr[str]`
- Tool allow-list
- If omitted, inherits from the parent

### `skills`

- Type: `arr[str]`
- Skill names to inject at startup

---

## `[harness.<name>]`

**MAY** be present. Each subsection is a harness-specific configuration passthrough.

- Implementations **MUST** preserve harness sections during parsing and round-tripping
- When casting to harness X, only `[harness.X]` is read
    - All other `[harness.*]` sections **MUST** be ignored
- A single manifest **MAY** contain `[harness.<name>]` sections for multiple harnesses simultaneously
- Harness sections are opaque to this spec — their contents are defined by the target harness

---

## `[extras.<name>]`

**MAY** be present. Extras sections are opaque third-party configuration.

- Implementations **MUST** preserve extras sections during parsing and casting
- `<name>` is unconstrained — extras live under the `[extras.*]` namespace, so a name matching a reserved top-level section (`tools`, `agent`, etc.) is structurally distinct
