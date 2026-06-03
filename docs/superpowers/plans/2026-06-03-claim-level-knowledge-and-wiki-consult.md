# Claim-Level Knowledge + `/wiki-consult` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Edit `RUNBOOK.md` so every produced vault stores facts as provenanced, supersedable **claims** (across entity/concept/synthesis pages) and ships a read-only `/wiki-consult` skill that returns trust-ranked, freshness- and contradiction-aware answers.

**Architecture:** Every change is an edit to the single-file `RUNBOOK.md` artifact (the meta-repo's only deliverable). Newly-instantiated vaults inherit the changes; existing vaults degrade gracefully (forward-only). There is **no executable test suite** — the artifact is markdown. "Verification" for each task is therefore a `grep`/`Read` structural check with an expected result, plus a final self-review of the whole runbook against the spec. Commits are per-logical-change (the runbook's own "one commit per phase" rule governs *produced vaults*, not edits to the runbook).

**Tech Stack:** Markdown editing only. Tools: `Read`, `Edit`, `Bash` (`grep`, `git`). No code, no package manager, no qmd at edit time.

**Spec:** `docs/superpowers/specs/2026-06-03-claim-level-knowledge-and-wiki-consult-design.md`

---

## Locked design decisions

The spec leaves a few low-level details under-specified. These are resolved here so the engineer does not re-derive them mid-task. They are internally consistent and lint-deterministic; if the user disagrees with any, change it in one place and propagate.

1. **Anchor / key / pointer caret convention.**
   - Body bullets carry the Obsidian anchor *with* a caret: `... ^c-db-engine`.
   - `claims:` map keys are the anchor id *without* the caret: `c-db-engine:`.
   - `superseded_by:` same-page pointer is the bare anchor id (no caret): `superseded_by: c-db-engine`.
   - `superseded_by:` cross-page pointer: `qmd://<vault-name>/<path>#c-<slug>` (the `#fragment` is the bare anchor id).
   - This makes the lint join (key ↔ body anchor) a deterministic string match after stripping the leading `^`.

2. **Empty `claims:` map.** A page with no provenanced facts yet carries `claims: {}`. Templates ship this. Lint treats a *missing* `claims:` key (old vaults) as equivalent to `claims: {}` — never a hard error (forward-only migration, spec §7).

3. **`claims:` is not hard-required by lint.** New pages from templates always have it; old pages without it are grandfathered. Lint enforces the *join* and *per-entry field completeness* (spec §6), not the presence of the map.

4. **Synthesis keeps its page-level `superseded_by:`.** Spec §3 says synthesis "only loses write-once" and keeps `question`, `answered_at`, the 180-day rule. Its existing page-level `superseded_by:` frontmatter field (II.1 / II.2 / lint pass 5) is **retained** for *wholesale* page replacement (one synthesis replaced by a newer one). *Individual* conclusion-claim drift within a living synthesis uses the new claim-level `superseded_by:` inside `claims:`. Both mechanisms coexist; write-once is the only thing removed.

5. **Controlled vocabularies.** `by ∈ {human, wiki-research, deep-research, import}`; `confidence ∈ {low, medium, high}`.

6. **`wiki-consult` placement.** Inserted as new **Task 3.5** (before the Phase 3 commit), commit renumbered **3.5 → 3.6**. This avoids renumbering Tasks 3.2/3.3/3.4, which are cross-referenced in the runbook's self-review section. `wiki-consult`'s `SKILL.md` holds the **shared retrieval + trust-ranking core**; `wiki-research`'s playbook delegates to it by reference (no separate playbook file for wiki-consult).

---

## File touched

Only one file changes: `RUNBOOK.md` (2285 lines). The edits, in document order:

| Region | Lines (pre-edit) | Task |
|---|---|---|
| I.3 Invariant 3 (write-once) | 81 | 1 |
| II.1 page-type table | 95–101 | 1 |
| II.2 frontmatter contracts (+ new Claims subsection) | 103–139 | 2 |
| II.6 supersession (rewrite) | 167–177 | 3 |
| Phase 2 templates (entity/concept/synthesis) | 514–589 | 4 |
| Phase 3: new wiki-consult task + commit renumber + Phase 3 table + II.8 note | 202, 243, 667, 1175–1182 | 5 |
| wiki-research `SKILL.md` | 689–746 | 6 |
| wiki-research `playbook.md` (Phase 2, 6, 7) | 790–1021 | 7 |
| Vault `CLAUDE.md`: Layout / Required frontmatter / new Claims section / Synthesis section / Skills | 1441, 1473–1480, 1610–1615, 1654 | 8 |
| Vault `CLAUDE.md`: Lint passes | 1785–1802 | 9 |
| Consistency sweep: commit msgs, hand-off, intro, commit-graph list | 3, 1181, 2156, 2202 | 10 |

---

## Task 0: Create the working branch

**Files:** none (git only).

- [ ] **Step 1: Branch off main**

```bash
git -C /home/debian/AleCode/llm-wiki checkout -b claim-level-knowledge
```

- [ ] **Step 2: Confirm clean starting point**

Run: `git -C /home/debian/AleCode/llm-wiki status`
Expected: `On branch claim-level-knowledge` / `nothing to commit, working tree clean`.

---

## Task 1: Drop write-once from the invariant and page-type table

Removes page-level write-once and adds `claims` to the page-type contracts. Two edits in `RUNBOOK.md`, one commit.

**Files:**
- Modify: `RUNBOOK.md:81` (Invariant I.3.3)
- Modify: `RUNBOOK.md:95-101` (II.1 page-type table)

- [ ] **Step 1: Rewrite Invariant I.3.3**

Replace this line:

```
3. **Synthesis pages are write-once.** Never edit an existing synthesis page to change the answer. Write a new synthesis at a new slug; set the old page's `superseded_by:` to point at the new one.
```

with:

```
3. **Facts are claims; supersession is claim-level and non-destructive.** Every provenanced fact on an entity, concept, or synthesis page is a *claim*: a block-anchored body bullet (`^c-…`) backed by a `claims:` frontmatter entry (Part II.2). A claim is never edited to reverse its meaning and never deleted — it is *superseded* (Part II.6): a new claim is written and the old claim's `superseded_by:` is pointed at it. Synthesis pages are **no longer write-once** — their answers are conclusion-claims that supersede the same way (a synthesis may also be wholesale-replaced via its page-level `superseded_by:`). Un-anchored body bullets are valid un-provenanced notes, not claims.
```

- [ ] **Step 2: Rewrite the II.1 page-type table**

Replace these three table rows (lines 97–99):

