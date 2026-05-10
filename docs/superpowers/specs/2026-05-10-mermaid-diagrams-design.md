# Mermaid Diagrams Convention Design Spec

**Date:** 2026-05-10
**Status:** Approved (in brainstorming session); style aligned with `wiki-airbnb` post-execution.
**Next step:** Hand off to writing-plans skill for implementation plan.

---

## 1. Summary

Produced vaults today have no convention for diagrams. An agent writing a wiki page that needs a diagram has nothing to consult and will likely fall back to ASCII or Unicode line-drawing inside a fenced code block. That output is hard to edit, ages poorly, and renders only as monospace text in Obsidian and on GitHub.

This spec adds a `## Diagrams` section to the vault `CLAUDE.md` template inside `RUNBOOK.md` so that produced vaults express a clear preference for ` ```mermaid ` blocks, with ASCII allowed only as an escape hatch when Mermaid can't express the structure. The section also captures a hard-won Obsidian-specific recipe for **multi-line node labels** (wide boxes, left-aligned content, bold-via-`<b>`) so agents working in produced vaults don't have to re-discover it from scratch.

## 2. Goals and non-goals

**Goals:**
- Vaults instantiated from `RUNBOOK.md` carry a "Diagrams" rule in their `CLAUDE.md`.
- The rule is auto-loaded into agent context every session (because vault `CLAUDE.md` is).
- The rule includes the Obsidian multi-line-node recipe so agents can produce wide, left-aligned, bullet-style nodes without re-walking the gotchas (`themeCSS` injection silently dropped, `htmlLabels: false` silently dropped, etc.).
- ASCII fallback is permitted but disciplined — when used, the agent must leave a one-line note explaining why so a future reader doesn't "fix" it.

**Non-goals:**
- Editing the existing ASCII diagram in `RUNBOOK.md` Part I.1 (meta-repo content, not vault content; explicitly out of scope per brainstorming session).
- Editing this meta-repo's own `CLAUDE.md` to apply the rule to specs/plans (also out of scope per brainstorming).
- Migrating diagrams in already-produced vaults. The rule applies to new authoring.
- Adding a new vendor dependency (Mermaid renders client-side in Obsidian and GitHub; nothing to install).
- Editing skills (`wiki-research`, `obsidian-skills`). The rule lives in `CLAUDE.md`, which is read once per session and applies to all write paths.
- Documenting flavors of Mermaid behavior beyond the multi-line-node case. Other quirks (custom themes, ELK layout, sequenceDiagram-specific issues) get added to the gotchas sub-section if and when they're hit in practice.

## 3. Approach

**A new `## Diagrams` section in the vault `CLAUDE.md` template**, inserted directly after the existing `## Wikilinks` section. The section has two parts:

