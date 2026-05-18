# Format

---

## TOML v1.1

`theta.toml` uses [TOML v1.1](https://toml.io/). One file per agent. All paths are relative to `theta.toml`.


## Schema version

Every manifest MUST declare its schema version:

```toml
[theta]
schema = "2026-04"
```

Implementations MUST reject manifests with incompatible schema versions and provide migration guidance.

### Versioning policy

The schema version MUST be bumped when:

- A required field is added, renamed, or removed
- A field's type changes
- A section is renamed or removed
- A new section is introduced (even if optional)

The schema version MUST NOT be bumped when:

- An optional field is added to an existing section
- Documentation or descriptions change

Implementations **MUST** reject unrecognized fields within typed sections (`[theta]`, `[agent]`, `[instructions]`, `[tools.*]`, `[skills.*]`, `[[subagents]]`). This ensures manifests are strict and typos are caught early. The `[extras.*]` and `[harness.*]` sections are the designated extensibility points — they accept arbitrary nested values without rejection.

## Versions serialized schema

| Version | Schema |
|---|---|
| `2026-04` | [`theta.schema.json`](https://github.com/tamarillo-ai/theta-spec/blob/main/schemas/2026-04/theta.schema.json) |


## Minimum config

A minimal manifest contains two required sections:

```toml
[theta]
schema = "2026-04"

[agent]
name = "my-agent"
description = "add your description here"
```

`[theta]` and `[agent]` are the only mandatory sections. `version`, `authors`, and `model` are optional metadata within `[agent]`. The rest of the configuration surface is optional. The full section reference can be found [here](index.md).

## Canonical section ordering

Implementations that scaffold or emit `theta.toml` files **SHOULD** order top-level sections as:

`[theta]` --> `[agent]` --> `[instructions]` --> `[instructions.rules.*]` --> `[tools.*]` --> `[skills.*]` --> `[[subagents]]` --> `[harness.*]` --> `[extras.*]`

This ordering is normative for `theta cast from` output and recommended for `theta add` operations. Hand-edited manifests **MAY** deviate from the canonical order — the parser is order-insensitive within the TOML grammar.