```
| `entity` | A specific proper noun: a tool, a person, an organization, a model, a product, a dataset, a place. | `type`, `kind`, `date_updated` |
| `concept` | A *kind of thing*: an abstraction, a technique, a category, a methodology. | `type`, `date_updated` |
| `synthesis` | An answer to a specific question, citing other pages and external sources. **Write-once.** | `type`, `question`, `answered_at`, `superseded_by`, `sources`, `date_updated` |
```

with:

```
| `entity` | A specific proper noun: a tool, a person, an organization, a model, a product, a dataset, a place. | `type`, `kind`, `claims`, `date_updated` |
| `concept` | A *kind of thing*: an abstraction, a technique, a category, a methodology. | `type`, `claims`, `date_updated` |
| `synthesis` | An answer to a specific question, citing other pages and external sources. Freely editable; answers are conclusion-claims (supersede, don't reverse-in-place). | `type`, `question`, `answered_at`, `superseded_by`, `sources`, `claims`, `date_updated` |
```

- [ ] **Step 3: Verify**

Run: `grep -n "write-once\|Write-once" RUNBOOK.md`
Expected: only the **wiki-research SKILL.md** line (line ~702) still matches (fixed in Task 6); lines 81 and 99 no longer match. Confirm exactly one remaining match in the 690–710 range.

Run: `grep -c "claims" RUNBOOK.md`
Expected: ≥ 3 (the three table rows now mention `claims`).

- [ ] **Step 4: Commit**

```bash
git add RUNBOOK.md
git commit -m "spec: drop synthesis write-once; claims in invariant + page-type table"
```

---

## Task 2: Add the claim data model to II.2 frontmatter contracts

Documents the `claims:` map shape, the body-anchor join, controlled vocab, and graceful degradation (spec §3).

**Files:**
- Modify: `RUNBOOK.md:139` (insert a new subsection after the synthesis frontmatter contract paragraph)

- [ ] **Step 1: Insert the Claims subsection**

After this existing paragraph (line 139):

```
The synthesis frontmatter is contractual: the lint pass (Part III Phase 5) checks all five required fields are present and well-formed, and `superseded_by:` resolves to a real synthesis page in the vault when set.
```

insert the following block (a new `### Claims (all page types)` subsection, before `## II.3 Wikilinks`):

````
### Claims (all page types)

Every *provenanced fact* on an entity, concept, or synthesis page is a **claim**: a body bullet carrying an Obsidian block anchor (`^c-<slug>`), joined to a `claims:` frontmatter map keyed by that same anchor. The frontmatter holds metadata only; the body bullet is the single source of truth for the claim's wording (no text-mirror — keeps it DRY; lint enforces the join). A page with no provenanced facts yet carries an empty map: `claims: {}`.

```yaml
claims:
  c-db-engine:
    sources:                                 # per-fact provenance — never empty
      - https://internal-wiki/adr-014
      - qmd://<vault-name>/raw/migration-postmortem.md
    by: wiki-research                         # human | wiki-research | deep-research | import
    asserted_at: 2026-06-03                   # YYYY-MM-DD
    confidence: high                          # low | medium | high
    superseded_by: null                       # null | <anchor-id> | qmd://<vault-name>/<path>#<anchor-id>
```

```markdown
## Facts
- Uses DynamoDB for the primary store ^c-db-engine
```

**Anchor / key / pointer convention.** The body bullet carries the anchor *with* its caret (`^c-db-engine`). The `claims:` map key and the `superseded_by:` same-page pointer use the bare anchor id *without* the caret (`c-db-engine`). A cross-page pointer is `qmd://<vault-name>/<path>#c-<slug>`. Lint joins key ↔ body anchor by stripping the leading `^`.

**Field contract** (every claim entry has all five):

- `sources` — non-empty list. `qmd://<vault-name>/<path>` for in-vault pages and `raw/` documents; `https://...` for non-ingested external sources.
- `by` — write-identity, one of `human | wiki-research | deep-research | import`. When `wiki-research` writes a fact grounded in a deep-research run, `by:` is `wiki-research` and the deep-research evidence appears in `sources:`. Direct human edits are `human`.
- `asserted_at` — the date the claim was asserted (`YYYY-MM-DD`).
- `confidence` — `low | medium | high`.
- `superseded_by` — `null` while the claim is live; otherwise the successor's bare anchor id (same-page) or `qmd://<vault-name>/<path>#c-<slug>` (cross-page).

**Graceful degradation.** A body bullet with no `^anchor` (and thus no `claims:` entry) is a valid **un-provenanced note**, not a lint error — it simply carries no provenance and reads as lower-trust in consult output (II.8). Existing vaults' free-text bullets remain valid, and a page lacking the `claims:` key altogether is treated as `claims: {}`. Pages upgrade to claims opportunistically when next edited (forward-only migration — see II.6).
````

- [ ] **Step 2: Verify**

Run: `grep -n "Claims (all page types)\|asserted_at\|c-db-engine" RUNBOOK.md`
Expected: the new subsection heading appears once in the II.2 region (~line 140), and `asserted_at` / `c-db-engine` appear inside it.

Run: `grep -n "## II.3 Wikilinks" RUNBOOK.md`
Expected: still present, now after the inserted subsection (confirm the insert landed before II.3).

- [ ] **Step 3: Commit**

```bash
git add RUNBOOK.md
git commit -m "spec: document the claim data model in II.2 frontmatter contracts"
```

---

## Task 3: Rewrite II.6 to claim-level supersession

Replaces page-level synthesis supersession with the uniform claim-level mechanism and the `## Superseded` convention (spec §4, §7).

**Files:**
- Modify: `RUNBOOK.md:167-177` (entire II.6 body, heading included)

- [ ] **Step 1: Replace the II.6 section**

Replace this block (lines 167–177):

```
## II.6 Synthesis supersession

To update a synthesis answer:

1. Write a new synthesis page with the updated answer at a new slug.
2. Set the old page's `superseded_by:` to the new page's `qmd://<vault-name>/<path>`.
3. Mention the supersession in the new page's body ("supersedes [[old-slug]] — what changed: …").

Synthesis history stays queryable; no silent semantic drift.

Entity and concept pages, by contrast, are freely editable in place. Bump `date_updated` on every meaningful edit.
```

with:

````
## II.6 Claim-level supersession

Facts are never destroyed or reversed in place. To change what a page asserts, **supersede the claim** — uniform across entity, concept, and synthesis pages:

1. Write the new claim as a normal bullet + anchor + `claims:` entry in the live section (`## Facts`, or a synthesis's answer body).
2. Set the old claim's `superseded_by:` to the successor's pointer — the bare anchor id (`c-<slug>`) for a same-page successor, or `qmd://<vault-name>/<path>#c-<slug>` for a cross-page one (e.g. synthesis → synthesis).
3. Move the old bullet into the page's `## Superseded` section: strike it through and add a one-line reason and pointer.

