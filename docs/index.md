# theta-spec

**theta-spec** is a declarative, harness-agnostic configuration standard for AI coding agents. One `theta.toml` file defines the full configuration surface, i.e. instructions, rules, tools, skills, subagents. Any theta-spec compliant implementation can resolve, lock, and cast it to any supported harness.

---

## Motivation

Agent harnesses are here to stay. Each one ships its own configuration format but still all of them share similarities. Maintaining and declaring configurations is a burden. There is no standard way to share, version, or reproduce an agent configuration.

Parametrizing the configuration surface into a single manifest:

- Displays the exhaustive configuration at a glance
- Provides an entrypoint for searching the resources that define an agent
- Enables reproducible configurations between people and between agents
- Makes mutation strategies explicit and diffable
- Facilitates maintenance across harness updates
- Enables a project lifecycle tool, which is why the [theta CLI](https://theta.tamarillo.ai/) exists

A parametrized function and a well-defined cost function enables function approximation. If the goal is to optimize the way we work with agents, this is the first tiny step towards building the tooling needed to cover at least one gap, the parameters.

---

## Understanding the spec: `theta.toml`

Most of the spec is read-self-explanatory for anyone familiar with agent harnesses. A detailed and exhaustive description of each one of the defining fields can be found [condensed here](spec/manifest.md) and also [in the manifest section of this doc](manifest/index.md).

This is how a `theta.toml` looks:

```toml
[theta]
schema = "2026-04"

[agent]
name = "harness-researcher"
description = "researches agent harness configurations across public repos"
version = "0.1.0"
authors = ["ivan <ivan@tamarillo.ai>"]
model = "claude-sonnet-4-20250514"

[instructions]
system = "instructions/system.md"

[instructions.rules.concise]
src = "instructions/rules/concise.md"
apply = "always"

[instructions.rules.rust]
src = "instructions/rules/rust.md"
apply = "glob"
apply_to = ["*.rs"]

[tools.context7]
command = ["npx", "-y", "@upstash/context7-mcp@latest"]

[tools.playwright]
command = ["npx", "@anthropic-ai/playwright-mcp"]

[tools.memory]
command = ["npx", "-y", "@modelcontextprotocol/server-memory"]

[skills.osint]
source = { path = "skills/osint" }

[[subagents]]
name = "scraper"
description = "scrapes github repos for harness config patterns"
model = "claude-sonnet-4-20250514"
tools = ["playwright"]

[[subagents]]
name = "analyst"
ref = "agents/analyst/theta.toml"
```

---

## Structure of this docs

This doc is organized in four sections. The standard includes the definition of the spec as well as the protocols that any theta-spec compliant tool **MUST** satisfy. [theta](https://theta.tamarillo.ai/) is the reference implementation.

- **[Manifest](manifest/index.md)** — guide pages for `theta.toml`: sections, fields, casting tables, examples, and implementation notes
- **[Protocol](protocol/index.md)** — guide pages for protocol operations: lock, materialize, cast, import, and the agent config lifecycle
- **[Harness reference](harnesses/index.md)** — per-harness casting notes, output files, and known limitations
- **[Spec](spec/index.md)** — normative requirements specification: field-level requirements ([manifest spec](spec/manifest.md)) and operational requirements ([protocol spec](spec/protocol.md))

---

## Supported harnesses

The following harnesses are supported. Prioritization is based on adoption research.



- [Claude Code](https://code.claude.com/)
- [Codex CLI](https://github.com/openai/codex)
- [Cursor](https://cursor.com/) (3+)
- [GitHub Copilot](https://code.visualstudio.com/docs/copilot/overview)

---

## Related docs

- [theta](https://theta.tamarillo.ai/) — Rust CLI (reference implementation)
- [MCP](https://modelcontextprotocol.io/specification/2025-11-25) — tool protocol standard
- [Agent Skills spec](https://agentskills.io/specification) — skill packaging standard
