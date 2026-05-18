# Spec

The normative core of theta-spec. These pages define what a valid `theta.toml` looks like and how conformant implementations MUST behave.

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are used per [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

- **[Manifest](manifest.md)** — field-level requirements for every `theta.toml` section
- **[Protocol](protocol.md)** — operational requirements for resolve, lock, materialize, cast, and import

The machine-readable counterpart for this is the [`theta.schema.json`](https://github.com/tamarillo-ai/theta-spec/blob/main/schemas/2026-04/theta.schema.json).

For explanatory guides with examples and casting tables, see the [manifest guide](../manifest/index.md) and [protocol guide](../protocol/index.md).