```markdown
## Facts
- Uses DynamoDB for the primary store ^c-db-engine

## Superseded
- ~~Uses Postgres for the primary store~~ ^c-db-engine-old
  (superseded by ^c-db-engine, 2026-06-03 — migrated off RDS, see ADR-014)
```

Live sections (`## Facts`, synthesis answer body) show only current truth; superseded claims stay in-page and remain qmd-queryable, so "what did we believe, when, and why did it change" is preserved without forking pages or relying on git archaeology.

**All three page types are freely editable** for corrections, additions, and wording fixes — bump `date_updated` on every meaningful edit. What is *not* allowed is silently reversing a fact's meaning: a material answer change is a supersession, not an in-place edit. Synthesis pages are no longer write-once; a synthesis's answer is one or more conclusion-claims that supersede the same way. A synthesis may additionally be **wholesale-replaced** by pointing its page-level `superseded_by:` (II.2) at a newer synthesis; use claim-level supersession for finer-grained drift within a living synthesis. (`question`, `answered_at`, and the 180-day staleness rule are unchanged.)

**Forward-only migration.** These rules govern newly-instantiated vaults and all new writes in existing vaults. Old free-text bullets without anchors remain valid un-provenanced notes (II.2); no bulk retrofit is required or performed. Pages upgrade to claims opportunistically when next edited.
````

- [ ] **Step 2: Verify**

Run: `grep -n "## II.6" RUNBOOK.md`
Expected: `## II.6 Claim-level supersession`.

Run: `grep -n "## Superseded\|freely editable\|Forward-only migration" RUNBOOK.md`
Expected: the `## Superseded` example, the "All three page types are freely editable" sentence, and "Forward-only migration" all appear in the II.6 region. The old "Entity and concept pages, by contrast, are freely editable in place." line is gone.

- [ ] **Step 3: Commit**

```bash
git add RUNBOOK.md
git commit -m "spec: rewrite II.6 to claim-level supersession across all page types"
```

---

## Task 4: Update the Phase 2 page templates

Adds `claims: {}` plus `## Facts` / `## Superseded` sections to the three templates so new pages are claim-ready (spec §9, Phase 2 templates).

**Files:**
- Modify: `RUNBOOK.md:514-532` (entity template)
- Modify: `RUNBOOK.md:540-553` (concept template)
- Modify: `RUNBOOK.md:561-589` (synthesis template)

- [ ] **Step 1: Update the entity template**

Replace the entity template body (the fenced block at lines 514–532):

```
---
type: entity
kind: tool                 # person | org | tool | model | repo | dataset | product | place | <other>
aliases: []
homepage:
date_updated: YYYY-MM-DD
---

# <Entity Name>

One-paragraph description.

## Properties
- Producer: [[<Org>]]
- ...

## Notes
```

with:

```
---
type: entity
kind: tool                 # person | org | tool | model | repo | dataset | product | place | <other>
aliases: []
homepage:
claims: {}                 # map of ^c-<slug> anchor → {sources, by, asserted_at, confidence, superseded_by}
date_updated: YYYY-MM-DD
---

# <Entity Name>

One-paragraph description.

## Properties
- Producer: [[<Org>]]
- ...

## Facts
<!-- Each provenanced fact is an anchored bullet with a matching claims: entry. Example:
- Uses DynamoDB for the primary store ^c-db-engine
Un-anchored bullets are valid un-provenanced notes. -->

## Notes

## Superseded
<!-- Struck-through claims moved here when superseded (see II.6). Empty until needed. -->
```

- [ ] **Step 2: Update the concept template**

Replace the concept template body (lines 540–553):

```
---
type: concept
confidence: medium         # low | medium | high
related: []
date_updated: YYYY-MM-DD
---

# <Concept Name>

One-paragraph definition.

## Details
```

with:

```
---
type: concept
confidence: medium         # low | medium | high  (page-level default; per-claim confidence lives in claims:)
related: []
claims: {}                 # map of ^c-<slug> anchor → {sources, by, asserted_at, confidence, superseded_by}
date_updated: YYYY-MM-DD
---

# <Concept Name>

One-paragraph definition.

## Details

## Facts
<!-- Each provenanced fact is an anchored bullet with a matching claims: entry.
Un-anchored bullets are valid un-provenanced notes. -->

## Superseded
<!-- Struck-through claims moved here when superseded (see II.6). Empty until needed. -->
```

- [ ] **Step 3: Update the synthesis template**

Replace the synthesis template body (the fenced block at lines 561–589):

```
---
type: synthesis
question: "<the question, verbatim>"
answered_at: YYYY-MM-DD
superseded_by: null
sources: []
date_updated: YYYY-MM-DD
---

# <The question>

Short answer first (1–3 sentences). State the bottom line directly.

## Reasoning

<Detailed reasoning. Use [[wikilinks]] to entities/concepts. Cite web sources
inline as [n] with a footnote section, or with the URL in parentheses on first
reference.>

## Open questions

<Anything not resolved.>

## Detailed report

<Optional. Relative path to the deep-research artifact under `.research/...`
if the user wants the long-form linked.>
```

with:

```
---
type: synthesis
question: "<the question, verbatim>"
answered_at: YYYY-MM-DD
superseded_by: null        # page-level: set to qmd://<vault-name>/<path> if wholesale-replaced
sources: []
claims: {}                 # conclusion-claims: ^c-<slug> anchor → {sources, by, asserted_at, confidence, superseded_by}
date_updated: YYYY-MM-DD
---

# <The question>

Short answer first (1–3 sentences). State the bottom line directly. Express each
load-bearing conclusion as an anchored claim with a matching claims: entry, e.g.:

- The best ratio is 10:1 ^c-conclusion

## Reasoning

<Detailed reasoning. Use [[wikilinks]] to entities/concepts. Cite web sources
inline as [n] with a footnote section, or with the URL in parentheses on first
reference.>

## Open questions

<Anything not resolved.>

## Superseded
<!-- Struck-through conclusion-claims moved here when superseded (see II.6). Empty until needed. -->

## Detailed report

<Optional. Relative path to the deep-research artifact under `.research/...`
if the user wants the long-form linked.>
```

- [ ] **Step 4: Verify**

Run: `grep -c "claims: {}" RUNBOOK.md`
Expected: ≥ 3 (one per template; more if later tasks add examples — this is the first three).

Run: `grep -n "## Facts\|## Superseded" RUNBOOK.md`
Expected: `## Facts` in the entity and concept templates; `## Superseded` in all three templates.

- [ ] **Step 5: Commit**

```bash
git add RUNBOOK.md
git commit -m "scaffold: add claims stub + Facts/Superseded sections to page templates"
```

---

