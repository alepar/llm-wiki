# Claim-Level Knowledge + `/wiki-consult` Design Spec

**Date:** 2026-06-03
**Status:** Approved (in brainstorming session). Not yet executed.
**Next step:** Hand off to writing-plans skill for an implementation plan, executed in a separate llm-wiki session.

---

## 1. Summary

This spec makes a produced vault a **trust-first store of provenanced facts**, in service of a larger goal: an attachable knowledge wiki that an agent can consult while working on a *different* project. Two changes work together:

1. **Claim-level knowledge.** Today, facts on entity/concept pages are free-text body lines with no provenance, no author, and no history — they are "freely editable in place" (`RUNBOOK.md:177`). This spec turns each fact into a **claim**: a block-anchored body bullet (`^c-xxx`) backed by a frontmatter `claims:` map carrying its sources, author, assertion date, confidence, and supersession pointer. This is applied uniformly across all three page types, which also lets **synthesis pages become living pages** (the write-once rule is dropped, because claim-level supersession now provides the integrity guarantee that write-once was a blunt workaround for).

2. **A new `/wiki-consult` skill.** A fast, **read-only** consult path — "what does the vault already know about X?" — that returns a trust-ranked, provenance-annotated, freshness-aware answer. It never writes, never supersedes, and never hits the web. It is distinct from `wiki-research` (expensive; writes) and `recall` (raw debug query). `wiki-research`'s retrieval phase is refactored to delegate to wiki-consult's shared retrieval/trust-ranking logic, so the vault has a single retrieval path.

All changes are edits to `RUNBOOK.md` — the runbook is the artifact; newly-instantiated vaults inherit the changes. Existing vaults degrade gracefully (see §7).

