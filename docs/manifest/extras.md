# Extras

The `[extras.<name>]` section provides an extensibility mechanism for third-party tools to define their relevant configurations in the manifest.

---

## Purpose

Third-party tools store their configuration in `theta.toml` under a plugin namespace. Plugin sections are opaque to the protocol, preserved during parsing and casting, never interpreted.

```toml
[extras.awesome_memory_plugin]
backend = "chromadb"
max_results = 20

[extras.eval_framework]
tasks = ["tasks/coding.yaml", "tasks/scientific-computing.yaml"]
scorer = "llmaaj"
```

## Rules

- Extras sections **MUST** be preserved during parsing and casting — implementations **MUST NOT** drop, reorder, or modify extras contents
- Extras live under the `[extras.<name>]` namespace, so any `<name>` is structurally unambiguous. `[extras.tools]` is a separate table from top-level `[tools]` even when the inner key matches a reserved section name.