## Task 5: Add the `wiki-consult` skill (new Phase 3 task)

Inserts the read-only consult skill as **Task 3.5**, renumbers the commit task to **3.6**, updates the Phase 3 table, the commit message, and the II.8 scope note (spec §5, §9).

**Files:**
- Modify: `RUNBOOK.md:243` (Phase 3 table row)
- Modify: `RUNBOOK.md:202` (II.8 implementation note)
- Modify: `RUNBOOK.md:1175` (renumber commit heading 3.5 → 3.6, after inserting 3.5)
- Insert: new `### Task 3.5 — .claude/skills/wiki-consult/SKILL.md` between Task 3.4 (ends line 1173) and the commit task

- [ ] **Step 1: Update the Phase 3 table row**

Replace line 243:

```
| 3 | Committed skills | `.claude/skills/{recall,wiki-research,update-vendors}/...` |
```

with:

```
| 3 | Committed skills | `.claude/skills/{recall,wiki-consult,wiki-research,update-vendors}/...` |
```

- [ ] **Step 2: Add the II.8 wiki-consult note**

Replace this sentence in II.8 (line 202):

```
Implementation: scopes are realized via qmd path filters. Consult `qmd query --help` for current parameter names; the vault `CLAUDE.md` retrieval-primitives section documents the working syntax. The wiki-research skill scopes phase-2 retrieval per its playbook. The recall skill exposes scope as a flag for debugging.
```

with:

```
Implementation: scopes are realized via qmd path filters. Consult `qmd query --help` for current parameter names; the vault `CLAUDE.md` retrieval-primitives section documents the working syntax. The wiki-consult skill uses `hybrid` scope — curated claims are presented as *the answer*, raw-doc hits as *supporting evidence* — and is the shared retrieval core that wiki-research's phase-2 delegates to. The recall skill exposes scope as a flag for debugging.
```

- [ ] **Step 3: Renumber the commit task heading**

Replace line 1175:

```
### Task 3.5 — Commit skills
```

with:

```
### Task 3.6 — Commit skills
```

- [ ] **Step 4: Insert the wiki-consult task**