1. A short intro paragraph stating the preference (Mermaid in fenced ` ```mermaid ` blocks; Obsidian renders natively) and a follow-on paragraph stating the ASCII escape-hatch discipline ("rare; leave a one-line note").
2. A `### Mermaid + Obsidian gotchas (multi-line node labels)` sub-section with the recipe and a generic single-node example.

Rejected alternatives:

- **Adding the rule to the `wiki-research` skill SKILL.md.** Lighter context footprint but only triggers during research-driven authoring. Misses ad-hoc page edits. Vault `CLAUDE.md` is the project-wide load point.
- **Documenting the rule in `RUNBOOK.md` Part II only (no vault-side change).** Lightest footprint but produced vaults wouldn't see it at runtime — only humans reading the runbook before instantiation.
- **Making it a hard rule in Part I.3 (Invariants).** Too strong. Mermaid can't always express what the agent needs (custom notations, layout sketches), and the runbook reserves invariants for framework-level guarantees — see Part I.3 preamble.
- **Minimal-flavor section (rule + literal-text clarifier + basic example).** This was the originally chosen design. Replaced post-execution with the wiki-airbnb-aligned style after the user confirmed they want the Obsidian gotchas captured up front rather than rediscovered each time. The "literal-text code blocks aren't diagrams" clarifier was dropped because the wiki-airbnb style trusts agents to know what a diagram is and the explicit carve-out wasn't paying its way.

## 4. The change

### 4.1 What gets edited

**One file:** `RUNBOOK.md`. **One insertion point:** inside the vault `CLAUDE.md` template at Phase 5, Task 5.1, immediately after the `## Wikilinks` block, before the `## Retrieval scopes` heading.

The vault `CLAUDE.md` template is wrapped in a 4-backtick `` ````markdown `` outer fence (see `RUNBOOK.md` near line 1424), so the inserted block — which itself contains multiple 3-backtick fences — nests cleanly without any escape gymnastics.

### 4.2 The literal text added

`````markdown
## Diagrams

Prefer Mermaid for any flowchart, sequence, state, class, or block diagram.
Use a fenced ` ```mermaid ` block — Obsidian renders these natively in
reading view and Live Preview, and they survive renames and edits better
than ASCII box-drawing art.

ASCII fallback is allowed only when Mermaid genuinely can't express the
layout (rare). When you fall back, leave a one-line note in the page
explaining why so a future reader doesn't "fix" it.

### Mermaid + Obsidian gotchas (multi-line node labels)

What we want from a node with several lines of content: **wide boxes** so
the diagram doesn't waste horizontal space, and **left-aligned content**
so bullet points line up. Mermaid's defaults give neither — boxes auto-
wrap text at ~200px and all node text is centered. Several documented
Mermaid knobs (`themeCSS` injection, `flowchart.htmlLabels: false`) are
silently dropped by Obsidian's renderer, so we can't fix this via theme
or directive configuration.

What does work, applied per node:

1. **Wider boxes** — use markdown-string labels (backtick-wrapped inside
   the quotes) and raise the wrap point at the top of the diagram:

   ```
   %%{init: {'flowchart': {'wrappingWidth': 500}}}%%
   ```

   `wrappingWidth` (default 200px) only takes effect on markdown-string
   labels. Plain HTML labels still wrap at the default.

2. **Left-aligned content** — wrap the label body in an inline
   `<div style='text-align:left'>...</div>`. Obsidian passes inline
   `style=` through to the foreignObject HTML where it strips
   stylesheet-level overrides.

3. **Bold inside the `<div>`** — CommonMark suppresses markdown parsing
   inside an HTML block, so `**TEXT**` renders as literal asterisks.
   Use `<b>TEXT</b>` instead.

Putting it together (single node):

```
A["`<div style='text-align:left'><b>NODE TITLE</b>
• first bullet point
• second bullet point</div>`"]
```

None of this matters for simple single-line labels — only reach for the
recipe when a node carries multi-line content.
`````

(The 5-backtick wrapper around the block above is just a fence-nesting trick for *this spec file*: the literal text contains 3-backtick blocks, so the spec uses a 5-backtick outer fence to hold it intact. The actual `RUNBOOK.md` edit uses normal 3-backtick fences inside the runbook's existing 4-backtick `` ````markdown `` outer fence — same nesting strategy at one level lower.)

## 5. Compatibility and risk

- **Obsidian.** Mermaid is rendered natively in Obsidian preview since v0.13 (2021). Vault portability preserved. The gotchas in §4.2 are specifically calibrated against Obsidian's renderer behavior.
- **GitHub.** Mermaid is rendered natively in `.md` files since 2022. Browse-on-GitHub UX preserved. The `wrappingWidth` directive and inline `<div style=...>` should also render correctly on GitHub (worth confirming on the next real wiki page that uses the recipe).
- **`raw/` ingested content (defuddle output).** Invariant 7 (Part I.3) already keeps `raw/` documents distinct from wiki pages. The new rule does not direct agents to rewrite ingested ASCII art; the rule applies to *authoring*, not to verbatim source text. No change needed there.
- **`obsidian-skills` submodule (kepano).** Provides `defuddle` and `obsidian-markdown` skills. The latter is a wikilink/callout/property syntax reference. Not diagram-related; no clash expected. If a future kepano update adds a contradicting opinion, the project's own `CLAUDE.md` rule wins (Claude Code's CLAUDE.md is project-scoped and loaded ahead of submodule skills).

## 6. Verification

After the runbook edit:

1. `grep -n '^## Diagrams' RUNBOOK.md` shows exactly one match, sitting between the `## Wikilinks` and `## Retrieval scopes` headings of the vault `CLAUDE.md` template.
2. `grep -n '^### Mermaid + Obsidian gotchas' RUNBOOK.md` shows exactly one match, immediately under the `## Diagrams` heading.
3. The runbook's 4-backtick outer fences still balance — count of `` ````markdown `` openings equals count of `` ```` `` closings (was 5/5 before; should remain 5/5).
4. A subsequent `RUNBOOK.md` execution against a fresh vault produces a vault `CLAUDE.md` containing the new section verbatim. (Deferred until next vault instantiation; not blocking.)

## 7. Out of scope (recap)

- The Part I.1 ASCII diagram in `RUNBOOK.md` is unchanged.
- The meta-repo's own `CLAUDE.md` is unchanged.
- No vendor list change in the `update-vendors` skill.
- No skill SKILL.md edits.

## 8. Provenance

The wording and structure of §4.2 are aligned with `~/AleCode/wiki-airbnb` commit `aede155` (2026-05-10 — "schema: document Mermaid + Obsidian recipe for multi-line node labels"). The example in the gotchas was istio-specific in `wiki-airbnb`; in the runbook template it's genericized to a domain-neutral `NODE TITLE` placeholder so vaults instantiated from the runbook don't inherit istio-flavored sample text.
