# Harness extensions

The `[harness.<name>]` section holds harness-specific configuration that has no portable equivalent.

```toml
[harness.claude_code]
view_mode = "focus"

[harness.claude_code.sandbox]
enabled = true
network = { allowedDomains = ["*.anthropic.com"] }

[harness.github_copilot]
"chat.agent.maxRequests" = 50
"chat.tools.global.autoApprove" = false
```

---

## Rules

- `[harness.<name>]` sections are **passed through** to the target harness without interpretation by theta-spec
- When casting to harness X, only `[harness.X]` is read — all other `[harness.*]` sections are ignored
- A single `theta.toml` **MAY** contain `[harness.<name>]` sections for multiple harnesses simultaneously
- Typed fields are deserialized into typed structs and consumed during cast — all other fields are preserved as passthrough

---


## Per-harness reference

Each harness reference page documents its typed fields, passthrough fields, config surface, and casting behavior.

| Harness | TOML key | Reference |
|---|---|---|
| [Claude Code](../harnesses/claude-code.md) | `[harness.claude_code]` | [Harness-specific config](../harnesses/claude-code.md#harness-specific-config) |
| [Cursor](../harnesses/cursor.md) | `[harness.cursor]` | [Harness-specific config](../harnesses/cursor.md#harness-specific-config) |
| [GitHub Copilot](../harnesses/copilot.md) | `[harness.github_copilot]` | [Harness-specific config](../harnesses/copilot.md#harness-specific-config) |
| [Codex CLI](../harnesses/codex-cli.md) | `[harness.codex]` | [Harness-specific config](../harnesses/codex-cli.md#harness-specific-config) |
