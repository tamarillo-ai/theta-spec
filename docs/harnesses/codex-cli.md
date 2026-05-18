# Codex CLI

!!! info "Harness compatibility"
    **Last verified**: 2026-04-13  
    **Docs**: [developers.openai.com/codex](https://developers.openai.com/codex/config-reference)  
    Stale? [File an issue](https://github.com/tamarillo-ai/theta-spec/issues/new?labels=stale-harness&title=stale:+codex-cli).

## Config surface

3 config scopes: user (`~/.config/codex/config.toml`), project (`config.toml`), system. Named profiles supported. Rules: `.codex/rules/*.rules` (Starlark). Agents: `AGENTS.md`, `.codex/agents/*.toml`. Hooks: `.codex/hooks.json`. Skills: `.agents/skills/*/SKILL.md`.

Sources: [Config reference](https://developers.openai.com/codex/config-reference) · [Config basic](https://developers.openai.com/codex/config-basic) · [Config advanced](https://developers.openai.com/codex/config-advanced) · [Rules](https://developers.openai.com/codex/rules) · [Security](https://developers.openai.com/codex/agent-approvals-security) · [MCP](https://developers.openai.com/codex/mcp) · [Hooks](https://developers.openai.com/codex/hooks) · [Skills](https://developers.openai.com/codex/skills) · [Subagents](https://developers.openai.com/codex/subagents)

## Output files

| theta operation | Output | Format |
|---|---|---|
| Cast agent config | `config.toml` | TOML |
| Cast instructions | `AGENTS.md` | Markdown |
| Cast subagents | `.codex/agents/<name>.toml` | TOML |
| Cast skills | `.agents/skills/<name>/SKILL.md` (default) or `.codex/skills/<name>/SKILL.md` (when `[harness.codex].codex_specific_skills = true`) | Markdown |
| Cast harness config | `config.toml` (merged) | TOML |

## Casting notes

- Natively reads `AGENTS.md` — no rename needed.
- `exec_policy` is written as a string path value in `config.toml`, not generated as a `.rules` file.
- Supports `model_reasoning_effort` natively with extended enum: `minimal`, `low`, `medium`, `high`, `xhigh`.

## Harness-specific config


| `[harness.codex]` field | Cast target | Notes |
|---|---|---|
| `sandbox_mode` | `config.toml --> sandbox_mode` | `"read-only"` \| `"workspace-write"` \| `"danger-full-access"` — protected paths: `.git`, `.agents`, `.codex` |
| `approval_policy` | `config.toml --> approval_policy` | `"untrusted"` \| `"on-request"` \| `"never"` \| granular table |
| `web_search` | `config.toml --> web_search` | `"disabled"` \| `"cached"` (default) \| `"live"` |
| `personality` | `config.toml --> personality` | `"none"` \| `"friendly"` \| `"pragmatic"` |
| `model_reasoning_effort` | `config.toml --> model_reasoning_effort` | Extends portable enum with `"minimal"` and `"xhigh"` |
| `exec_policy` | `config.toml --> exec_policy` | Path to Starlark exec policy file |
| `features` | `config.toml --> [features]` | Feature flags: `multi_agent`, `undo`, `shell_snapshot`, `web_search`, `apps`, etc |
| `sandbox_workspace_write` | `config.toml --> [sandbox_workspace_write]` | `network_access`, `writable_roots`, `exclude_tmpdir_env_var`, `exclude_slash_tmp` |
| `model_providers` | `config.toml --> [model_providers]` | Custom providers: `name`, `base_url`, `env_key`, `wire_api`, `supports_websockets`, `auth` |
| `codex_specific_skills` | — | When `true`, cast writes skills to `.codex/skills/<name>/SKILL.md`. Default `false` (writes to `.agents/skills/<name>/SKILL.md`, which Codex CLI also discovers natively) |
| `version` | — | Semver range constraint — cast warns if installed version is outside range |

`agents`, `tui`, `history`, `analytics`, `otel`, `shell_environment_policy`, `profiles`, `projects`, and auth fields pass through without validation.

## Known limitations

- No configurable compaction — always auto.
- Named profiles (`[profiles.<name>]`) are harness-specific and not portable.
- Web search modes have no portable equivalent.

!!! info "`theta` CLI"
    `theta cast to codex-cli --notes` and `theta cast from codex-cli --notes` print the full list of known cast limitations.