This is the MVP toward **attachable knowledge llm-wikis**. It deliberately defers the auto-trigger reflex (the agent consulting the vault *without being asked*) and cross-repo attachment (consulting vault Z from project A's session). Those are tracked as follow-ups (§8).

## 2. Goals and non-goals

**Goals:**
- Every fact written to a produced vault can carry per-fact provenance (`sources`), author (`by`), assertion date (`asserted_at`), and `confidence`.
- A fact can be **superseded** without destroying it or forking its page: claim-level, in-page, qmd-queryable history.
- Synthesis pages are freely editable for corrections/additions; material answer changes are handled by superseding the conclusion-claim, not forking the page.
- A produced vault has a manual `/wiki-consult <question>` command that returns a concise, provenance-rich answer, flags stale claims and contradictions, and reports gaps.
- `wiki-research` and `wiki-consult` share one retrieval/trust-ranking implementation.
- The vault stays **vault-pure**: markdown + Obsidian-native constructs (frontmatter, `^block-anchors`, wikilinks). No new runtime, no database, no sidecar formats.

**Non-goals:**
- **Auto-trigger / consult-reflex** (the deferred punch-list item #4). `/wiki-consult` is manual-only in this MVP.
- **Cross-repo attachment** (item #5). wiki-consult is invoked from within the vault session; making a vault consultable from a foreign project's session is a follow-up.
- **Bulk migration of existing vaults.** Forward-only; old free-text bullets remain valid (§7).
- **A structured/multi-field author identity object.** `by` is a small controlled-vocabulary string; richer authorship is a multiplayer concern, deferred.
- **Changing the trust hierarchy** (synthesis > entity/concept > raw > web). Unchanged; claims add provenance *within* that hierarchy.
- **Adding a vendor dependency.** Everything renders client-side in Obsidian/GitHub.

## 3. The claim data model

Same mechanism on entity, concept, and synthesis pages.

**Frontmatter** — a `claims:` map keyed by the block anchor used in the body:

```yaml
claims:
  c-db-engine:
    sources:                       # provenance, per fact (punch-list #1)
      - https://internal-wiki/adr-014
      - qmd://<vault>/raw/migration-postmortem.md
    by: wiki-research              # write-identity (punch-list #2)
    asserted_at: 2026-06-03
    confidence: high               # low | medium | high
    superseded_by: null            # null | <anchor> | qmd://<vault>/<path>#<anchor>
```

**Body** — human-readable bullets carrying Obsidian block anchors:

```markdown
## Facts
- Uses DynamoDB for the primary store ^c-db-engine

## Superseded
- ~~Uses Postgres for the primary store~~ ^c-db-engine-old
  (superseded by ^c-db-engine, 2026-06-03 — migrated off RDS, see ADR-014)
```

**Design decisions (all locked in brainstorming):**

- **Uniform across all three page types.** Synthesis keeps its other special fields (`question`, `answered_at`, the 180-day staleness rule) and *only* loses write-once. A synthesis's answer is expressed as one or more conclusion-claims.
- **No text-mirror.** The body bullet is the single source of truth for a claim's wording; frontmatter holds metadata only, joined to the body by anchor. Keeps it DRY; lint enforces the join (§6).
- **`by` is a controlled vocabulary:** `human | wiki-research | deep-research | import`. Extensible later. When `wiki-research` writes a fact grounded in a deep-research run, the writer is `wiki-research` and the deep-research evidence appears in `sources`. Direct human edits are `human`.
- **Graceful degradation.** A body bullet with no `^anchor` (and thus no `claims:` entry) is a valid **un-provenanced note** — visibly lower-trust in consult output — not a lint error.

## 4. Claim-level supersession

Replaces the page-level write-once/fork mechanic (current `RUNBOOK.md` II.6) with a claim-level one:

1. Write the new claim as a normal bullet + anchor + `claims:` entry in the live section.
2. Set the old claim's `superseded_by` to the new anchor (`<anchor>` for same-page; `qmd://<vault>/<path>#<anchor>` for cross-page, e.g. synthesis→synthesis).
3. Move the old bullet to a `## Superseded` section: strikethrough + a one-line reason and pointer.

Live sections (`## Facts`, synthesis body) show only current truth. Superseded claims stay in-page and qmd-queryable, so "what did we believe, and when, and why did it change" is preserved without forking pages or relying on git archaeology. This gives synthesis its drift history while letting the page be freely edited.

## 5. The `/wiki-consult` skill

**Purpose:** mid-task, read-only "what does the vault already know about X?"

**Properties:**
- **Read-only.** Never writes, never supersedes, never ingests, never hits the web.
- **Manual invocation:** `/wiki-consult <question>`. No auto-trigger (deferred).
- **Hybrid scope:** queries both curated pages and `raw/`. Curated claims are presented as *the answer*; raw-doc hits are presented as *supporting evidence* (so a topic with no claim yet still returns something).
- **Trust-ranked:** synthesis > entity/concept > raw, per the existing hierarchy. Superseded claims are skipped.
- **Freshness-aware:** reuse the vault's 180-day threshold (`RUNBOOK.md:826`) for both synthesis freshness and claim `asserted_at` age. Warn; never hide.
- **Contradiction-aware:** if two live claims conflict, surface both with their provenance and flag. Do **not** resolve (unlike `wiki-research` Phase 6, which stops on contradiction — consult is read-only, so it just flags and continues).
- **Gap-reporting + escalation:** on a miss or stale/contradicted answer, *suggest* `/wiki-research` but never run it.

**Output shape (illustrative):**

```
From the vault:
• Primary store: DynamoDB — confidence: high, asserted 2026-06-03 by wiki-research
  source: ADR-014  [[acme-infra]]
• ⚠ Auth: Auth0 — confidence: medium, asserted 2025-08-01 (10mo old — may be stale)  [[acme-auth]]
• ⚠ Conflict: [[acme-infra]] says us-east-1, [[acme-dr-plan]] says us-west-2 — both live

No vault entry for: rate-limiting strategy.
→ Gaps/stale found. Escalate with /wiki-research?
```

**Relationship to `recall`:** unchanged. `recall` stays a raw qmd query for debugging; `wiki-consult` is the answer-shaped, trust-ranked, provenance-aware read path. No merge.

## 6. wiki-research refactor + lint

**Retrieval delegation.** `wiki-research`'s Phase 2–3 (qmd-first retrieval + trust-tier bucketing, `RUNBOOK.md:790-841`) is refactored to call wiki-consult's shared retrieval/trust-ranking logic, so there is a single retrieval path. `wiki-research` then continues into its web-research + write phases as before. wiki-consult is the read core; wiki-research wraps it with write capability.

**Phase 7 (write) updates.** When `wiki-research` writes or updates entity/concept/synthesis pages, it now emits **claims**: each fact gets an anchor and a `claims:` entry with `sources`, `by: wiki-research`, `asserted_at`, `confidence`, and `superseded_by: null`. Updating an existing fact follows the §4 supersession flow.

**Lint additions (Phase 5 lint section):**
- **Bidirectional anchor/claim join:** every `claims:` key has a matching `^anchor` in the body, and every `^c-` anchor in the body has a `claims:` entry. (Un-anchored bullets are exempt — they are un-provenanced notes, §3.)
- **Required claim fields present:** `sources`, `by`, `asserted_at`, `confidence`, `superseded_by` on every claim entry.
- **`superseded_by` resolves** to a real anchor — in-page, or a valid `qmd://<vault>/<path>#<anchor>` cross-page target.
- **`by` ∈ {human, wiki-research, deep-research, import}.**
- **`confidence` ∈ {low, medium, high}.**

## 7. Migration

**Forward-only with graceful degradation.** The `RUNBOOK.md` change affects newly-instantiated vaults. For an existing vault:
- New writes use claims.
- Old free-text bullets remain valid; lint treats an un-anchored bullet as an un-provenanced note, not an error.
- `wiki-consult` presents un-provenanced notes at lower trust (no confidence/source/date to show).
- No bulk retrofit is required or performed. Pages can be upgraded to claims opportunistically when next edited.

## 8. Deferred follow-ups (toward attachable knowledge wikis)

- **#4 — Consult-reflex / auto-trigger.** The agent consults the vault automatically as a side-input to whatever task it's doing, without an explicit `/wiki-consult` call. The hard, highest-value piece; deferred so the read core can be proven first.
- **#5 — Cross-repo attachment.** Make a vault's qmd-over-MCP retrieval consumable from a *foreign* project's session, so vault Z attaches to working-project A. Mild tension with the "no MCP servers of our own" philosophy to be resolved then.
- **wiki-research/wiki-consult full DRY consolidation** beyond the retrieval phase, if the refactor surfaces more shared surface.

## 9. `RUNBOOK.md` sections touched

- **II.1–II.2** — page-type table + frontmatter contracts: add `claims:` to all three types.
- **II.6** — rewrite from page-level synthesis supersession to claim-level supersession across all types; document the `## Superseded` convention; remove the synthesis write-once rule.
- **II.8** — note wiki-consult's hybrid scope usage.
- **Phase 2 templates** (`entity.md`, `concept.md`, `synthesis.md`) — add the `claims:` frontmatter stub + `## Facts` / `## Superseded` body sections.
- **Phase 3** — add the new `wiki-consult` skill (`SKILL.md` + any playbook); edit the `wiki-research` SKILL/playbook to delegate retrieval and to emit claims in Phase 7.
- **Phase 5 lint** — add the claim lint rules (§6).

Out of scope to touch: the meta-repo's own `CLAUDE.md`/specs conventions; the trust hierarchy; vendor/submodule wiring; `recall`.