Immediately **before** the (now) `### Task 3.6 — Commit skills` heading, insert the following new task. (It sits after the close of Task 3.4's `update-vendors` failure-modes fenced block, line ~1173.)

`````
### Task 3.5 — `.claude/skills/wiki-consult/SKILL.md`

- [ ] **Step 1: Create directory and file**

```bash
mkdir -p .claude/skills/wiki-consult
```

Write `<vault-root>/.claude/skills/wiki-consult/SKILL.md` verbatim:

````markdown
---
name: wiki-consult
description: Use mid-task when you need to know what the vault already knows about something — a fast, read-only, trust-ranked, provenance-annotated answer drawn only from existing vault pages and raw sources. Never writes, never supersedes, never hits the web. For research that fills gaps and writes pages use wiki-research; for raw qmd debugging use recall.
---

# /wiki-consult — read-only vault consult

Answers "what does the vault already know about X?" from existing content only.
This is the vault's shared **read core**: wiki-research delegates its retrieval
and trust-ranking phase here, then wraps it with web research and write
capability. wiki-consult itself never writes, never supersedes, never ingests,
and never hits the web.

## Which read path

- **wiki-consult** — answer-shaped, trust-ranked, provenance-aware. The default
  for "does the vault already cover this?"
- **recall** — raw qmd query for debugging retrieval and scores. No ranking,
  no provenance synthesis.
- **wiki-research** — when the answer needs fresh web research and new pages
  written. wiki-consult *suggests* escalating here on a miss; it never runs it.

## Properties

- **Read-only.** No writes, no supersession, no ingest, no `qmd update`/`embed`,
  no web.
- **Manual.** Invoked as `/wiki-consult <question>`. No auto-trigger.
- **Hybrid scope.** Queries curated pages *and* `raw/`. Curated claims are the
  answer; raw-doc hits are supporting evidence (so a topic with no claim yet
  still returns something).
- **Trust-ranked.** synthesis > entity/concept > raw (the vault trust
  hierarchy). Superseded claims (`superseded_by:` not null) are skipped.
- **Freshness-aware.** Uses the vault's 180-day threshold for both synthesis
  `answered_at` and claim `asserted_at`. Stale items are warned, never hidden.
- **Contradiction-aware.** If two live claims conflict, surface both with their
  provenance and flag — do not resolve.
- **Gap-reporting.** On a miss, stale answer, or contradiction, suggest
  `/wiki-research` but never run it.

## Procedure (the shared retrieval + trust-ranking core)

wiki-research's playbook Phase 2 delegates to these six steps; keep them here as
the single retrieval path.

1. **Retrieve (hybrid).** Run a hybrid qmd query — MCP `mcp__qmd__query`
   preferred (`searches: [{type:'lex', query:'<terms>'}, {type:'vec',
   query:'<question phrased naturally>'}]`), CLI fallback `qmd query
   --collection <vault-name> --json "<query>"` — over the whole vault. On qmd
   unavailable, fall back to Read+Grep and surface the degraded mode once.
2. **Bucket by trust tier** (read each hit's frontmatter `type:`):
   - Tier 1 — `type: synthesis`, page-level `superseded_by: null`.
   - Tier 2 — `type: entity` or `type: concept`.
   - Tier 3 — `raw/` documents (supporting evidence only).
3. **Resolve claims.** For tier 1–2 hits, read the full file. Read each fact
   from its `## Facts` (or synthesis answer) bullet and join to its `claims:`
   entry by anchor (strip the leading `^`). **Skip any claim whose
   `superseded_by:` is not null.** Un-anchored bullets are un-provenanced
   notes — include them but mark them lower-trust (no confidence/source/date).
4. **Freshness.** For each surfaced synthesis and claim, compute age from
   `answered_at` / `asserted_at`. If `(today - date) > 180` days, prefix the
   line with `⚠` and an age note. Never drop it.
5. **Contradiction.** If two live claims assert conflicting values for the same
   fact, surface both with provenance and a `⚠ Conflict` flag. Do not resolve.
6. **Report.** Present curated claims as the answer (confidence, asserted date,
   `by`, source, `[[page]]`), raw hits as supporting evidence, a gaps line for
   anything with no vault entry, then the escalation prompt if there were any
   gaps, stale items, or conflicts.

## Output shape (illustrative)

```
From the vault:
• Primary store: DynamoDB — confidence: high, asserted 2026-06-03 by wiki-research
  source: ADR-014  [[acme-infra]]
• ⚠ Auth: Auth0 — confidence: medium, asserted 2025-08-01 (10mo old — may be stale)  [[acme-auth]]
• ⚠ Conflict: [[acme-infra]] says us-east-1, [[acme-dr-plan]] says us-west-2 — both live

No vault entry for: rate-limiting strategy.
→ Gaps/stale found. Escalate with /wiki-research?
```

## Hard rules

- Never write, edit, supersede, ingest, or run `qmd update` / `qmd embed`.
- Never hit the web. If the vault can't answer, say so and suggest
  `/wiki-research`.
- Skip superseded claims from the answer — they are history, not current truth.
  To inspect history, the human can use `/recall`.
- Relationship to `recall` is unchanged: `recall` is the raw qmd query;
  `wiki-consult` is the answer-shaped, trust-ranked read path. No merge.
````
`````

- [ ] **Step 5: Verify**

Run: `grep -n "### Task 3.5\|### Task 3.6\|name: wiki-consult" RUNBOOK.md`
Expected: `### Task 3.5 — .../wiki-consult/SKILL.md`, then `### Task 3.6 — Commit skills`, and `name: wiki-consult` inside the inserted block. There must be **no** remaining `### Task 3.5 — Commit skills`.

Run: `grep -n "recall,wiki-consult,wiki-research" RUNBOOK.md`
Expected: the Phase 3 table row (line ~243).

- [ ] **Step 6: Commit**

```bash
git add RUNBOOK.md
git commit -m "spec: add wiki-consult read-only consult skill (Phase 3 Task 3.5)"
```

---

## Task 6: Refactor the wiki-research `SKILL.md`

Removes the write-once rule, notes that superseded claims are skipped in the trust hierarchy, and states that retrieval delegates to wiki-consult (spec §6).

**Files:**
- Modify: `RUNBOOK.md:693-702` (Trust hierarchy block)
- Modify: `RUNBOOK.md:715-718` (Workflow step 2)

- [ ] **Step 1: Update the trust hierarchy + write-once line**

Replace this block (lines 689–702):

```
## Trust hierarchy

When weighing evidence:

1. **Synthesis pages** (`type: synthesis`, not superseded) — validated answers,
   highest trust.
2. **Entity / concept pages** (`type: entity` or `type: concept`) — curated,
   second.
3. **Raw articles** (`https://...` already in qmd) — hand-picked, third.
4. **Fresh web research** — useful for filling gaps and freshness checks, but
   always cross-checked against the above.

Never silent-edit an existing page. Every write requires explicit user
approval. Synthesis pages are write-once: never edit; supersede.
```

with:

```
## Trust hierarchy

When weighing evidence:

1. **Synthesis pages** (`type: synthesis`, not superseded) — validated answers,
   highest trust.
2. **Entity / concept pages** (`type: entity` or `type: concept`) — curated,
   second.
3. **Raw articles** (`https://...` already in qmd) — hand-picked, third.
4. **Fresh web research** — useful for filling gaps and freshness checks, but
   always cross-checked against the above.

Within a page, weigh **live claims** only — skip any claim whose `superseded_by:`
is not null. Un-anchored bullets are un-provenanced notes; treat them as
lower-trust than provenanced claims.

Never silent-edit an existing page. Every write requires explicit user approval.
Facts are claims: never reverse a fact in place — supersede it (claim-level,
across entity/concept/synthesis alike; see the vault `CLAUDE.md` and RUNBOOK
II.6).
```

- [ ] **Step 2: Update Workflow step 2 to name the delegation**

Replace this Workflow item (lines 715–718):

```
2. Run `qmd query` (MCP preferred, CLI fallback) and bucket results by
   frontmatter `type:` into the trust tiers above.
```

with:

```
2. Retrieve + trust-rank via the **wiki-consult** read core (`.claude/skills/
   wiki-consult/SKILL.md`, "Procedure"): one hybrid `qmd query`, bucketed by
   frontmatter `type:` into the trust tiers above, claims resolved, superseded
   claims skipped. This is the single retrieval path — do not reimplement it.
```

- [ ] **Step 3: Verify**

Run: `grep -n "write-once\|Write-once" RUNBOOK.md`
Expected: **no matches** anywhere in the file.

Run: `grep -n "wiki-consult read core\|superseded_by:.*is not null" RUNBOOK.md`
Expected: the delegation note and the skip-superseded note both appear in the wiki-research SKILL region (~690–720).

- [ ] **Step 4: Commit**

```bash
git add RUNBOOK.md
git commit -m "spec: wiki-research SKILL delegates retrieval, drops write-once, skips superseded claims"
```

---

## Task 7: Refactor the wiki-research `playbook.md`

Makes Phase 2 delegate to the wiki-consult core, generalizes Phase 6 supersession to claim-level across all types, and makes Phase 7 emit claims (spec §6 "Phase 7 write updates").

**Files:**
- Modify: `RUNBOOK.md:790-814` (Phase 2)
- Modify: `RUNBOOK.md:940-952` (Phase 6 resolution outcomes)
- Modify: `RUNBOOK.md:958-966` (Phase 7a)
- Modify: `RUNBOOK.md:982-997` (Phase 7c frontmatter + answer body intro)

- [ ] **Step 1: Replace Phase 2 with a delegation to the wiki-consult core**

Replace this block (lines 790–814):

```
## Phase 2 — qmd-first retrieval

Run a hybrid query against the vault:

- MCP preferred: `mcp__qmd__query` with
  `searches: [{type:'lex', query:'<terms>'}, {type:'vec', query:'<the question phrased naturally>'}]`
  and `intent: '<one-line description of why you're searching>'`.
- CLI fallback: `qmd query --collection <vault-name> --json "<query>"`.

Bucket results by frontmatter `type:` (read the frontmatter of each hit; the
chunk text alone is not enough):

| Tier | Bucket | Trust |
|------|-----|-------|
| 1 | `type: synthesis`, `superseded_by: null` | highest |
| 2 | `type: entity` or `type: concept` | high |
| 3 | external URL hits (qmd-ingested web sources) | medium |

For tier 1 and tier 2 results, read the **full file** from disk (use `qmd get
<path>` or `Read`). Read frontmatter and body. For tier 3, the chunk text in
the search result is enough at this stage — read full pages later if a claim
depends on context not in the chunk.

If the query returns no results, treat it as a green-field topic and skip
directly to phase 4.
```

with:

```
## Phase 2 — Retrieval (via the wiki-consult core)

Retrieval and trust-ranking are the **shared wiki-consult read core** — do not
reimplement them. Run steps 1–5 of `.claude/skills/wiki-consult/SKILL.md`
("Procedure"): one hybrid `qmd query` (MCP `mcp__qmd__query` preferred, CLI
`qmd query --collection <vault-name> --json` fallback), bucketed by frontmatter
`type:` into the trust tiers, with full-file reads for tier 1–2, claims
resolved by anchor, superseded claims skipped, and freshness/contradiction
flags computed.

| Tier | Bucket | Trust |
|------|-----|-------|
| 1 | `type: synthesis`, page-level `superseded_by: null` | highest |
| 2 | `type: entity` or `type: concept` | high |
| 3 | `raw/` documents / external URL hits | medium |

What wiki-research does *beyond* the read core: carry the bucketed evidence and
flagged gaps forward into the coverage check (Phase 3) and seed brief (Phase 4),
and — unlike read-only consult — continue into web research and writes.

If the query returns no results, treat it as a green-field topic and skip
directly to phase 4.
```

- [ ] **Step 2: Generalize the Phase 6 "Supersede" outcome**

Replace these two resolution bullets (lines 943–949):

```
- **Trust web** — supersede the affected synthesis page (write a new one,
  set old `superseded_by:`) or update the affected entity/concept page (with
  diff approval; phase 7).
- **Mark uncertain** — capture in synthesis under `## Open questions`; on
  the affected concept page, lower `confidence:` to `low` and note the
  disagreement.
- **Supersede** — only valid for synthesis pages; never directly edit them.
```

with:

```
- **Trust web** — supersede the affected claim (claim-level, II.6): write the
  corrected claim with its `claims:` entry, point the old claim's
  `superseded_by:` at it, and move the old bullet to `## Superseded` (phase 7,
  with diff approval). Applies to entity, concept, and synthesis claims alike.
- **Mark uncertain** — capture in the synthesis under `## Open questions`; on
  the affected claim, lower its `confidence:` to `low` and note the
  disagreement.
- **Supersede** — claim-level supersession (II.6) is the mechanism for every
  page type. A whole synthesis may also be wholesale-replaced via its
  page-level `superseded_by:`.
```

- [ ] **Step 3: Make Phase 7a emit claims**

Replace the Phase 7a steps (lines 958–966):

```
For each entity/concept page touched by the research:

1. Read the current file.
2. Compose the proposed edit (typically: refine description, add a property,
   link a new related concept, refresh `date_updated`).
3. Show the unified diff to the user.
4. On approval, write the file and update `date_updated` to today.
```

with:

```
For each entity/concept page touched by the research:

1. Read the current file.
2. Compose the proposed edit. For each **new provenanced fact**, add an
   anchored bullet under `## Facts` (`... ^c-<slug>`) and a matching `claims:`
   entry: `sources` (the web/vault evidence), `by: wiki-research`,
   `asserted_at: <today>`, `confidence`, `superseded_by: null`. To **change an
   existing fact**, follow the II.6 supersession flow — never overwrite the old
   bullet. Non-fact edits (refine prose, add a property, link a related
   concept) stay free-text.
3. Show the unified diff to the user.
4. On approval, write the file and update `date_updated` to today.
```

- [ ] **Step 4: Make Phase 7c emit a conclusion-claim**

Replace the Phase 7c frontmatter block (lines 986–997):

```
```yaml
---
type: synthesis
question: "<the user's question, verbatim>"
answered_at: YYYY-MM-DD
superseded_by: null
sources:
  - qmd://<vault-name>/<path>      # for each vault page cited
  - https://...                    # for each web source cited
date_updated: YYYY-MM-DD
---
```
```

with:

```
```yaml
---
type: synthesis
question: "<the user's question, verbatim>"
answered_at: YYYY-MM-DD
superseded_by: null
sources:
  - qmd://<vault-name>/<path>      # for each vault page cited
  - https://...                    # for each web source cited
claims:
  c-conclusion:
    sources: [ ... ]               # the evidence behind the bottom-line answer
    by: wiki-research
    asserted_at: YYYY-MM-DD
    confidence: high               # low | medium | high
    superseded_by: null
date_updated: YYYY-MM-DD
---
```

Express the bottom-line answer as one or more anchored conclusion-claims, e.g.
`- <the answer> ^c-conclusion`, each with a matching `claims:` entry as above.
```

- [ ] **Step 5: Verify**

Run: `grep -n "wiki-consult core\|shared wiki-consult read core" RUNBOOK.md`
Expected: appears in the playbook Phase 2 region (~790).

Run: `grep -n "only valid for synthesis pages" RUNBOOK.md`
Expected: **no matches** (the old Phase 6 bullet is gone).

Run: `grep -n "c-conclusion\|by: wiki-research" RUNBOOK.md`
Expected: both appear in the Phase 7c region (~990).

- [ ] **Step 6: Commit**

```bash
git add RUNBOOK.md
git commit -m "spec: wiki-research playbook delegates retrieval and emits claims in Phase 7"
```

---

## Task 8: Update the vault `CLAUDE.md` prose (Phase 5)

Updates the runtime guidance the produced vault carries: Layout, Required frontmatter, a new Claims section, the rewritten Synthesis section, and the Skills list (spec §9, Phase 5).

**Files:**
- Modify: `RUNBOOK.md:1441` (Layout skills line)
- Modify: `RUNBOOK.md:1473-1480` (Required frontmatter)
- Modify: `RUNBOOK.md:1610-1615` (Synthesis pages section)
- Modify: `RUNBOOK.md:1654-1658` (Skills section)

- [ ] **Step 1: Add wiki-consult to the Layout skills line**

Replace line 1441:

```
- `.claude/skills/`         — first-party skills (wiki-research, recall, update-vendors)
```

with:

```
- `.claude/skills/`         — first-party skills (wiki-research, wiki-consult, recall, update-vendors)
```

- [ ] **Step 2: Update the Required frontmatter block**

Replace this block (lines 1473–1480):

```
## Required frontmatter

Every page MUST have:
  type:         (entity | concept | synthesis)
  date_updated: (YYYY-MM-DD)

Entity pages also require:    kind:
Synthesis pages also require: question:, answered_at:, superseded_by:, sources:
```

with:

````
## Required frontmatter

Every page MUST have:
  type:         (entity | concept | synthesis)
  date_updated: (YYYY-MM-DD)

Entity pages also require:    kind:
Synthesis pages also require: question:, answered_at:, superseded_by:, sources:

All page types carry a `claims:` map (empty `claims: {}` until the page has a
provenanced fact). See the Claims section below. A page lacking the key is
treated as `claims: {}` (grandfathered) — never a hard error.
````

- [ ] **Step 3: Insert the Claims section**

Immediately **after** the Required frontmatter block (before `## Wikilinks`, line 1482), insert:

````
## Claims

Provenanced facts are **claims**: an anchored body bullet under `## Facts` (or a
synthesis's answer) joined to a `claims:` frontmatter entry by its anchor.

```yaml
claims:
  c-db-engine:
    sources: [ https://internal-wiki/adr-014, qmd://<vault-name>/raw/postmortem.md ]
    by: wiki-research          # human | wiki-research | deep-research | import
    asserted_at: 2026-06-03
    confidence: high           # low | medium | high
    superseded_by: null        # null | <anchor-id> | qmd://<vault-name>/<path>#<anchor-id>
```

```markdown
## Facts
- Uses DynamoDB for the primary store ^c-db-engine
```

- The body bullet is the single source of truth for wording; frontmatter holds
  metadata only.
- Anchor in the body carries the caret (`^c-db-engine`); the `claims:` key and
  same-page `superseded_by:` use the bare id (`c-db-engine`); cross-page is
  `qmd://<vault-name>/<path>#c-<slug>`.
- Never reverse a fact in place — **supersede** it (see Synthesis/Supersession
  below): write the new claim, point the old claim's `superseded_by:` at it,
  move the old bullet to `## Superseded`. Uniform across entity/concept/synthesis.
- An un-anchored bullet is a valid un-provenanced note (lower trust), not an
  error.
````

- [ ] **Step 4: Rewrite the Synthesis pages section**

Replace this block (lines 1610–1615):

```
## Synthesis pages

Never edit an existing synthesis page to change the answer. Instead:
- Write a new synthesis page with the updated answer at a new slug.
- Set the old page's `superseded_by:` to the new page's `qmd://<vault-name>/<path>`.
- Mention the supersession in the new page's body.
```

with:

```
## Synthesis pages and supersession

Synthesis pages are freely editable (no write-once). Express the answer as one
or more anchored conclusion-claims. To change what any page asserts — entity,
concept, or synthesis — supersede the claim rather than reversing it in place:

- Write the new claim (anchored bullet + `claims:` entry) in the live section.
- Set the old claim's `superseded_by:` to the successor's anchor id (same-page)
  or `qmd://<vault-name>/<path>#c-<slug>` (cross-page).
- Move the old bullet to `## Superseded` with a struck-through line, a one-line
  reason, and the pointer.

A whole synthesis may also be wholesale-replaced by setting its page-level
`superseded_by:` to a newer synthesis's `qmd://<vault-name>/<path>`. Superseded
claims stay in-page and qmd-queryable; live sections show only current truth.
```

- [ ] **Step 5: Update the Skills section**

Replace this block (lines 1654–1658):

```
Three committed first-party skills under `.claude/skills/`:

- **wiki-research** — disciplined research loop: search vault first, optionally invoke deep-research with vault context as a seed, cross-check, write a synthesis page. Default deep-research mode is **UltraDeep**; override only if user explicitly says "quick research" or "standard research". Use it for any non-trivial research request.
- **recall** — direct qmd query wrapper for raw debugging. Not used by wiki-research itself; for the human operator.
- **update-vendors** — vendor autoupdate workflow. Run periodically (or on the weekly nudge from the SessionStart hook).
```

with:

```
Four committed first-party skills under `.claude/skills/`:

- **wiki-research** — disciplined research loop: search vault first (delegating retrieval to wiki-consult's read core), optionally invoke deep-research with vault context as a seed, cross-check, write claim-backed pages. Default deep-research mode is **UltraDeep**; override only if user explicitly says "quick research" or "standard research". Use it for any non-trivial research request.
- **wiki-consult** — read-only "what does the vault already know about X?" Trust-ranked, provenance-annotated, freshness- and contradiction-aware. Never writes, supersedes, or hits the web; suggests `/wiki-research` on a gap. The shared retrieval core wiki-research delegates to.
- **recall** — direct qmd query wrapper for raw debugging. Not used by wiki-research itself; for the human operator.
- **update-vendors** — vendor autoupdate workflow. Run periodically (or on the weekly nudge from the SessionStart hook).
```

- [ ] **Step 6: Verify**

Run: `grep -n "^## Claims\|Four committed first-party\|Synthesis pages and supersession" RUNBOOK.md`
Expected: all three present in the CLAUDE.md template region (1440–1660).

Run: `grep -n "wiki-research, wiki-consult, recall, update-vendors" RUNBOOK.md`
Expected: the Layout line (~1441).

- [ ] **Step 7: Commit**

```bash
git add RUNBOOK.md
git commit -m "spec: vault CLAUDE.md gains Claims section, wiki-consult, claim-level supersession"
```

---

## Task 9: Add claim lint rules to the vault `CLAUDE.md`

Adds the claim-integrity lint pass and updates severity grouping (spec §6 lint additions).

**Files:**
- Modify: `RUNBOOK.md:1797` (insert a new pass after pass 5)
- Modify: `RUNBOOK.md:1799-1802` (severity grouping)

- [ ] **Step 1: Add the claim-integrity lint pass**

After pass 5 (line 1797):

```
5. **Supersession integrity.** Every `superseded_by:` resolves to a real `type: synthesis` file; chains don't loop.
```

insert:

````
6. **Claim integrity.** For every page that carries claims:
   - **Bidirectional join.** Every `claims:` key has a matching `^<key>` anchor in the body, and every `^c-…` anchor in the body has a `claims:` entry. (Un-anchored bullets are exempt — they are un-provenanced notes. A page lacking the `claims:` key is treated as `claims: {}`.)
   - **Required fields present.** Every claim entry has `sources` (non-empty), `by`, `asserted_at`, `confidence`, and `superseded_by`.
   - **`superseded_by` resolves.** When not `null`, it points to a real anchor — a `claims:` key on the same page, or a valid `qmd://<vault-name>/<path>#c-<slug>` cross-page target.
   - **Controlled vocabularies.** `by` ∈ {human, wiki-research, deep-research, import}; `confidence` ∈ {low, medium, high}.
````

- [ ] **Step 2: Update the severity grouping**

Replace this block (lines 1799–1802):

```
Group findings by severity:
- **Hard** (frontmatter missing required fields, supersession cycle, `obsidian unresolved` reports a broken non-`raw/` link) → block commit unless user overrides.
- **Soft** (orphans, stale syntheses) → surface; user decides.
- **Informational** (entity/concept staleness ages) → list for awareness.
```

with:

```
Group findings by severity:
- **Hard** (frontmatter missing required fields, supersession cycle, broken anchor/claim join, claim entry missing a required field, `superseded_by` that doesn't resolve, out-of-vocabulary `by`/`confidence`, `obsidian unresolved` reports a broken non-`raw/` link) → block commit unless user overrides.
- **Soft** (orphans, stale syntheses, claims older than 180 days by `asserted_at`) → surface; user decides.
- **Informational** (entity/concept staleness ages) → list for awareness.
```

- [ ] **Step 3: Verify**

Run: `grep -n "Claim integrity\|Bidirectional join\|broken anchor/claim join" RUNBOOK.md`
Expected: the new pass 6 and the updated Hard-severity line both appear in the lint region (~1797).

- [ ] **Step 4: Commit**

```bash
git add RUNBOOK.md
git commit -m "spec: add claim-integrity lint pass to vault CLAUDE.md"
```

---

## Task 10: Consistency sweep — commit messages, hand-off, intro, commit-graph

Brings every place that enumerates skills or the produced vault's surface into line with the four-skill reality (spec §9 implies; required for the self-sufficiency invariant).

**Files:**
- Modify: `RUNBOOK.md:1181` (Phase 3 commit message)
- Modify: `RUNBOOK.md:2156` (Task 7.1 commit-graph list)
- Modify: `RUNBOOK.md:2202` (Task 7.3 hand-off summary)
- Modify: `RUNBOOK.md:3` (top-of-file blurb)

- [ ] **Step 1: Update the Phase 3 commit message**

Replace line 1181:

```
git commit -m "scaffold: add wiki-research, recall, and update-vendors skills"
```

with:

```
git commit -m "scaffold: add wiki-research, wiki-consult, recall, and update-vendors skills"
```

- [ ] **Step 2: Update the Task 7.1 commit-graph expectation**

Replace line 2156:

```
4. `scaffold: add wiki-research, recall, and update-vendors skills`
```

with:

```
4. `scaffold: add wiki-research, wiki-consult, recall, and update-vendors skills`
```

- [ ] **Step 3: Update the Task 7.3 hand-off summary**

In the hand-off block (line 2202), replace:

```
The wiki-research, recall, and update-vendors skills are committed and discoverable.
```

with:

```
The wiki-research, wiki-consult, recall, and update-vendors skills are committed and discoverable.
```

- [ ] **Step 4: Update the top-of-file blurb**

In the blurb (line 3), replace:

```
committed `wiki-research` + `recall` slash-command skills.
```

with:

```
committed `wiki-research`, `wiki-consult`, and `recall` slash-command skills.
```

- [ ] **Step 5: Verify the whole-file consistency**

Run: `grep -n "write-once\|Write-once\|only valid for synthesis pages\|freely editable in place" RUNBOOK.md`
Expected: **no matches** (every stale write-once / page-level-supersession phrasing is gone).

Run: `grep -n "wiki-research, recall, and update-vendors\|recall, update-vendors\|Three committed" RUNBOOK.md`
Expected: **no matches** (every skill enumeration now includes wiki-consult / says "Four").

Run: `grep -cn "wiki-consult" RUNBOOK.md`
Expected: ≥ 8 (II.8 note, Phase 3 table, the new Task 3.5 skill body, wiki-research SKILL delegation, playbook Phase 2, CLAUDE.md Layout + Skills, hand-off, blurb).

- [ ] **Step 6: Commit**

```bash
git add RUNBOOK.md
git commit -m "spec: sweep skill enumerations + commit-graph for wiki-consult"
```

---

## Task 11: Final self-review against the spec

A read-only pass confirming the runbook still satisfies its editing invariants (meta-repo `CLAUDE.md` rules 1–4) and covers every spec section. No new file changes unless this pass finds a gap.

**Files:** none unless a gap is found (then an extra fix + commit).

- [ ] **Step 1: Re-read the spec's §9 "RUNBOOK.md sections touched" and check each off**

Open `docs/superpowers/specs/2026-06-03-claim-level-knowledge-and-wiki-consult-design.md` §9 and confirm each bullet maps to a task: II.1–II.2 (Tasks 1–2), II.6 (Task 3), II.8 (Task 5), Phase 2 templates (Task 4), Phase 3 wiki-consult + wiki-research edits (Tasks 5–7), Phase 5 lint (Task 9). Note that I.3 Invariant 3 (not listed in §9 but contradicting it) was correctly updated in Task 1.

- [ ] **Step 2: Confirm self-sufficiency invariant (meta-repo CLAUDE.md rule 1)**

Run: `grep -n "wiki-consult/SKILL.md\|claims: {}\|## Superseded" RUNBOOK.md | head`
Confirm the wiki-consult SKILL.md, the claim templates, and the Superseded sections are **inlined** in the runbook (rule 4: templates inline). A fresh session given only `RUNBOOK.md` can produce a claim-ready vault with all four skills.

- [ ] **Step 3: Confirm the runbook still reads top-to-bottom without dangling references**

Run: `grep -n "### Task 3\." RUNBOOK.md`
Expected: 3.1, 3.2, 3.3, 3.4, 3.5 (wiki-consult), 3.6 (Commit) — a clean ascending sequence with no duplicate 3.5.

Run: `grep -n "Task 3.3" RUNBOOK.md`
Expected: the self-review references at lines ~2275–2276 still correctly point at the wiki-research playbook (Task 3.3 was **not** renumbered).

- [ ] **Step 4: Render-sanity the new fenced blocks**

Read the Task 3.5 region and the II.2 Claims subsection. Confirm nested code fences use a longer outer fence than inner (the runbook already uses ```` ```` ```` and ````` ````` ````` for nesting). The wiki-consult SKILL.md is wrapped in a 4-backtick fence containing a 4-backtick markdown block — confirm the insert in Task 5 used a **5-backtick** outer wrapper for the task and **4-backtick** for the file body to avoid premature fence-closing.

Run: `Read RUNBOOK.md` around the inserted Task 3.5 (offset ~1175, limit ~180) and eyeball fence balance.

- [ ] **Step 5: If a gap or broken fence is found, fix it and commit**

```bash
git add RUNBOOK.md
git commit -m "spec: fix <describe gap> found in self-review"
```

If no gap: state "self-review clean, no further changes" and stop.

---

## Self-Review (plan author's checklist — already run)

**1. Spec coverage.** Every spec section maps to a task: §3 claim model → Tasks 1, 2, 4; §4 supersession → Task 3; §5 wiki-consult → Task 5; §6 wiki-research refactor + lint → Tasks 6, 7, 9; §7 migration → folded into Tasks 2, 3, 9 (graceful-degradation/grandfather clauses); §9 sections-touched → Tasks 1–9; the I.3 invariant contradiction (not in §9) → Task 1. Non-goals (auto-trigger, cross-repo, bulk migration) are correctly *not* implemented.

**2. Placeholder scan.** No `TBD`/`TODO`/"add appropriate"/"similar to Task N" — every edit shows the literal old and new text.

**3. Type/name consistency.** Field names are identical everywhere: `sources`, `by`, `asserted_at`, `confidence`, `superseded_by`; anchor form `^c-<slug>` in body, `c-<slug>` as key; vocab `{human, wiki-research, deep-research, import}` and `{low, medium, high}`; cross-page pointer `qmd://<vault-name>/<path>#c-<slug>` — used consistently in II.2, II.6, templates, wiki-consult, playbook, CLAUDE.md, and lint.
