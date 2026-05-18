# Overview

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are used per [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

- [These docs](https://theta-spec.tamarillo.ai/)
- [These docs' source](https://github.com/tamarillo-ai/theta-spec)
- CLI docs: [theta.tamarillo.ai](https://theta.tamarillo.ai/)

---

## Glossary

**Manifest** — a `theta.toml` file. The single source of truth for an agent's configuration.

**Harness** — the runtime that executes an agent. Each has its own native config format, check [supported harnesses](harnesses/index.md).

**Implementation** — a tool that reads `theta.toml` and implements the [protocol](protocol/index.md). The reference implementation is the [theta CLI](https://theta.tamarillo.ai/).

**Casting** — converting `theta.toml` into harness-native config files. Forward: `theta.toml --> harness`. Reverse: `harness --> theta.toml`. Casting MAY be lossy.

**Lossy cast** — a cast where information cannot be fully represented in the target format. Dropped fields MUST produce warnings.

**Lockfile** — `theta.lock`. Pins every dependency to an exact version, commit, or content hash. Deterministic: same manifest --> same lockfile.

**Materialization** — resolving all dependencies per the lockfile and writing them to disk so that casting can read actual file content.

**Source** — where a dependency comes from: `path`, `git`, or `system`. See [source resolution](protocol/sources.md).

**System store** — a personal library for reusing rules, skills, and agents across projects. An implementation concern, not a manifest section.

**Skill** — a composable capability package following the [Agent Skills spec](https://agentskills.io/specification).

**Subagent** — a child agent with its own model, tools, and permissions.

**Rule** — a markdown instruction file with metadata controlling when the harness injects it into LLM context.

**Tool** — an [MCP](https://modelcontextprotocol.io/) server declaration (stdio or HTTP).

**Diagnostic** — a validation message emitted by an implementation. Three levels: `error`, `warn`, `hint`.

**Schema version** — calendar-based identifier (e.g., `"2026-04"`) that pins the manifest format. Published with a [JSON Schema](manifest/format.md#versions-serialized-schema).

**Plugin** — an opaque third-party config section under `[extras.<name>]`. Preserved during parsing and casting, never interpreted by theta-spec. The escape hatch for external tools that want `theta.toml` as their configuration home.

**Harness extension** — harness-specific config under `[harness.<name>]`. Passthrough — only read when casting to the named harness.

**Frontmatter** — YAML metadata prepended to markdown during casting, when the target harness expects it.

---

## Scope

theta-spec defines two things:

- **The manifest format** — `theta.toml` sections, fields, types, and constraints. See [manifest](manifest/index.md).
- **The protocol** — how implementations MUST behave. See [protocol](protocol/index.md).


---

## Constraints

Any conformant implementation MUST satisfy these invariants:

- **Manifest is source of truth.** If it's not in `theta.toml`, it is invisible.
- **Casting is deterministic.** Same manifest + same target --> same output files. No LLM calls, no network access during cast.
- **Silent loss is forbidden.** When a field has no equivalent in the target harness, the implementation MUST emit a warning.

---

## Versioning

Calendar-based schema versions. Current: `"2026-04"`.

Each version has a corresponding [JSON Schema](manifest/format.md#versions-serialized-schema). The schema version MUST be bumped when a required field is added, renamed, or removed, or when a field's type changes. It MUST NOT be bumped for optional additions.

Implementations MUST reject manifests with incompatible schema versions and SHOULD provide migration guidance.

Full versioning policy: [format](manifest/format.md).

---

## Extensibility

There are two interfaces for extending behavior:

- **`[harness.<name>]`** — passthrough config for features that have no portable equivalent. This is the escape hatch for harness-specific behavior. Take a look at [harness extensions](manifest/harness-extensions.md) for more details.
- **`[extras.<name>]`** — opaque sections for third-party tools. They MUST be preserved during parsing and casting, never interpreted by theta-spec. Any external tool can use `theta.toml` as its config home by reading its own plugin section. See [extras](manifest/extras.md).

Adding a new harness means writing a casting adapter and a harness reference page. The manifest format does not change. To request support for a new harness, [open an issue](https://github.com/tamarillo-ai/theta-spec/issues/new?labels=new-harness&title=harness:+%3Cname%3E) with the harness name, links to its official config docs, and a list of its config file surfaces.

---

## Contributing

theta-spec is developed in the open. Contribution guidelines, issue templates, and PR conventions live in the [theta-spec repo](https://github.com/tamarillo-ai/theta-spec/blob/main/CONTRIBUTING.md).


---

## Normative references

| Standard | Usage in theta-spec |
|---|---|
| [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) | Requirement-level keywords (MUST, SHOULD, MAY) |
| [TOML v1.1](https://toml.io/en/v1.1.0) | Manifest serialization format |
| [JSON Schema](https://json-schema.org/) | Manifest validation |
| [MCP](https://modelcontextprotocol.io/) | Tool protocol for `[tools.*]` |
| [Agent Skills spec](https://agentskills.io/specification) | Skill packaging format for `[skills.*]` |
| [Semver](https://semver.org/) | `[agent].version` format |
| [XDG Base Directory](https://specifications.freedesktop.org/basedir-spec/latest/) | Cache and store paths |
