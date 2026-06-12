# Materialization

---

Reads `theta.lock` and populates `.theta/` with all resolved resources. See the [protocol spec](../spec/protocol.md#materialize) for normative requirements.

!!! info "`theta` CLI"
    `theta sync` locks if stale, then materializes. `theta sync --force` re-locks unconditionally.

    `THETA_OUT_DIR` redirects both `.theta/` and `theta.lock` to an alternate directory. Source files are still resolved relative to the manifest. Useful for materializing into a temporary directory without modifying the source tree:

    ```bash
    THETA_OUT_DIR=/tmp/output theta sync --manifest /real/project/theta.toml
    ```

    `theta get --output-format json` emits the complete materialized state as JSON after sync: agent identity, lock hash, system prompt, rules (with apply metadata), skills (with `SKILL.md` content and supporting files), and tools.

## `.theta/` structure

| Path | Content |
|---|---|
| `.theta/system.md` | Resolved system prompt |
| `.theta/rules/<name>.md` | Resolved rule files |
| `.theta/skills/<name>/` | Resolved skill packages (each with `SKILL.md`) |
| `.theta/subagents/<name>/theta.toml` | Resolved ref-subagent manifests |
| `.theta/subagents/<name>/rules/` | Subagent's resolved rules (recursive) |
| `.theta/subagents/<name>/skills/` | Subagent's resolved skills (recursive) |

`.theta/` is a derived artifact — **SHOULD** be gitignored.

## Sync pipeline

- **Staleness check** — compare `theta.toml` hash against `theta.lock` meta. Re-lock if stale or missing
- **Materialize** — for each locked resource, copy from source (local path, git cache, or system store) into `.theta/`
- **Content-hash skip** — if the destination file already matches the locked hash, skip the write
- **Orphan cleanup** — remove files and directories in `.theta/` that no longer appear in the lock

!!! info "`theta` CLI"
    A progress bar is shown when remote (git) sources are being fetched. The sync report prints counts of created, updated, and unchanged files.

## Determinism

Materialization **SHOULD NOT** access the network. All resources **SHOULD** be available in the local cache from the [lock](sources.md#lockfile) step, which pins exact versions, commits, and content hashes.

Implementations **MAY** re-fetch a commit already pinned in `theta.lock` when the local cache is missing or has been evicted. Such a re-fetch is byte-identical to the original and preserves reproducibility. Implementations **MUST NOT** resolve new refs (branches, tags, rev-parse expressions) during materialization — only the pinned commit SHA recorded by `lock` is allowed.

## Validation

After materialization, implementations **MUST** validate:

- Every skill directory contains a valid `SKILL.md` with at least `name` and `description`
- Every rule file exists and is non-empty
- Every subagent ref points to a valid `theta.toml`

Invalid dependencies **MUST** cause immediate failure with actionable diagnostics.
