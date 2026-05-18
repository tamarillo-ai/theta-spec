# Tools

The `[tools]` section declares [MCP](https://modelcontextprotocol.io/) servers. For normative field constraints, see the [manifest spec](../spec/manifest.md#toolsname).

---

## Fields

```toml
[tools.filesystem]
command = ["npx", "-y", "@modelcontextprotocol/server-filesystem", "./"]

[tools.osint-mcp]
command = ["uvx", "osint-mcp"]
env = { OSINT_API_KEY = "${env:OSINT_API_KEY}" }

[tools.remote-api]
url = "https://api.example.com/mcp"
headers = { Authorization = "Bearer ${env:API_KEY}" }
enabled = false
```

| Field | Type | Description |
|---|---|---|
| `command` | `arr[str]` | **REQUIRED** if `url` absent — command to start a stdio MCP server |
| `url` | `str` | **REQUIRED** if `command` absent — URL of a remote MCP server (streamable HTTP) |
| `env` | `table` (`str` --> `str`) | Environment variables for the server process |
| `headers` | `table` (`str` --> `str`) | HTTP headers sent with every MCP request (only for `url`-based tools) |
| `args` | `arr[str]` | Additional command arguments (appended after `command` entries) |
| `enabled` | `bool` | Defaults to `true` — when `false`, the tool is registered but not active |

!!! note "MCP transport names"
    The accepted transport identifiers are `stdio`, `http`, and `sse`. Implementations **MUST** accept `streamable-http` as an alias for `http` to align with the [MCP specification](https://modelcontextprotocol.io/specification/draft/basic/transports) terminology.

Each tool **MUST** have exactly one of `command` or `url`. Having both or neither is invalid.

Tool names **MUST** match `^[a-z0-9]+(-[a-z0-9]+)*$`.

## Syntax

=== "Compact (inline)"

    ```toml
    [tools]
    filesystem = { command = ["npx", "-y", "@modelcontextprotocol/server-filesystem", "./"] }
    fetch = { command = ["uvx", "mcp-fetch"] }
    ```

=== "Expanded"

    ```toml
    [tools.osint-mcp]
    command = ["uvx", "osint-mcp"]
    env = { OSINT_API_KEY = "${env:OSINT_API_KEY}" }
    enabled = true
    ```

## Environment variables

Environment variables are declared as key-value pairs in the `env` table. Values are passed through verbatim to harness-native config — theta does not resolve or interpolate them.

Harnesses that support env var interpolation (e.g. Cursor's `${env:VAR}`) will resolve the references at their own runtime. The `${env:VAR_NAME}` syntax in theta.toml is a convention for documenting which host variables a tool expects — it is not processed by theta.

!!! warning "Env values are passed through verbatim"
    theta copies env values as-is into harness-native config. If you write `env = { API_KEY = "sk-live-abc123" }`, that plaintext string ends up in `.mcp.json`, `.cursor/mcp.json`, etc. Some harnesses (Cursor) resolve `${env:VAR}` at their own runtime. Others (Claude Code, Codex) do not — the literal string `${env:VAR}` appears in the output. theta does not resolve or validate the syntax.

!!! info "`theta` implementation notes"
    `theta add tool --env KEY=VALUE` writes the pair to `theta.toml`. Keys **MUST** match POSIX format `[A-Za-z_][A-Za-z0-9_]*`, values **MUST** be non-empty.

## Headers

HTTP headers for remote (url-based) tools. Used for authentication and routing as defined in the [MCP Registry server.json](https://github.com/modelcontextprotocol/registry/blob/main/docs/modelcontextprotocol-io/remote-servers.mdx#http-headers) standard.

```toml
[tools.analytics]
url = "https://analytics.example.com/mcp"
headers = { Authorization = "Bearer ${env:ANALYTICS_API_KEY}", X-Region = "us-east-1" }
```

When adding a tool from the MCP registry, theta synthesizes headers from `remotes[].headers` metadata. Secret headers use `${env:NAME}` placeholders, same as env vars.

!!! warning "Headers on stdio tools"
    `headers` only applies to `url`-based tools. Setting headers on a `command`-based tool produces a validation warning.

!!! info "`theta` implementation notes"
    `theta add tool --header KEY=VALUE` writes the pair to `theta.toml`. For registry tools, required headers are auto-populated with `${env:NAME}` placeholders.

## Registry

Tools **MAY** be sourced from the [MCP Registry](https://registry.modelcontextprotocol.io/) using a registry reference of the form `<scope>/<name>[@<version>]`. Implementations that support registry references **SHOULD** synthesize the appropriate `command`, `url`, `env`, and `headers` fields from registry metadata.

| Package type | `runtime_hint` | Synthesized transport |
|---|---|---|
| npm | `npx` (or omitted) | `command = ["npx", "-y", "package@version"]` |
| pypi | `uvx` (or omitted) | `command = ["uvx", "package==version"]` |
| pypi | `pip` | `command = ["python", "-m", "package"]` |
| oci | any | `command = ["docker", "run", "--rm", "-i", "image:tag"]` |
| nuget | any | `command = ["dotnet", "tool", "run", "package"]` |
| remote | — | `url` + `headers` from metadata |

Env variables declared in registry metadata **SHOULD** be written as `${env:NAME}` placeholders.

!!! info "`theta` implementation notes"
    `theta add tool io.github.user/tool[@version]` fetches metadata from the MCP Registry and writes the synthesized fields to `theta.toml`. `--registry <url>` overrides the default registry URL (default: `https://registry.modelcontextprotocol.io/v0.1`). `--no-cache` bypasses the local metadata cache. Registry responses are cached at `~/.cache/theta/registry/` with a 3600-second TTL (follows XDG conventions). Transient failures (5xx, connection errors) retry up to 3 times with 500ms exponential backoff.

---

## Casting

All supported harnesses use the same MCP shape (name + command/url + env), with format differences.

| Harness | File | Format | Wrapper key | Disabled tools |
|---|---|---|---|---|
| [Claude Code](../harnesses/claude-code.md) | `.mcp.json` | JSON | `mcpServers` | Skipped |
| [Codex CLI](../harnesses/codex-cli.md) | `.codex/config.toml` | TOML | `mcp_servers` | `enabled = false` |
| [Cursor](../harnesses/cursor.md) | `.cursor/mcp.json` | JSON | `mcpServers` | Skipped |
| [GitHub Copilot](../harnesses/copilot.md) | `.vscode/mcp.json` | JSON | `servers` | Skipped |

!!! note "Casting details"
    `command[0]` is cast as `"command"` (scalar string), `command[1..]` merged with `args` becomes `"args"` (array). Codex CLI uses `enabled = false` for disabled tools; all other harnesses skip disabled tools entirely.

### Import notes

`theta cast from` reads harness-native MCP config back into `theta.toml`. Fields that theta does not model (`auth`, `envFile`, `sandboxEnabled`, `bearer_token_env_var`, etc.) round-trip through `[harness.<name>.tool.<name>]` as opaque extras — they are preserved verbatim and re-emitted on the next `cast to`. Implementations **SHOULD** emit a diagnostic hint per preserved field so the user knows which keys are non-portable.

!!! warning "Per-harness MCP fields"
    Every harness has MCP fields that `[tools]` does not model (e.g. Cursor's `auth`, Copilot's `sandboxEnabled`, Codex CLI's `bearer_token_env_var`). theta preserves them in `[harness.<name>.tool.<name>]` and emits a hint per field. These surfaces change frequently — consult each harness's upstream MCP docs:

    - [Claude Code MCP](https://code.claude.com/docs/en/settings) (`.mcp.json`)
    - [Codex CLI MCP](https://developers.openai.com/codex/mcp) (`.codex/config.toml`)
    - [Copilot MCP](https://code.visualstudio.com/docs/copilot/reference/mcp-configuration) (`.vscode/mcp.json`)
    - [Cursor MCP](https://cursor.com/docs/mcp) (`.cursor/mcp.json`)
