# Style guide

Conventions for theta-spec documentation style.

## General

- Headings use sentence case
- Use sentence case in lists and descriptions
- No periods at the end of list items unless the item spans multiple sentences
- Unordered lists first — numbered lists MUST be the last resort used in circumstances where there is a
- One idea per paragraph
- Use backticks for: commands, code expressions, field names, file paths, package names, any relaxed latex-ish math comment, e.g. `t_0` or `model_{theta}`, and environment variables
- Don't use em-dashes, use `-`
- Always prefer md links over bare URLs: `[name](url)`
- If a term is used a significant amount of times it SHOULD benefit from a centralized non-contextual definition in the glossary
- Never use emojis like `→`
- Math ligatures MUST be avoided

## RFC 2119

Keywords (MUST, MUST NOT, SHOULD, SHOULD NOT, MAY) carry normative meaning per [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

- Capitalize them
- Make them bold
- Do not explain what they mean

## Structure

- **Headings carry the architecture** — don't restate the heading in the first sentence
- **Tables over paragraphs** for structured data (fields, mappings, comparisons)
- **Code blocks** for anything the user might copy — MUST use syntax markers
- **Links over explanation** — if documented elsewhere avoid restating unless it is a summary followed by the link to it

## Code blocks

- All code blocks MUST have a language marker (`toml`, `console`, `json`, etc.)
- Prefer `console` with `$` prefixed commands over `bash`
- Command output SHOULD rarely be included — hard to keep current
- Use `title` for example files

## Admonitions

Use [mkdocs-material admonitions](https://squidfunk.github.io/mkdocs-material/reference/admonitions/) to add information that breaks out of the focus of the main text:

- `note` — supplementary context, background, or "good to know" asides
- `warning` — something that is not incorrect but may affect you, e.g. data loss, breaking changes, lossy casting, surprising behavior
- `info` — neutral callout for harness-specific notes, compatibility details, or implementation status
- `tip` — practical recommendation that isn't normative
- `danger` — hard constraint violation or security concern

These are starting points, not a closed list — use whatever mkdocs-material type fits. The point is to signal *why* the reader should care, not to follow a formula.

Titles MUST be informative.

## Harness-specific notes

There are two main ways to talk about harnesses and related concepts:

- **Inline admonition** — brief note in the spec page, linking to the harness reference
- **Harness reference page** — full details in `docs/harnesses/<harness>.md`

This minimizes update surface when harness behavior or configuration changes.

## Cross-references

- Every reference to another doc, concept, or field SHOULD be a markdown link
- External references MUST include the full URL
- Internal links use relative paths
- Self-contained references on this document MUST be added when
    - The concept being referred has an explicit definition on the docs and reading it disambiguates, clarifies, or provides extra context
    - Needed explanations or examples are already created elsewhere
