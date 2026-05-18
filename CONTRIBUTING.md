# Contributing to theta-spec

theta-spec is developed by tamarillo.ai [here](https://github.com/tamarillo-ai/theta-spec).

The reference implementation for the specification is [theta](https://theta.tamarillo.ai/).

## Enhancement proposals

We denominate spec changes **TEP** (theta-spec enhancement proposal). The naming convention is `TEP-<number>`.

## Requesting a new harness

To request support for a new harness, [open an issue](https://github.com/tamarillo-ai/theta-spec/issues/new?labels=new-harness&title=harness:+%3Cname%3E) with:

- Harness name and URL
- Links to official config documentation
- List of config file surfaces (what files, what format)
- Any known limitations for casting
- Any known statistics on usability

Adding a harness MUST include as a minimum implementation surface:

- A casting adapter in [theta](https://theta.tamarillo.ai/)
- A [harness reference page](https://github.com/tamarillo-ai/theta-spec/tree/main/docs/harnesses) in the spec docs
- An entry in [harnesses.toml](https://github.com/tamarillo-ai/theta-spec/blob/main/harnesses.toml)


## What goes where, rules of thumb

| Change type | Where |
|---|---|
| New manifest field or section | [Manifest guide](docs/manifest/) page + [manifest spec](docs/spec/manifest.md) |
| New protocol operation | [Protocol guide](docs/protocol/) page + [protocol spec](docs/spec/protocol.md) |
| New harness support | [Harness reference](docs/harnesses/) page + `harnesses.toml` entry |
| Casting behavior change | Harness page + relevant manifest guide casting tables |
| Bug fix / clarification | Inline edit, no version bump |


## Style

Style is explicit in `STYLE.md` for each relevant repo ([theta style](https://github.com/tamarillo-ai/theta/blob/main/STYLE.md), [theta-spec doc style](https://github.com/tamarillo-ai/theta-spec/blob/main/STYLE.md)). The common set of guidelines includes:

- Use [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) keywords with normative meaning
- Use sentence case in headings and lists
- Tables over prose for structured data
- Link to things inline
- Don't repeat information that's already documented elsewhere — link to it


## Building the docs

```bash
uv pip install mkdocs-material
mkdocs build
mkdocs serve
```
