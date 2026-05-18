# Manifest

The manifest is a `theta.toml` file. It fully describes an agent: identity, instructions, tools, skills, and subagents.

## Sections

The **Requirement** column uses [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) keywords:

- **MUST** means a valid `theta.toml` requires this section
- **SHOULD** means it is expected in most real configs
- **MAY** means it is entirely optional


| Section | Purpose | Requirement | Reference |
|---|---|---|---|
| `[theta]` | Schema version | MUST | [format](format.md) |
| `[agent]` | Name, description, version, authors, model | MUST | [agent](agent.md) |
| `[instructions]` | System prompts, rules | SHOULD | [instructions](instructions.md) |
| `[tools.*]` | MCP server declarations | SHOULD | [tools](tools.md) |
| `[skills.*]` | Composable capability packages | SHOULD | [skills](skills.md) |
| `[[subagents]]` | Subagent definitions | SHOULD | [subagents](subagents.md) |
| `[harness.<name>]` | Harness-specific config passthrough | MAY | [harness extensions](harness-extensions.md) |
| `[extras.<name>]` | Third-party tool config | MAY | [extras](extras.md) |


Field definitions and specification can be found [here](../spec/manifest.md).
