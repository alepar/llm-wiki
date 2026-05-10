# Mermaid Diagrams Convention Design Spec

**Date:** 2026-05-10
**Status:** Approved (in brainstorming session)
**Next step:** Hand off to writing-plans skill for implementation plan.

---

## 1. Summary

Produced vaults today have no convention for diagrams. An agent writing a wiki page that needs a diagram has nothing to consult and will likely fall back to ASCII or Unicode line-drawing inside a fenced code block. That output is hard to edit, ages poorly, and renders only as monospace text in Obsidian and on GitHub.

This spec adds a small "Diagrams" section to the vault `CLAUDE.md` template inside `RUNBOOK.md` so that produced vaults express a clear preference for ` ```mermaid ` blocks, with ASCII allowed only as an escape hatch when mermaid can't express the structure.

## 2. Goals and non-goals

**Goals:**
- Vaults instantiated from `RUNBOOK.md` carry a "Diagrams" rule in their `CLAUDE.md`.
- The rule is auto-loaded into agent context every session (because vault `CLAUDE.md` is).
- The rule includes one literal mermaid example so an agent has a concrete reference.
- The rule clarifies what is *not* a diagram (directory trees, command transcripts, frontmatter samples) so it is not misapplied to literal-text code blocks.

**Non-goals:**
- Editing the existing ASCII diagram in `RUNBOOK.md` Part I.1 (meta-repo content, not vault content; explicitly out of scope per brainstorming session).
- Editing this meta-repo's own `CLAUDE.md` to apply the rule to specs/plans (also out of scope per brainstorming).
- Migrating diagrams in already-produced vaults. The rule applies to new authoring.
- Adding a new vendor dependency (mermaid renders client-side in Obsidian and GitHub; nothing to install).
- Editing skills (`wiki-research`, `obsidian-skills`). The rule lives in `CLAUDE.md`, which is read once per session and applies to all write paths.

## 3. Approach

**A new `## Diagrams` section in the vault `CLAUDE.md` template**, inserted directly after the existing `## Wikilinks` section.

Rejected alternatives:

- **Adding the rule to the `wiki-research` skill SKILL.md.** Lighter context footprint but only triggers during research-driven authoring. Misses ad-hoc page edits. Vault `CLAUDE.md` is the project-wide load point.
- **Documenting the rule in `RUNBOOK.md` Part II only (no vault-side change).** Lightest footprint but produced vaults wouldn't see it at runtime — only humans reading the runbook before instantiation.
- **Making it a hard rule in Part I.3 (Invariants).** Too strong. Mermaid can't always express what the agent needs (custom notations, layout sketches), and the runbook reserves invariants for framework-level guarantees — see Part I.3 preamble.

## 4. The change

### 4.1 What gets edited

**One file:** `RUNBOOK.md`. **One insertion point:** inside the vault `CLAUDE.md` template at Phase 5, Task 5.1, immediately after the `## Wikilinks` block (currently lines 1482–1495, ending with `Use [label](https://...) for external links.`). The new `## Diagrams` heading goes after the existing blank line at 1496 and before the next heading `## Retrieval scopes` at line 1497.

The vault `CLAUDE.md` template is wrapped in a 4-backtick `` ````markdown `` outer fence (see `RUNBOOK.md` line 1424), so a mermaid example using ordinary 3-backtick inner fences nests cleanly. No zero-width-space escape is needed.

### 4.2 The literal text added

```markdown
## Diagrams

For diagrams — flowcharts, sequence, state, ER, class — prefer ```mermaid
blocks. Fall back to ASCII or Unicode line-drawing only when mermaid can't
express the structure. Mermaid renders natively in Obsidian preview and on
GitHub, stays diff-friendly, and remains editable by both humans and agents.

Code blocks holding literal text — directory trees, command transcripts,
sample frontmatter, file layouts — are not diagrams; leave those as plain
code blocks.

Example:

​```mermaid
flowchart TD
  A[Layer 1: Workflow prose] -->|consults| B[Layer 2: Retrieval primitives]
  B -->|invokes| C[Layer 3: qmd]
​```
```

(The two zero-width spaces in front of the inner ` ```mermaid ` and closing ` ``` ` above are an artifact of *this spec file* — to keep the inner fences from terminating this spec's outer fence. The actual `RUNBOOK.md` edit uses normal triple-backticks because it sits inside the runbook's `` ````markdown `` 4-backtick outer fence.)

## 5. Compatibility and risk

- **Obsidian.** Mermaid is rendered natively in Obsidian preview since v0.13 (2021). Vault portability preserved.
- **GitHub.** Mermaid is rendered natively in `.md` files since 2022. Browse-on-GitHub UX preserved.
- **`raw/` ingested content (defuddle output).** Invariant 7 (Part I.3) already keeps `raw/` documents distinct from wiki pages. The new rule does not direct agents to rewrite ingested ASCII art; the rule applies to *authoring*, not to verbatim source text. No change needed there.
- **`obsidian-skills` submodule (kepano).** Provides `defuddle` and `obsidian-markdown` skills. The latter is a wikilink/callout/property syntax reference per `RUNBOOK.md` line 1612. Not diagram-related; no clash expected. If a future kepano update adds a contradicting opinion, the project's own `CLAUDE.md` rule wins (Claude Code's CLAUDE.md is project-scoped and loaded ahead of submodule skills).

## 6. Verification

After the runbook edit:

1. `grep -n '^## Diagrams' RUNBOOK.md` shows exactly one match, sitting between the `## Wikilinks` and `## Retrieval scopes` headings of the vault `CLAUDE.md` template.
2. The mermaid example renders correctly when the runbook is viewed on GitHub (sanity check of the syntax, not of the runbook's output).
3. A subsequent `RUNBOOK.md` execution against a fresh vault produces a vault `CLAUDE.md` containing the new section verbatim. (Deferred until next vault instantiation; not blocking.)

## 7. Out of scope (recap)

- The Part I.1 ASCII diagram in `RUNBOOK.md` is unchanged.
- The meta-repo's own `CLAUDE.md` is unchanged.
- No vendor list change in the `update-vendors` skill.
- No skill SKILL.md edits.
