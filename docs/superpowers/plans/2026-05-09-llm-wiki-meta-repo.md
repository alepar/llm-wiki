# LLM-Wiki Meta-Repo Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build out the `llm-wiki` meta-repo by writing four files (`docs/prior-art.md`, `CLAUDE.md`, `README.md`, `RUNBOOK.md`) per the design spec at `docs/superpowers/specs/2026-05-09-llm-wiki-meta-design.md`.

**Architecture:** Faithful integration of the existing 1611-line base runbook at `/Users/alepar/AleCode/family/docs/plans/2026-05-08-llm-wiki-vault-instantiation.md` with new sections (three retrieval scopes, raw/ folder, kepano obsidian-skills, Obsidian CLI lint, auto-commit, SessionStart hook, install-helper recipe, vendor-autoupdate). Single-file `RUNBOOK.md` preserves the "give-this-to-a-fresh-Claude-session" property.

**Tech Stack:** Markdown, git, no code (documentation project).

**Working directory:** `/Users/alepar/AleCode/llm-wiki/`

**Reference files (read-only inputs):**
- Spec: `/Users/alepar/AleCode/llm-wiki/docs/superpowers/specs/2026-05-09-llm-wiki-meta-design.md` (already committed)
- Base runbook: `/Users/alepar/AleCode/family/docs/plans/2026-05-08-llm-wiki-vault-instantiation.md`

**Commit message convention:** `<scope>: <subject>` (lowercase). Add `Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>` trailer to each commit (matches the existing spec commit's pattern).

---

## Task 1: Bootstrap repo with `.gitignore`

**Files:**
- Create: `/Users/alepar/AleCode/llm-wiki/.gitignore`

This task lands a minimal `.gitignore` covering OS/editor noise so the working tree stays clean while subsequent tasks land content.

- [ ] **Step 1: Create `.gitignore`**

Write `/Users/alepar/AleCode/llm-wiki/.gitignore` with this exact content:

```
# OS / editor state
.DS_Store
*.swp
.idea/
.vscode/
```

- [ ] **Step 2: Verify file content**

Run: `cat /Users/alepar/AleCode/llm-wiki/.gitignore`
Expected: the four lines above (plus the comment header).

- [ ] **Step 3: Commit**

```bash
cd /Users/alepar/AleCode/llm-wiki
git add .gitignore
git commit -m "$(cat <<'EOF'
chore: add .gitignore

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
git log --oneline
```

Expected: 2 commits in log — the spec commit and this one.

---

## Task 2: Write `docs/prior-art.md`

**Files:**
- Create: `/Users/alepar/AleCode/llm-wiki/docs/prior-art.md`

Captures the prior-art research (Karpathy, kennyg, NiharShrotri, ekadetov, kepano, Obsidian CLI landscape) with adopted/rejected decisions and rationale.

- [ ] **Step 1: Write `docs/prior-art.md`**

Write to `/Users/alepar/AleCode/llm-wiki/docs/prior-art.md` with this exact content:

````markdown
# Prior Art Notes

Research conducted 2026-05-09 ahead of writing this repo's runbook. Captures what was adopted from each source and what was deliberately left out, with rationale.

## 1. Karpathy — `llm-wiki` gist

**URL:** https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f

**Summary.** The canonical sketch. Three folders: `raw/` (immutable sources), `wiki/` (LLM-maintained markdown with `index.md` + append-only `log.md`), `schema/` (CLAUDE.md/AGENTS.md). Three workflows: ingest, query, lint. Frontmatter optional, page types not formally prescribed, link syntax not specified. Recommends qmd at scale and Obsidian as a viewer. The gist is explicit that "everything is optional — pick what's useful."

**Adopted:**
- Three-folder split (`raw/`, `wiki/`, `schema/`) — the load-bearing idea. Our vault uses topic-first folders for the "wiki" content (no top-level `wiki/`), but `raw/` lands as a vault-root folder for ingested source documents.
- Lint as a workflow, not just tooling.

**Rejected:**
- Underspecification of frontmatter and page types — the gist is a manifesto; a runbook needs commitments.
- `index.md` as the navigation primitive — doesn't scale past ~50 pages; qmd-from-day-one supersedes.
- Append-only `log.md` — git commit history (with auto-commit-after-approved-write) gives the same audit trail for free.

## 2. kennyg — `llm-wiki` gist (Obsidian-flavored)

**URL:** https://gist.github.com/kennyg/6c45cace2e1c4e424a28fcd51dd6c25b

**Summary.** Same skeleton as Karpathy's, but commits to specifics: explicit `type:` field with four values (source-summary, entity, concept, synthesis), `date_updated`, `source_count`, `confidence: high/medium/low` on concepts, `Wiki/{sources,entities,concepts,synthesis}/` layout, `[key::value]` Dataview inline metadata, official Obsidian CLI as a dependency.

**Adopted:**
- `confidence:` on concept pages (was already in our base spec).
- The four-type taxonomy informs our three (entity/concept/synthesis); `source-summary` is collapsed into `raw/` folder content rather than a separate page type.

**Rejected:**
- `source_count:` on entity/concept pages — adds maintenance burden without clear value.
- `[key::value]` Dataview inline metadata — Obsidian-plugin-specific; invisible to qmd.
- Type-first folder layout — we use topic-first per Invariant 5.

## 3. NiharShrotri/llm-wiki

**URL:** https://github.com/NiharShrotri/llm-wiki

**Summary.** A real Python package with a Typer CLI (`wiki init/add/ingest/query/lint/serve/status/reindex`) and a FastAPI web UI with seven pages. Folders match kennyg's. Frontmatter carries `sources:` for provenance. Retrieval is qmd hybrid (BM25 + EmbeddingGemma-300M + Qwen3-Reranker-0.6B) with three scopes: `wiki`, `raw`, `hybrid`. Three-pass ingest pipeline (extraction → page draft → source summary). Local-only Ollama + Qwen3-14B stack.

**Adopted:**
- Three retrieval scopes (`wiki`, `raw`, `hybrid`) — single best idea found. Cite-against-raw, synthesize-against-wiki, fall-back-to-hybrid maps cleanly onto our wiki-research / recall split.
- `sources:` frontmatter array as the canonical provenance handle (already in our base spec for syntheses).

**Rejected:**
- FastAPI web UI — product surface area not appropriate for a runbook.
- Hard-coded Ollama + specific model — implementation detail, not pattern.
- Building a Python package — creates a second source of truth (the code) competing with the schema doc. Markdown + skills + qmd is leaner.

## 4. ekadetov/llm-wiki

**URL:** https://github.com/ekadetov/llm-wiki

**Summary.** Packages the pattern as a Claude Code plugin: `.claude-plugin/`, `commands/`, `hooks/`, `skills/wiki/`, `WALKTHROUGH.md`. SessionStart hook auto-installs qmd and marp-cli. Auto-commits wiki changes. Expects an Obsidian vault at a fixed path. 61% Python / 39% Shell.

**Adopted:**
- Skill-based packaging (closer to our design than NiharShrotri's Python package).
- SessionStart hook for dependency bootstrap (we use it for qmd freshness check + index update — not auto-install).
- Auto-commit on user-approved wiki mutation — gives free git audit trail; aligns with our "git history replaces log.md" choice.

**Rejected:**
- Hard-coded vault path.
- Bundling marp-cli (slide decks) by default — scope creep.
- Auto-install of qmd — install is mutating; we keep the user in control via the install-helper recipe.

## 5. kepano/obsidian-skills (Steph Ango / Obsidian)

**URL:** https://github.com/kepano/obsidian-skills

**Summary.** Five skills, no CLI of its own. Skills: `obsidian-markdown` (write OFM with wikilinks/embeds/callouts/properties), `obsidian-bases` (Bases files: views, filters, formulas), `json-canvas` (Canvas files), `obsidian-cli` (use the official CLI from an agent), `defuddle` (clean-markdown extraction from web pages — token-saving reader-mode). Follows the Agent Skills spec.

**Adopted:**
- Pinned as a git submodule at `.claude/skills/obsidian-skills/` in produced vaults.
- `defuddle` — canonical web-page-to-markdown converter for the `raw/` ingest path.
- `obsidian-markdown` — sub-skill encoding wikilink/callout/property syntax for authoring.

**Not advertised in produced vaults' CLAUDE.md:**
- `obsidian-bases`, `json-canvas` — Obsidian-specific surfaces; not part of the LLM-wiki pattern. Accessible via the submodule but not surfaced.
- `obsidian-cli` (the kepano sub-skill) — accessible; we use the official CLI directly for `unresolved`.

## 6. Obsidian CLI landscape (May 2026)

**Surveyed:**
- **Official Obsidian CLI** — shipped Feb 2026 in v1.12.4. Subcommands: `unresolved`, `search`, `read`, `create`, `daily`, `tasks`, `tags counts`, `diff`. No frontmatter validator, no full vault-lint command.
- **Linter plugin** (platers/obsidian-linter) — most mature in-vault normalization tool. Plugin, not CLI.
- **davidpp/obsidian-cli** — third-party, AI-optimized; explicit get/set on YAML frontmatter, integrates Omnisearch.
- **jwhonce/obsidian-cli** — older third-party CLI; basic note management; less active.
- **bitbonsai/mcpvault** — MCP server for Obsidian; validates YAML frontmatter and blocks dangerous JS objects.

**Adopted:**
- Official `obsidian unresolved` for broken-wikilink check in the lint pass — blessed, free, handles Obsidian's link resolution rules correctly.
- Official `obsidian search` as fallback retrieval when qmd is unavailable.

**Rejected:**
- Linter plugin — not CI-friendly, runs interactively in Obsidian.
- davidpp/obsidian-cli — flagged as candidate if we outgrow the official CLI; not adopted now.
- jwhonce/obsidian-cli — less active than davidpp's.
- mcpvault — different shape (MCP); our retrieval is already qmd.

**Sharp take.** No battle-tested CLI vault-linter exists as of May 2026. The official CLI is too new to lean on heavily; third-party CLIs are one-maintainer projects. Vault hygiene stays LLM-driven (orphans, contradictions, stale claims); deterministic-rule lint is limited to what `obsidian unresolved` and our frontmatter pass can check.

## Cross-cutting takeaways

1. NiharShrotri's three retrieval scopes — adopted.
2. kepano's `obsidian-markdown` and `defuddle` skills — adopted as a submodule.
3. ekadetov's auto-commit pattern — adopted (after-user-approved-write only).
4. ekadetov's SessionStart hook — adopted (read-only freshness check + idempotent index update).
5. Official `obsidian unresolved` — adopted for lint.
6. Vendor autoupdate philosophy — first-party design, applied to qmd, obsidian-cli, deep-research, obsidian-skills.
7. Of five "llm-wiki" sources, exactly one (NiharShrotri) is implemented end-to-end and one (ekadetov) is packaged as a reusable artifact. The two gists are sketches.
````

- [ ] **Step 2: Validate**

```bash
test -f /Users/alepar/AleCode/llm-wiki/docs/prior-art.md && echo "exists"
wc -l /Users/alepar/AleCode/llm-wiki/docs/prior-art.md
grep -c '^## ' /Users/alepar/AleCode/llm-wiki/docs/prior-art.md
```

Expected: `exists`; ~115-130 lines; **at least 7** level-2 sections (1-6 plus cross-cutting).

- [ ] **Step 3: Commit**

```bash
cd /Users/alepar/AleCode/llm-wiki
git add docs/prior-art.md
git commit -m "$(cat <<'EOF'
docs: add prior-art notes (research provenance for runbook decisions)

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Write `CLAUDE.md` and `README.md`

**Files:**
- Create: `/Users/alepar/AleCode/llm-wiki/CLAUDE.md`
- Create: `/Users/alepar/AleCode/llm-wiki/README.md`

Per the spec §13 build sequence, these two land in one commit.

- [ ] **Step 1: Write `CLAUDE.md`**

Write to `/Users/alepar/AleCode/llm-wiki/CLAUDE.md` with this exact content:

````markdown
# llm-wiki — meta-repo

This repository holds the canonical runbook for instantiating LLM-friendly markdown wikis. The runbook lives at [`RUNBOOK.md`](./RUNBOOK.md) and is designed to be handed verbatim to a fresh Claude Code session along with a chosen domain — that session scaffolds a working vault.

This file is the agent-facing description of what this repo IS and how to edit it without breaking it. The human-facing handoff is in [`README.md`](./README.md).

## Identity

This is a tooling repository, not a wiki. It contains:

- `RUNBOOK.md` — the consolidated, self-sufficient instantiation runbook. Single file by design; produced vaults are self-contained.
- `CLAUDE.md` (this file) — agent-facing context.
- `README.md` — human-facing handoff.
- `docs/prior-art.md` — research notes on what was adopted from each source.
- `docs/superpowers/specs/` — design specs for this meta-repo's evolution.
- `docs/superpowers/plans/` — implementation plans.

## Philosophy

A produced vault is **markdown + skills + qmd**. No Python packages, no FastAPI UIs, no MCP servers of our own — only what's needed for an agent to do disciplined research and writing against a typed-page wiki.

Operating principles:

- **Vault-pure** — produced vaults are git repos of markdown files, plus a few committed dotfiles for skills and templates. They are portable to any other tool (Obsidian, plain text editor, different search engine) without modification.
- **Agent-curated** — every write requires explicit user approval (diff summary shown first), but commits are automatic after approval (Invariant 9).
- **Topic-first** — page paths are `<topic>/<subtopic>/<slug>.md`; page type lives in frontmatter.
- **Submodule-vendored** — third-party skills (deep-research, obsidian-skills) are pinned as submodules. The `update-vendors` skill in produced vaults handles bumps.
- **No silent state mutations of vault content** — `qmd ingest`/`qmd collection add` are user-run; `qmd update`/`qmd embed` are agent-allowed (idempotent index ops).
- **Single-file runbook** — `RUNBOOK.md` is self-sufficient. A fresh Claude session given just that file (plus user inputs) produces a working vault.

## Sources / credits

- **Andrej Karpathy** — original "llm-wiki" gist; three-folder split; ingest/query/lint workflow framing.
- **kennyg** — concrete frontmatter conventions; `confidence:` on concepts.
- **NiharShrotri** ([github](https://github.com/NiharShrotri/llm-wiki)) — three retrieval scopes (wiki/raw/hybrid); `sources:` array as canonical provenance.
- **ekadetov** ([github](https://github.com/ekadetov/llm-wiki)) — Claude Code skill packaging; SessionStart hook bootstrap; auto-commit pattern.
- **Steph Ango / kepano** ([github](https://github.com/kepano/obsidian-skills)) — `defuddle` and `obsidian-markdown` skills; agent-skills format for Obsidian.
- **tobilu / qmd** — hybrid retrieval (BM25 + vector + LLM rerank).
- **199-biotechnologies / claude-deep-research-skill** — multi-source web research with citation tracking.
- **Obsidian (official CLI)** — `obsidian unresolved` for blessed wikilink resolution.

See [`docs/prior-art.md`](./docs/prior-art.md) for the full survey of what was adopted from each.

## Rules for editing `RUNBOOK.md`

The runbook is the artifact. Edits must preserve these properties:

1. **Self-sufficiency.** A fresh Claude session given just `RUNBOOK.md` and user inputs (`<vault-name>`, `<vault-root>`, `<identity-statement>`) must be able to execute Phase 0 → Phase 7 end-to-end. Don't add references to external files that the executing session won't have.

2. **Invariants are framework-invariant.** Part I.3 invariants must not be relaxed based on perceived domain need. Use Part II.7 (per-domain extensions) for domain-specific changes.

3. **Phase atomicity.** Each phase ends with at least one git commit. Don't fold phases together; don't add a phase that doesn't have a commit endpoint.

4. **Templates inline.** Every literal template (`.templates/<type>.md`, skill SKILL.md files, vault `CLAUDE.md`) is reproduced inline in the runbook, not referenced by external link.

5. **Vendor autoupdates.** Every external dependency mentioned in the runbook (qmd, Obsidian CLI, submodules) must have an autoupdate path documented — pinned for reproducibility, refreshable via `/update-vendors`.

6. **Adding new vendor components.** When adding a new external dep:
   - Document it in the runbook with a pinned version/SHA.
   - Add it to the `update-vendors` skill's vendor list.
   - Update `docs/prior-art.md` with the rationale.
   - Update this file's "Sources / credits" section.

7. **Removing components.** When removing a vendor or skill:
   - Mark the affected runbook section as deprecated for one revision before deleting.
   - Note the deprecation in `docs/prior-art.md`.

## What this repo is NOT

- Not a wiki itself.
- Not a runtime tool — it produces vaults; it doesn't run them.
- Not a single-domain artifact — vaults instantiated from this runbook are domain-agnostic.
````

- [ ] **Step 2: Write `README.md`**

Write to `/Users/alepar/AleCode/llm-wiki/README.md` with this exact content:

````markdown
# llm-wiki

A blueprint for instantiating LLM-friendly markdown wikis: typed pages, topic-first layout, qmd hybrid retrieval, deep-research integration, Obsidian-compatible.

## What you get

A self-contained git repo of markdown files (your vault), pre-wired with:

- **Three core page types** (entity, concept, synthesis) plus optional domain-specific extensions.
- **Topic-first folders** with placement rules.
- **qmd hybrid retrieval** — BM25 + vector + LLM rerank — with three scopes (wiki / raw / hybrid).
- **Three committed first-party skills**: `wiki-research` (disciplined research loop), `recall` (raw qmd debug), `update-vendors` (vendor autoupdate workflow).
- **Two pinned submodules**: `deep-research` (multi-source web research) and `obsidian-skills` (defuddle for ingest, obsidian-markdown for authoring).
- **Auto-commit after user-approved writes** — git history is the audit trail.
- **SessionStart hook** for qmd index freshness + weekly vendor-update nudge.
- **Obsidian-compatible** — open the vault in Obsidian for free preview; `obsidian unresolved` powers the lint pass.

## How to instantiate a new wiki

You'll need:

- [Claude Code](https://docs.anthropic.com/claude/docs/claude-code) installed and a fresh session.
- About 30 minutes for the initial scaffold + qmd setup.
- A target directory for the new vault (a fresh empty path, e.g., `~/AleCode/<your-vault-name>`).

Steps:

1. **Open Claude Code in any directory.**

2. **Hand it the runbook.** Say:

   > Execute the vault instantiation runbook at `<path-to-this-repo>/RUNBOOK.md`.

3. **Provide the three inputs** when prompted:
   - `<vault-name>` — kebab-case, e.g., `cooking-notes`, `quant-investing`.
   - `<identity-statement>` — one paragraph (3–5 sentences) describing what this vault is for.
   - `<vault-root>` — absolute path to the new vault directory.

4. **Follow the runbook's phases.** The session runs Phase 0 (collect inputs) through Phase 7 (verify). It commits after each phase.

5. **Run the qmd setup commands** the session prints. The agent never auto-runs install or `qmd collection add` — it gives you commands and waits.

6. **Smoke-test** with a domain-relevant question via `/wiki-research <question>`.

The vault is operational at the end of Phase 7. The first real wiki page is your domain content; write it in a separate session.

## Maintaining a vault

Inside a produced vault, run `/update-vendors` periodically to check for upstream updates to qmd, Obsidian CLI, deep-research, and obsidian-skills. The skill summarizes changes and offers ready-to-run upgrade commands; you decide what to apply.

The SessionStart hook nudges you weekly if you haven't run `/update-vendors`.

## What this is NOT

- **Not a coach, journaling system, or project tracker.** No periodic files, no goals, no habits. If you need those, this isn't the right starting point.
- **Not multi-actor.** Single-vault, single-domain.
- **Not auto-mutating of vault content.** Every write requires explicit user approval (the agent commits automatically after you approve).

## Background

This pattern is "llm-wiki" — originated by [Andrej Karpathy](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) and elaborated by [kennyg](https://gist.github.com/kennyg/6c45cace2e1c4e424a28fcd51dd6c25b), [NiharShrotri](https://github.com/NiharShrotri/llm-wiki), and [ekadetov](https://github.com/ekadetov/llm-wiki).

This repo's runbook synthesizes those approaches with kepano's [obsidian-skills](https://github.com/kepano/obsidian-skills), the official [Obsidian CLI](https://obsidian.md/cli), and [tobilu/qmd](https://github.com/tobilu/qmd) for retrieval.

See [`docs/prior-art.md`](./docs/prior-art.md) for what was adopted from each source. See [`CLAUDE.md`](./CLAUDE.md) for design philosophy.
````

- [ ] **Step 3: Validate**

```bash
test -f /Users/alepar/AleCode/llm-wiki/CLAUDE.md && echo "claude-md exists"
test -f /Users/alepar/AleCode/llm-wiki/README.md && echo "readme exists"
grep -c '^## ' /Users/alepar/AleCode/llm-wiki/CLAUDE.md
grep -c '^## ' /Users/alepar/AleCode/llm-wiki/README.md
```

Expected: both files exist; CLAUDE.md has at least 5 sections; README.md has at least 5 sections.

- [ ] **Step 4: Commit**

```bash
cd /Users/alepar/AleCode/llm-wiki
git add CLAUDE.md README.md
git commit -m "$(cat <<'EOF'
docs: add CLAUDE.md (agent-facing) and README.md (human-facing)

CLAUDE.md captures repo identity, philosophy, sources/credits, and rules for
editing RUNBOOK.md without breaking framework invariants. README.md is the
human handoff: how to instantiate a new wiki using RUNBOOK.md.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Initialize `RUNBOOK.md` from base

**Files:**
- Create: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md` (copy of base, then modified)
- Read-only: `/Users/alepar/AleCode/family/docs/plans/2026-05-08-llm-wiki-vault-instantiation.md`

This task copies the base runbook into the meta-repo as a starting point. Subsequent tasks (5–11) modify it in place via Edit operations. No commit yet — RUNBOOK.md commits at the end of Task 11.

- [ ] **Step 1: Copy the base runbook**

```bash
cp /Users/alepar/AleCode/family/docs/plans/2026-05-08-llm-wiki-vault-instantiation.md /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
```

- [ ] **Step 2: Verify the copy**

```bash
wc -l /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
head -3 /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
```

Expected: ~1611 lines; first line is `# LLM-Wiki Vault Instantiation Runbook`.

- [ ] **Step 3: Read the file fully into context**

Use the `Read` tool on `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md` to load the full content. You will edit this file repeatedly across the next several tasks; familiarity with the structure prevents misplaced edits.

(No git operation in this task — modifications follow.)

---

## Task 5: Update Part I (architecture, invariants)

**Files:**
- Modify: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md`

Updates Part I.1 (the shape) to mention the `raw/` folder; revises Invariants 4 and 7; adds Invariant 9.

- [ ] **Step 1: Update the architectural sidecar paragraph (after the ASCII diagram in Part I.1)**

Find this exact text in RUNBOOK.md:

```
A sidecar deep-research submodule pinned at `.claude/skills/deep-research/` is invoked by `wiki-research` when the vault lacks coverage; output lands in `.research/<topic-slug>_<YYYYMMDD>/` (excluded from indexing by being dot-prefixed) and the curated synthesis page lands in the vault normally.
```

Replace with:

```
Two sidecar submodules pin at `.claude/skills/`:
- `deep-research` — invoked by `wiki-research` when the vault lacks coverage. Output lands in `.research/<topic-slug>_<YYYYMMDD>/` (excluded from indexing by being dot-prefixed) and the curated synthesis page lands in the vault normally.
- `obsidian-skills` (kepano) — provides `defuddle` (web→clean markdown for `raw/` ingest) and `obsidian-markdown` (wikilink/callout/property syntax reference for authoring).

Source documents ingested for long-term grounding land in a vault-root `raw/` folder, indexed by qmd alongside the wiki content. The vault therefore has three retrieval scopes: `wiki` (curated typed pages), `raw` (source documents), and `hybrid` (both).
```

- [ ] **Step 2: Revise Invariant 4 in Part I.3**

Find this exact text:

```
4. **qmd mutating operations are never agent-run.** `qmd collection add`, `qmd embed`, `qmd update`, `qmd ingest <url>` — all user-run only. Agent prints the commands and waits.
```

Replace with:

```
4. **Vault-content-changing qmd operations are never agent-run.** `qmd collection add` (initial setup) and `qmd ingest <url>` (pulls external content) are user-run only; agent prints the commands and waits. Index-only operations `qmd update` and `qmd embed` are idempotent and don't change vault content — the agent runs these automatically (typically from the SessionStart hook). qmd install is agent-guided via the install-helper recipe (Phase 6) and user-executed by default; the user may opt the agent in to executing the install per session.
```

- [ ] **Step 3: Revise Invariant 7 in Part I.3**

Find this exact text:

```
7. **External web sources are not copied into the vault.** They are referenced by `https://...` URL. The vault stays portable and human-readable.
```

Replace with:

```
7. **External web sources are not copied into the vault as wiki pages.** When long-term grounding is needed, sources may be ingested into `raw/` as cleaned-up markdown via the defuddle skill. Synthesis pages cite ingested sources by `qmd://<vault-name>/raw/<slug>.md` and non-ingested sources by `https://...`. The vault stays portable and human-readable; `raw/` documents have minimal frontmatter (no `type:` field) so they're never confused with wiki pages.
```

- [ ] **Step 4: Add Invariant 9 to Part I.3**

Find the end of Invariant 8 in Part I.3:

```
8. **Concurrent-sessions discipline.** Treat any uncommitted changes the executor did not author as belonging to another session. Use `git add -p` to stage only own hunks.
```

Append (immediately after, on a new line):

```
9. **Auto-commit follows user-approved writes.** Agent shows a diff summary before any write (file path, +/- line counts, key frontmatter fields touched). User approves explicitly. Agent writes, then commits with a topic-shaped message and reports the SHA. One commit per approved-write turn — no batching across turns. Full unified diff is available on user request but not shown by default. Git history is the audit trail; there is no separate `log.md`.
```

- [ ] **Step 5: Validate Part I changes**

```bash
grep -n 'three retrieval scopes' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n 'Vault-content-changing qmd' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n 'Auto-commit follows user-approved writes' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -c '^[0-9]\. \*\*' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
```

Expected: each grep returns at least one matching line; the numbered-invariant count is **at least 9** (was 8).

(No commit — RUNBOOK.md commits at end of Task 11.)

---

## Task 6: Update Part II (document model — scopes, raw/, citation forms)

**Files:**
- Modify: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md`

Adds the three-retrieval-scopes section, the `raw/` folder semantics section, and updates synthesis citation forms.

- [ ] **Step 1: Update synthesis `sources:` example in Part II.2**

Find this exact text:

```
**Synthesis** also requires:

```yaml
question: "<verbatim>"
answered_at: YYYY-MM-DD
superseded_by: null | <wiki path of replacement synthesis>
sources:
  - qmd://<vault-name>/<path-in-vault>      # for each in-vault page cited
  - https://...                             # for each external source cited
```
```

Replace with:

```
**Synthesis** also requires:

```yaml
question: "<verbatim>"
answered_at: YYYY-MM-DD
superseded_by: null | <wiki path of replacement synthesis>
sources:
  - qmd://<vault-name>/<path-in-vault>      # for each in-vault wiki page cited
  - qmd://<vault-name>/raw/<slug>.md        # for each ingested raw document cited
  - https://...                             # for each non-ingested external source cited
```
```

- [ ] **Step 2: Add Part II.8 (three retrieval scopes) after Part II.7**

Find this exact text (the closing line of Part II.7, just before Part III header):

```
The triad (entity, concept, synthesis) remains framework-invariant — new types sit alongside.

---

# Part III: Procedure
```

Replace with:

```
The triad (entity, concept, synthesis) remains framework-invariant — new types sit alongside.

## II.8 Three retrieval scopes

Vault retrieval has three logical scopes:

| Scope | Content | When to use |
|---|---|---|
| `wiki` | curated typed pages (`type:` set; everything outside `raw/`) | Synthesizing answers; trust hierarchy lookups |
| `raw` | source documents under `raw/` (no `type:` field; minimal frontmatter) | Citing primary sources; finding original quotes |
| `hybrid` | both | Default for general questions; fallback when scope is unclear |

Implementation: scopes are realized via qmd path filters. Consult `qmd query --help` for current parameter names; the vault `CLAUDE.md` retrieval-primitives section documents the working syntax. The wiki-research skill scopes phase-2 retrieval per its playbook. The recall skill exposes scope as a flag for debugging.

## II.9 The `raw/` folder

The vault root contains a `raw/` folder for ingested clean source documents — defuddle output from web articles, manually placed papers/blog posts/repo docs.

**Properties:**
- Committed to git (vault stays portable).
- Indexed by qmd (gives the `raw` scope its content).
- Citable from synthesis pages as `qmd://<vault-name>/raw/<slug>.md`.

**Frontmatter (minimal):**

```yaml
source_url: https://...
ingested_at: YYYY-MM-DD
kind: article | paper | doc | repo | post
```

**`raw/` documents are NOT wiki pages:**
- No `type:` field — they're sources, not curated entries.
- No `date_updated:` — `ingested_at:` is the only timestamp.
- The lint pass excepts `raw/` from page-type frontmatter rules (Phase 5 lint section in vault `CLAUDE.md`).

**What goes in `raw/` vs `.research/`:**
- `raw/` — durable, citable source documents that future synthesis pages will reference. Indexed.
- `.research/` — ephemeral artifacts from deep-research runs (HTML/PDF reports, evidence stores, intermediate drafts). Not indexed.

A page that's still being filled in is a normal wiki page, not a `raw/` document and not a `.research/` artifact.

---

# Part III: Procedure
```

- [ ] **Step 3: Validate Part II changes**

```bash
grep -c '^## II\.' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n 'qmd://<vault-name>/raw/' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n 'three logical scopes' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
```

Expected: at least **9** Part II subsections (II.1–II.9); `qmd://<vault-name>/raw/` appears in multiple places; "three logical scopes" appears once.

(No commit.)

---

## Task 7: Update Part III Phase 1 + Phase 2 (raw/ bootstrap)

**Files:**
- Modify: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md`

Adds `raw/` folder bootstrap as Task 1.5 within Phase 1.

- [ ] **Step 1: Update README.md template content in Phase 1.3 to mention `raw/` and `obsidian-skills`**

Find this exact text:

```
## Layout

- `CLAUDE.md` — vault schema and workflow rules, auto-loaded by Claude Code.
- `.templates/` — frontmatter starters for entity / concept / synthesis pages.
- `.claude/skills/` — `wiki-research`, `recall`, and the pinned `deep-research` submodule.
- `.research/` — raw artifacts from deep-research runs (kept for traceability, not indexed).
- `<topic>/<subtopic>/` — actual wiki pages, organized by domain.
```

Replace with:

```
## Layout

- `CLAUDE.md` — vault schema and workflow rules, auto-loaded by Claude Code.
- `.templates/` — frontmatter starters for entity / concept / synthesis pages.
- `.claude/skills/` — first-party skills (`wiki-research`, `recall`, `update-vendors`) and the pinned submodules (`deep-research`, `obsidian-skills`).
- `.claude/hooks/` — SessionStart hook for qmd freshness checks.
- `raw/` — ingested source documents (web articles, papers); indexed by qmd as the `raw` retrieval scope.
- `.research/` — ephemeral deep-research artifacts (kept for traceability, not indexed).
- `<topic>/<subtopic>/` — actual wiki pages, organized by domain.
```

- [ ] **Step 2: Add Task 1.5 — Bootstrap `raw/` folder, after Task 1.4**

Find this exact text in Phase 1 (the Task 1.4 → Task 1.5 transition area; Task 1.5 in the base is the "First commit"):

```
### Task 1.5 — First commit

- [ ] **Step 1: Stage and commit**

```bash
git add .gitignore README.md .research/.gitkeep
git commit -m "scaffold: initialize <vault-name> repo (gitignore, README, .research)"
git status
```

Expected: clean tree after commit.
```

Replace with (renumbering: old 1.5 becomes 1.6, new 1.5 inserted):

```
### Task 1.5 — Create `raw/.gitkeep` and `raw/README.md`

- [ ] **Step 1: Create the directory and stub files**

```bash
mkdir -p raw
touch raw/.gitkeep
```

- [ ] **Step 2: Create `raw/README.md`**

Write `<vault-root>/raw/README.md` with verbatim content:

```markdown
# raw/ — ingested source documents

This folder holds durable source documents that the wiki cites — typically web pages converted to clean markdown (via the `defuddle` skill), papers, or doc snapshots.

## What goes here

- Web articles ingested for long-term grounding.
- Papers, blog posts, repo docs whose content the vault wants to reference.
- Any source that synthesis pages will cite by `qmd://<vault-name>/raw/<slug>.md`.

## What does NOT go here

- Wiki pages (entity / concept / synthesis) — those go in topic-first folders at the vault root.
- Ephemeral deep-research artifacts (HTML/PDF reports, evidence stores) — those go in `.research/`.
- Drafts in progress — those are normal wiki pages, not `raw/` documents.

## Frontmatter (required)

```yaml
source_url: https://...
ingested_at: YYYY-MM-DD
kind: article | paper | doc | repo | post
```

No `type:` field — `raw/` documents are NOT wiki pages.

## Indexing

`raw/` is indexed by qmd alongside the wiki content. Queries can scope to `wiki` (excludes `raw/`), `raw` (only `raw/`), or `hybrid` (both — default).
```

### Task 1.6 — First commit

- [ ] **Step 1: Stage and commit**

```bash
git add .gitignore README.md .research/.gitkeep raw/.gitkeep raw/README.md
git commit -m "scaffold: initialize <vault-name> repo (gitignore, README, .research, raw)"
git status
```

Expected: clean tree after commit.
```

- [ ] **Step 3: Validate Phase 1 changes**

```bash
grep -n '### Task 1\.5' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n '### Task 1\.6' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n 'raw/.gitkeep raw/README.md' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
```

Expected: each grep returns at least one match.

(No commit.)

---

## Task 8: Update Part III Phase 3 (add update-vendors skill)

**Files:**
- Modify: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md`

Adds `update-vendors` as a third committed skill alongside `recall` and `wiki-research`. Inserts as Task 3.4 (renumbering the existing Task 3.4 commit step to 3.5).

- [ ] **Step 1: Insert Task 3.4 — Write `update-vendors/SKILL.md`**

Find this exact text in Phase 3 (the Task 3.3 → Task 3.4 transition; Task 3.4 in the base is the "Commit skills"):

```
### Task 3.4 — Commit skills

- [ ] **Step 1: Stage and commit**

```bash
git add .claude/skills/
git commit -m "scaffold: add wiki-research and recall slash-command skills"
```
```

Replace with:

```
### Task 3.4 — `.claude/skills/update-vendors/SKILL.md`

- [ ] **Step 1: Create directory and file**

```bash
mkdir -p .claude/skills/update-vendors
```

Write `<vault-root>/.claude/skills/update-vendors/SKILL.md` verbatim:

````markdown
---
name: update-vendors
description: Use when the user asks to check or apply updates to the vault's vendored dependencies (qmd, Obsidian CLI, deep-research submodule, obsidian-skills submodule). Detects current vs latest versions, builds change summaries, presents per-dep verdicts, applies approved updates.
---

# /update-vendors — vendor autoupdate workflow

Detects upstream updates to vault dependencies and offers to apply them. The user always pulls the trigger; the skill never updates without explicit per-dep approval.

## Vendors covered

- **qmd** (CLI tool) — installed via package manager.
- **Obsidian CLI** (CLI tool) — installed via package manager.
- **deep-research** (git submodule at `.claude/skills/deep-research/`).
- **obsidian-skills** (git submodule at `.claude/skills/obsidian-skills/`).

## Workflow

### Phase 1 — Detect current versions

For each vendor, in parallel:

- **Submodules:**
  ```
  git -C .claude/skills/<name> rev-parse HEAD
  git -C .claude/skills/<name> describe --tags --always
  ```

- **CLI tools:**
  ```
  qmd --version
  obsidian --version
  ```

### Phase 2 — Detect latest available

- **Submodules:**
  ```
  git ls-remote <url> HEAD
  git ls-remote --tags <url>
  ```

- **CLI tools:** fetch upstream README/releases page; parse latest version. Reuse the install-helper recipe's parsing logic from the runbook's Phase 6.

### Phase 3 — Build change summaries

- **Submodules:** upstream commit log between pinned SHA and HEAD:
  ```
  git -C .claude/skills/<name> log --oneline <pinned-sha>..<latest-sha>
  ```
  Cap at ~20 lines; print a notice if more.

- **CLI tools:** release notes for new version (parse from releases page or CHANGELOG).

### Phase 4 — Present per-dep verdicts

For each vendor with an update available:

```
<name>: <current-version> → <latest-version> (<delta>)
   summary:
   <change summary, ~10 lines>

   update? [yes / show-full-diff / skip]
```

For deps that are up-to-date, just confirm:

```
<name>: pinned <version> (up to date)
```

Wait for user response per dep.

### Phase 5 — Apply approved updates

- **Submodules** (on `yes`):
  ```
  git submodule update --remote .claude/skills/<name>
  git add .claude/skills/<name>
  git commit -m "deps: bump <name> to <new-sha>"
  ```

- **CLI tools** (on `yes`): re-run install via the install-helper recipe (vault `CLAUDE.md` retrieval-primitives section documents the recipe).

### Phase 6 — Update freshness timestamp

After all per-dep decisions are made (regardless of skip/apply), touch the freshness marker:

```
date -u +%Y-%m-%dT%H:%M:%SZ > .research/.deps-last-checked
```

This silences the SessionStart weekly nudge until 7 days have passed.

## Failure modes

- **Network unavailable** for `git ls-remote` or upstream fetch: surface the error, ask if user wants to skip remaining checks or retry.
- **Submodule has uncommitted changes**: stop; treat as another session's in-progress work; ask user.
- **CLI version-detect fails** (e.g., `qmd --version` errors): surface, recommend reinstall via install-helper.
- **Install-helper fails** for a CLI update: surface the upstream error; old version remains in place.
````

### Task 3.5 — Commit skills

- [ ] **Step 1: Stage and commit**

```bash
git add .claude/skills/
git commit -m "scaffold: add wiki-research, recall, and update-vendors skills"
```
```

- [ ] **Step 2: Update wiki-research SKILL.md to reference UltraDeep default**

Find this exact text in the wiki-research SKILL.md content (Task 3.2 of the runbook):

```
## Workflow

The full phase-by-phase procedure lives in `playbook.md`. Read it before
starting. High-level:
```

Replace with:

```
## Default mode

When invoking the deep-research skill (Phase 5 of the playbook), the default mode is **UltraDeep**. Override only if the user explicitly says "quick research" or "standard research" in their original ask. The seed brief (Phase 4) explains the choice with one line: "vault-quality grounding wants thorough sourcing".

## Workflow

The full phase-by-phase procedure lives in `playbook.md`. Read it before
starting. High-level:
```

- [ ] **Step 3: Update wiki-research playbook.md Phase 5 default mode**

Find this exact text in the wiki-research playbook content (Task 3.3 of the runbook):

```
Default mode: Standard. Override only if the user explicitly said "quick
research", "deep research", or "ultradeep research" in their original ask.
```

Replace with:

```
Default mode: **UltraDeep**. Override only if the user explicitly says
"quick research" or "standard research" in their original ask. Vault-quality
grounding wants thorough sourcing — the seed brief should mention this so
the deep-research skill calibrates accordingly.
```

- [ ] **Step 4: Validate Phase 3 changes**

```bash
grep -n '### Task 3\.4 — `.claude/skills/update-vendors' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n 'wiki-research, recall, and update-vendors' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n 'Default mode: \*\*UltraDeep\*\*' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
```

Expected: each grep returns at least one match.

(No commit.)

---

## Task 9: Update Part III Phase 4 + add Phase 4.5 (obsidian-skills submodule, Obsidian CLI, SessionStart hook)

**Files:**
- Modify: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md`

Phase 4 gains a second submodule (obsidian-skills). New Phase 4.5 installs Obsidian CLI and scaffolds the SessionStart hook.

- [ ] **Step 1: Update Phase 4 — add Task 4.3 for obsidian-skills submodule**

Find this exact text (the Task 4.2 commit + Phase 5 transition):

```
### Task 4.2 — Commit

- [ ] **Step 1: Stage and commit**

```bash
git add .gitmodules .claude/skills/deep-research
git commit -m "scaffold: pin deep-research skill as submodule"
```

---

## Phase 5 — Vault `CLAUDE.md`
```

Replace with:

```
### Task 4.2 — Add the obsidian-skills submodule

- [ ] **Step 1: Add and initialize**

```bash
git submodule add https://github.com/kepano/obsidian-skills.git .claude/skills/obsidian-skills
git submodule update --init --recursive
```

Expected output: clone progress, ending with `done.`. The submodule is now populated.

- [ ] **Step 2: Verify**

```bash
ls .claude/skills/obsidian-skills/defuddle/SKILL.md
ls .claude/skills/obsidian-skills/obsidian-markdown/SKILL.md
test -f .claude/skills/obsidian-skills/defuddle/SKILL.md && echo "defuddle OK"
test -f .claude/skills/obsidian-skills/obsidian-markdown/SKILL.md && echo "obsidian-markdown OK"
cat .gitmodules
```

Expected:
- Both `SKILL.md` paths printed.
- `defuddle OK` and `obsidian-markdown OK`.
- `.gitmodules` content includes a second `[submodule]` block:

```
[submodule ".claude/skills/deep-research"]
    path = .claude/skills/deep-research
    url = https://github.com/199-biotechnologies/claude-deep-research-skill.git
[submodule ".claude/skills/obsidian-skills"]
    path = .claude/skills/obsidian-skills
    url = https://github.com/kepano/obsidian-skills.git
```

- [ ] **Step 3: Verify nested skill discovery**

In a fresh Claude Code session opened in `<vault-root>`, the system reminder should list both `defuddle` and `obsidian-markdown` (or the qualified `obsidian-skills/defuddle`, `obsidian-skills/obsidian-markdown`) as user-invocable skills.

If the skills appear: nested discovery works, no further action.

If the skills do NOT appear: scaffold thin wrapper SKILL.md files at `.claude/skills/defuddle/SKILL.md` and `.claude/skills/obsidian-markdown/SKILL.md` that delegate via `Skill(obsidian-skills/<name>, ...)` paths. Surface to the user that the workaround was applied.

### Task 4.3 — Commit submodules

- [ ] **Step 1: Stage and commit**

```bash
git add .gitmodules .claude/skills/deep-research .claude/skills/obsidian-skills
git commit -m "scaffold: pin deep-research and obsidian-skills as submodules"
```

---

## Phase 4.5 — Obsidian CLI install + SessionStart hook

The Obsidian CLI is needed for the lint pass (`obsidian unresolved`). The SessionStart hook keeps the qmd index fresh and nudges weekly to run `/update-vendors`.

### Task 4.5.1 — Install Obsidian CLI via the install-helper recipe

The install-helper recipe is documented in Phase 6 (Task 6.2). For the Obsidian CLI, run that recipe with these inputs:

- Repo URL for README fetch: `https://raw.githubusercontent.com/obsidianmd/obsidian-cli/main/README.md` (fall back to https://obsidian.md/cli if the README fetch fails).
- Expected binary name: `obsidian`.
- Verification command after install: `obsidian --version`.

The recipe's defaults apply: lowest-friction path first (brew on Darwin, scoop/winget on Windows, apt on Linux), prebuilt binary as fallback, language toolchains as third option, source build as last resort.

Print the recommendation, wait for user (default), or execute on explicit per-session opt-in. Verify with `obsidian --version`.

### Task 4.5.2 — Scaffold the SessionStart hook

- [ ] **Step 1: Create the hooks directory and script**

```bash
mkdir -p .claude/hooks
```

Write `<vault-root>/.claude/hooks/session-start.sh` verbatim:

```bash
#!/usr/bin/env bash
# SessionStart hook for the vault.
# - Checks qmd freshness and updates the index if stale.
# - Nudges to run /update-vendors if vendors haven't been checked in 7+ days.
# - Always exits 0; never blocks session start.

set -u
LOG=".research/.session-start.log"
mkdir -p "$(dirname "$LOG")" 2>/dev/null

log_err() { printf '%s\n' "$*" >> "$LOG" 2>/dev/null; }

# 1. qmd freshness check
if ! command -v qmd >/dev/null 2>&1; then
  printf 'qmd unavailable — see CLAUDE.md retrieval-primitives section for setup\n'
elif ! qmd status >/dev/null 2>&1; then
  printf 'qmd: status check failed — see Phase 6 setup or .research/.session-start.log\n'
  qmd status 2>>"$LOG"
else
  # Check for staleness: any .md newer than the qmd index
  newest_md=$(find . -name '*.md' -not -path './.*' -not -path './node_modules/*' -type f -printf '%T@\n' 2>/dev/null | sort -rn | head -1)
  index_mtime=$(qmd status --json 2>/dev/null | grep -o '"last_updated":[^,}]*' | head -1 | tr -d ' "' | cut -d: -f2-)
  if [ -n "$newest_md" ] && [ -n "$index_mtime" ] && [ "$(printf '%.0f' "$newest_md")" -gt "$(printf '%.0f' "$index_mtime" 2>/dev/null || echo 0)" ]; then
    qmd update >/dev/null 2>>"$LOG" && printf 'qmd: refreshed index\n'
    # If many new files lack embeddings, also embed
    qmd embed --check 2>/dev/null | grep -q 'pending' && qmd embed >/dev/null 2>>"$LOG" && printf 'qmd: re-embedded\n'
  fi
fi

# 2. Vendor-update weekly nudge
DEPS_FILE=".research/.deps-last-checked"
if [ ! -f "$DEPS_FILE" ]; then
  printf 'Vendor deps never checked — run /update-vendors to review.\n'
else
  last_check=$(cat "$DEPS_FILE" 2>/dev/null)
  last_epoch=$(date -d "$last_check" +%s 2>/dev/null || date -j -f '%Y-%m-%dT%H:%M:%SZ' "$last_check" +%s 2>/dev/null || echo 0)
  now_epoch=$(date +%s)
  age_days=$(( (now_epoch - last_epoch) / 86400 ))
  if [ "$age_days" -ge 7 ]; then
    printf 'Vendor deps last checked %s days ago — run /update-vendors to review.\n' "$age_days"
  fi
fi

exit 0
```

- [ ] **Step 2: Make it executable**

```bash
chmod +x .claude/hooks/session-start.sh
```

- [ ] **Step 3: Wire the hook into `.claude/settings.json`**

Write `<vault-root>/.claude/settings.json` verbatim (or merge if it exists):

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/session-start.sh"
          }
        ]
      }
    ]
  }
}
```

- [ ] **Step 4: Smoke-test the hook**

```bash
bash .claude/hooks/session-start.sh
echo "exit: $?"
```

Expected: zero or one short line of output, exit 0. (Errors are routed to `.research/.session-start.log`.)

### Task 4.5.3 — Commit Phase 4.5 work

- [ ] **Step 1: Stage and commit**

```bash
git add .claude/hooks/session-start.sh .claude/settings.json
git commit -m "scaffold: add SessionStart hook for qmd freshness + vendor-update nudge"
```

(Obsidian CLI install does not produce repo changes — it's installed system-wide.)

---

## Phase 5 — Vault `CLAUDE.md`
```

- [ ] **Step 2: Validate Phase 4 + 4.5 changes**

```bash
grep -n '### Task 4\.2 — Add the obsidian-skills submodule' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n '### Task 4\.3 — Commit submodules' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n '## Phase 4\.5' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n 'session-start.sh' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n 'kepano/obsidian-skills' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
```

Expected: each grep returns at least one match.

(No commit.)

---

## Task 10: Update Part III Phase 5 (vault `CLAUDE.md` template)

**Files:**
- Modify: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md`

Phase 5's template gains: three retrieval scopes section, auto-commit policy, vendor-autoupdate, lint pass using `obsidian unresolved`, references to defuddle/obsidian-markdown skills, and updated layout.

- [ ] **Step 1: Update the Layout section in the vault `CLAUDE.md` template**

Find this exact text inside Phase 5's `CLAUDE.md` template:

```
- `<topic>/<subtopic>/`     — see Placement rule below for where to put a new page
- `.templates/`             — frontmatter templates (entity, concept, synthesis)
- `.claude/skills/`         — committed skills (wiki-research, recall) and the
                              deep-research submodule
- `.research/`              — deep-research output workspace (NOT indexed)
- `.docs/` (optional)       — design notes, plans, internal docs
```

Replace with:

```
- `<topic>/<subtopic>/`     — see Placement rule below for where to put a new page
- `raw/`                    — ingested source documents (web articles, papers); minimal
                              frontmatter; indexed by qmd as the `raw` retrieval scope
- `.templates/`             — frontmatter templates (entity, concept, synthesis)
- `.claude/skills/`         — first-party skills (wiki-research, recall, update-vendors)
                              and pinned submodules (deep-research, obsidian-skills)
- `.claude/hooks/`          — SessionStart hook for qmd freshness
- `.research/`              — ephemeral deep-research artifacts (NOT indexed)
- `.docs/` (optional)       — design notes, plans, internal docs
```

- [ ] **Step 2: Add "Retrieval scopes" section after the Wikilinks section in the vault `CLAUDE.md` template**

Find this exact text (the end of the Wikilinks section, before Workflow):

```
Use [label](qmd://<vault-name>/path/to/page.md) for explicit URL form.
Use [label](https://...) for external links.

## Workflow
```

Replace with:

```
Use [label](qmd://<vault-name>/path/to/page.md) for explicit URL form.
Use [label](https://...) for external links.

## Retrieval scopes

Vault retrieval has three logical scopes:

| Scope | Content | When to use |
|---|---|---|
| `wiki` | curated typed pages (everything outside `raw/`) | Synthesizing answers; trust hierarchy lookups |
| `raw` | source documents under `raw/` | Citing primary sources; finding original quotes |
| `hybrid` | both | Default for general questions; fallback when scope is unclear |

Implementation: scopes are realized via qmd path filters. Consult `qmd query --help` for current parameter names. The wiki-research skill scopes phase-2 retrieval per its playbook (typically `hybrid` first, then narrowing). The recall skill exposes scope as a flag for debugging.

## Workflow
```

- [ ] **Step 3: Add "Auto-commit" section after the Concurrent-sessions section**

Find this exact text:

```
- If your hunk genuinely conflicts with theirs (same lines), pause and ask
  the user.

## Synthesis pages
```

Replace with:

```
- If your hunk genuinely conflicts with theirs (same lines), pause and ask
  the user.

## Auto-commit after user-approved writes

This vault commits after every user-approved write. Pattern:

1. Agent shows a diff summary before writing — file path, +/- line counts, key frontmatter fields touched, slug name for new pages. Full unified diff is available on request, not shown by default.
2. User approves explicitly ("yes", "looks good", or directed edits).
3. Agent writes the file(s).
4. Agent runs `git add <files-just-written>`. If other sessions' uncommitted hunks are present in the working tree, agent uses `git add -p` to chunk-stage only its own hunks (concurrent-sessions discipline above).
5. Agent commits with message scoped to the topic of the write (e.g., `wiki: add synthesis "what is the best ratio for X"`).
6. Agent reports the commit SHA.

**Boundary:** one commit per approved-write turn. No batching across turns. Multi-page writes approved together (e.g., synthesis + new entity + updated concept in one wiki-research run) become one commit.

Git history is the audit trail — there is no separate `log.md`.

## Synthesis pages
```

- [ ] **Step 4: Update the Skills section in the vault `CLAUDE.md` template**

Find this exact text:

```
## Skills

Two committed skills under `.claude/skills/`:

- **wiki-research** — disciplined research loop: search vault first,
  optionally invoke deep-research with vault context as a seed, cross-check,
  write a synthesis page. Use it for any non-trivial research request.
- **recall** — direct qmd query wrapper for raw debugging. Not used by
  wiki-research itself; for the human operator.

Plus one submodule:

- **deep-research** — pinned at `.claude/skills/deep-research/`, points at
  https://github.com/199-biotechnologies/claude-deep-research-skill. Drives
  multi-source web research with citation tracking. wiki-research calls
  into it as needed; you generally won't invoke it directly.

After cloning the vault, populate the deep-research submodule:

```
git submodule update --init --recursive
```

Periodically (e.g., monthly) check for upstream updates:

```
git submodule update --remote .claude/skills/deep-research
git -C .claude/skills/deep-research log --oneline -10
git add .claude/skills/deep-research && git commit -m "bump deep-research skill"
```
```

Replace with:

```
## Skills

Three committed first-party skills under `.claude/skills/`:

- **wiki-research** — disciplined research loop: search vault first, optionally invoke deep-research with vault context as a seed, cross-check, write a synthesis page. Default deep-research mode is **UltraDeep**; override only if user explicitly says "quick research" or "standard research". Use it for any non-trivial research request.
- **recall** — direct qmd query wrapper for raw debugging. Not used by wiki-research itself; for the human operator.
- **update-vendors** — vendor autoupdate workflow. Run periodically (or on the weekly nudge from the SessionStart hook).

Two pinned submodules:

- **deep-research** at `.claude/skills/deep-research/` — pins https://github.com/199-biotechnologies/claude-deep-research-skill. Drives multi-source web research with citation tracking. wiki-research calls into it as needed; you generally won't invoke it directly.
- **obsidian-skills** at `.claude/skills/obsidian-skills/` — pins https://github.com/kepano/obsidian-skills. Provides:
  - `obsidian-skills/defuddle` — clean-markdown extraction from web pages. Used by wiki-research to populate `raw/` ingest.
  - `obsidian-skills/obsidian-markdown` — wikilink/callout/property syntax reference. Used as authoring guidance.
  - Other sub-skills (`obsidian-bases`, `json-canvas`, `obsidian-cli`) are accessible but not advertised in this vault.

After cloning the vault, populate both submodules:

```
git submodule update --init --recursive
```

For upstream updates, run `/update-vendors` (the skill checks both submodules, the official Obsidian CLI, and qmd, then offers to apply). Don't bump submodules manually — `update-vendors` handles the commit messaging and freshness tracking.
```

- [ ] **Step 5: Add "Vendor dependency updates" section after the Skills section**

Find this exact text (just after the Skills section, before Skill-suppression):

```
## Skill-suppression
```

Insert this block immediately before:

```
## Vendor dependency updates

The vault depends on external components: qmd, the official Obsidian CLI, and two pinned submodules (`deep-research`, `obsidian-skills`). All are pinned for reproducibility but autoupdate is a first-class workflow.

Run `/update-vendors` to:
- Detect current vs latest version for each dep.
- Build change summaries (commit logs for submodules, release notes for CLIs).
- Present per-dep verdicts with options: yes / show-full-diff / skip.
- Apply approved updates (for submodules: `git submodule update --remote` + commit; for CLIs: re-run install via the install-helper recipe in this file's retrieval-primitives section).

The SessionStart hook nudges weekly when `.research/.deps-last-checked` is older than 7 days. Never blocks, never auto-runs — only the user pulls the trigger.

## Skill-suppression
```

- [ ] **Step 6: Update the Lint section to use `obsidian unresolved`**

Find this exact text:

```
## Lint (run before each commit)

Walk the five passes:

1. **Frontmatter completeness.** Every changed `.md` file has `type:` (one
   of the recognized types) and `date_updated:`. Entities have `kind:`.
   Syntheses have `question:`, `answered_at:`, `superseded_by:`,
   `sources:`.
2. **Wikilink resolution.** Every `[[name]]` resolves case-insensitively
   (with hyphens / underscores / spaces equivalent) to an existing `.md`
   file or a declared alias. Broken links are flagged with the source file
   and a guess at intent.
3. **Orphan pages.** Pages with no inbound `[[links]]` from anywhere in
   the vault. Surface; user decides whether to add citations or accept.
4. **Stale `date_updated`.** Synthesis pages older than 180 days are
   flagged for re-research. Entity / concept staleness is informational
   only.
5. **Supersession integrity.** Every `superseded_by:` resolves to a real
   `type: synthesis` file; chains don't loop.

Group findings by severity:
- **Hard** (frontmatter missing required fields, supersession cycle) →
  block commit unless user overrides.
- **Soft** (broken wikilinks, orphans, stale syntheses) → surface; user
  decides.
- **Informational** (entity/concept staleness ages) → list for awareness.
```

Replace with:

```
## Lint (run before each commit)

Walk these passes:

1. **Frontmatter completeness.** Every changed `.md` file has `type:` (one of the recognized types) and `date_updated:`. Entities have `kind:`. Syntheses have `question:`, `answered_at:`, `superseded_by:`, `sources:`. **Exception: `raw/` documents** require `source_url:`, `ingested_at:`, `kind:` and have no `type:` or `date_updated:` field.

2. **Wikilink resolution.** Run `obsidian unresolved` against the vault root. Surface any unresolved links it reports. (The official CLI handles Obsidian's case-insensitive matching with spaces/hyphens/underscores equivalent — what you'd otherwise reimplement.) If `obsidian` CLI is unavailable, fall back to walking `[[name]]` references manually with case-insensitive matching; surface "obsidian CLI unavailable, on home-grown lint" once.

3. **Orphan pages.** Wiki pages with no inbound `[[links]]` from anywhere in the vault. Surface; user decides whether to add citations or accept. (`raw/` documents are excepted — they're not expected to have inbound wikilinks from other `raw/` documents.)

4. **Stale `date_updated`.** Synthesis pages older than 180 days are flagged for re-research. Entity / concept staleness is informational only. `raw/` documents are excepted.

5. **Supersession integrity.** Every `superseded_by:` resolves to a real `type: synthesis` file; chains don't loop.

Group findings by severity:
- **Hard** (frontmatter missing required fields, supersession cycle, `obsidian unresolved` reports a broken non-`raw/` link) → block commit unless user overrides.
- **Soft** (orphans, stale syntheses) → surface; user decides.
- **Informational** (entity/concept staleness ages) → list for awareness.
```

- [ ] **Step 7: Update the Retrieval primitives section to add the install-helper reference**

Find this exact text in the Retrieval primitives section:

```
**Setup commands (user-run only):**

```bash
qmd collection add . --name <vault-name> --mask '**/*.md'
# After this, run `qmd status` and confirm the indexed file count matches
# the count of `.md` files outside dotfolders. If dotfolder content leaks
# in, replace the single command above with one `qmd collection add` per
# top-level topic folder, all sharing `--name <vault-name>`.
qmd context add /                  "<identity-statement>"
qmd context add qmd://<vault-name> "<identity-statement>"
qmd embed
```

**Update commands (user-run only):**

```bash
qmd update
# qmd embed   # only if many new files were added
```

**Never run automatically.** The agent never invokes `qmd collection add`,
`qmd embed`, `qmd update`, or `qmd ingest <url>`. Only `qmd query`, `qmd
search`, `qmd get`, `qmd multi-get`, `qmd status` are agent-callable.
```

Replace with:

```
**Setup commands (user-run only):**

```bash
qmd collection add . --name <vault-name> --mask '**/*.md'
# After this, run `qmd status` and confirm the indexed file count matches
# the count of `.md` files outside dotfolders (raw/ is included by design).
# If dotfolder content leaks in, replace the single command above with one
# `qmd collection add` per top-level topic folder + raw, all sharing
# `--name <vault-name>`.
qmd context add /                  "<identity-statement>"
qmd context add qmd://<vault-name> "<identity-statement>"
qmd embed
```

**Update commands (agent-run automatically from SessionStart hook, also user-runnable):**

```bash
qmd update
qmd embed   # only if many new files were added
```

**Install commands (user-run by default; agent-runnable on per-session opt-in via the install-helper recipe).** The recipe is documented in Phase 6 of the runbook (this is reproduced here for runtime reference):

1. Fetch `https://raw.githubusercontent.com/tobilu/qmd/main/README.md` (or the GitHub releases page if the README fetch fails).
2. Parse install methods: `brew`, `scoop`, `winget`, `apt`, `cargo install`, prebuilt-binary download, `bun install -g`, `npm install -g`, source build.
3. Detect environment: `uname -s`, `uname -m`, `command -v` checks for each package manager.
4. Rank by friction (lowest first): platform-native PM with autoupdate (brew/scoop/winget/apt) → prebuilt binary → language toolchain (cargo/bun/npm) → source build.
5. Present recommendation + up to 2 alternatives. Default action: print and wait. On explicit per-session opt-in, agent executes.
6. Verify with `qmd status`.

**Agent automation policy:**

- **Always agent-callable:** `qmd query`, `qmd search`, `qmd get`, `qmd multi-get`, `qmd status`, `qmd update`, `qmd embed` (the last two are idempotent index ops; the SessionStart hook runs them automatically when stale).
- **Agent-callable on per-session opt-in:** qmd install (via install-helper recipe).
- **Never agent-run:** `qmd collection add`, `qmd ingest <url>` (vault-scope-changing).
```

- [ ] **Step 8: Update the Failure modes summary in the vault `CLAUDE.md` template**

Find this exact text:

```
## Failure modes (summary)

- qmd not installed → Read+Grep fallback; offer install via the setup
  block above.
- qmd MCP crash mid-session → one-shot retry; on second failure, fall back
  for the rest of the session.
- deep-research submodule empty → tell the user to run
  `git submodule update --init --recursive`; do not proceed.
- Index corruption → suggest `qmd update` or `qmd embed -f`; user runs.
```

Replace with:

```
## Failure modes (summary)

- qmd not installed → Read+Grep fallback; offer install via the install-helper recipe in the Retrieval primitives section.
- qmd MCP crash mid-session → one-shot retry; on second failure, fall back for the rest of the session.
- deep-research or obsidian-skills submodule empty → tell the user to run `git submodule update --init --recursive`; do not proceed.
- Index corruption → suggest `qmd update` or `qmd embed -f`; user runs.
- Obsidian CLI unavailable → home-grown wikilink-resolution fallback in the lint pass; surface "on home-grown lint" once.
- SessionStart hook errored → check `.research/.session-start.log`; never blocks the session.
```

- [ ] **Step 9: Validate Phase 5 changes**

```bash
grep -n '## Retrieval scopes' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n '## Auto-commit after user-approved writes' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n '## Vendor dependency updates' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n 'obsidian unresolved' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n 'obsidian-skills/defuddle' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
grep -n 'install-helper recipe' /Users/alepar/AleCode/llm-wiki/RUNBOOK.md
```

Expected: each grep returns at least one match.

(No commit.)

---

## Task 11: Update Part III Phase 6 + Phase 7 + Part IV (install-helper recipe; final commit)

**Files:**
- Modify: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md`

Replaces Phase 6's hardcoded install block with the install-helper recipe; updates Phase 7 verification to cover new components; updates Part IV failure modes; commits RUNBOOK.md as a single cohesive commit.

- [ ] **Step 1: Replace Phase 6 Task 6.2 (install) with the install-helper recipe**

Find this exact text:

```
### Task 6.2 — Print install commands (Case A only)

- [ ] **Step 1: Tell the user qmd needs installing**

Print verbatim:

> qmd is not installed. Run these three commands, then tell me when Claude Code has been restarted:
>
> ```bash
> # 1. Install qmd CLI (Bun preferred per upstream)
> bun install -g @tobilu/qmd      # or: npm install -g @tobilu/qmd
>
> # 2. Install the official Claude Code plugin
> claude plugin marketplace add tobi/qmd
> claude plugin install qmd@qmd
>
> # 3. Restart Claude Code so the plugin's qmd MCP server launches
> ```

- [ ] **Step 2: Wait for user confirmation**

After "done" / "installed" / equivalent, re-run `qmd status` to verify install. If still `command not found`, surface the error and ask which command failed.
```

Replace with:

```
### Task 6.2 — Install qmd via the install-helper recipe (Case A only)

The install-helper recipe is the canonical install workflow for any external CLI tool the runbook depends on (qmd here, Obsidian CLI in Phase 4.5, future deps via the same recipe). It avoids hardcoding install commands that go stale as upstream packaging evolves.

- [ ] **Step 1: Fetch latest install guidance**

```bash
curl -fsSL https://raw.githubusercontent.com/tobilu/qmd/main/README.md > /tmp/qmd-README.md 2>/dev/null \
  || curl -fsSL 'https://github.com/tobilu/qmd/releases/latest' > /tmp/qmd-releases.html
```

If both fetches fail, surface the network error and stop.

- [ ] **Step 2: Parse install methods from README**

Read `/tmp/qmd-README.md` (or releases page). Extract install commands matching these patterns:
- `brew install` / `brew tap`
- `scoop install` / `scoop bucket`
- `winget install`
- `apt install` / `apt-get install`
- `cargo install`
- Prebuilt binary: `curl ... | sh` or release-page download
- `bun install -g`
- `npm install -g`
- Source build: `git clone ... && cd ... && make install`

Build a list: `[{method, command, platform-applicability, notes}]`.

- [ ] **Step 3: Detect environment**

```bash
echo "OS: $(uname -s)"
echo "Arch: $(uname -m)"
for tool in brew scoop winget apt cargo bun npm; do
  command -v "$tool" >/dev/null 2>&1 && echo "$tool: present" || echo "$tool: missing"
done
```

- [ ] **Step 4: Rank candidates by friction (lowest first)**

1. Platform-native package manager with autoupdate (brew on Darwin / scoop on Windows / winget on Windows / apt on Linux/Debian) — IF that PM is present in the environment AND the README mentions it.
2. Prebuilt binary via release-page download or install script.
3. Language toolchain manager (cargo / bun / npm) — IF that toolchain is already present.
4. Source build — last resort.

- [ ] **Step 5: Present recommendation to the user**

Print the top recommendation + up to 2 alternatives. Example for a mac user with brew:

```
Recommended: brew install tobilu/qmd/qmd
Alternatives:
  - prebuilt binary: curl -fsSL https://github.com/tobilu/qmd/releases/.../install.sh | sh
  - bun: bun install -g @tobilu/qmd

Run the recommended command? [agent-runs / I'll-run-it / show-other-options]
```

- [ ] **Step 6: Execute or wait**

Default: print and wait for user. If user explicitly says "you run it" or equivalent, agent executes the recommended command via Bash (this is a per-session opt-in to Invariant 4's install-only carve-out).

If the user runs it themselves, wait for "done" / "installed" / equivalent.

- [ ] **Step 7: Install the qmd Claude Code plugin**

After the qmd CLI is on PATH, install the plugin and restart Claude Code:

```bash
claude plugin marketplace add tobi/qmd
claude plugin install qmd@qmd
# then restart Claude Code so the plugin's qmd MCP server launches
```

- [ ] **Step 8: Verify post-install**

```bash
qmd status 2>&1
```

Expected: `qmd status` runs successfully. If still `command not found`, surface the upstream error verbatim and ask which step failed.
```

- [ ] **Step 2: Update Phase 6 Task 6.4 verification to include `raw/`**

Find this exact text:

```
- The indexed-file count is plausible for a fresh vault (~5–10 files: CLAUDE.md, README.md, three `.templates/*.md`, plus the `wiki-research/playbook.md` and the two SKILL.md files — depending on whether the dotfolder mask traversal includes them).
```

Replace with:

```
- The indexed-file count is plausible for a fresh vault (~6–12 files: CLAUDE.md, README.md, three `.templates/*.md`, `raw/README.md`, plus the `wiki-research/playbook.md` and the three first-party SKILL.md files — depending on whether the dotfolder mask traversal includes them).
```

- [ ] **Step 3: Update Phase 7 Task 7.1 verification commit-graph expectations**

Find this exact text:

```
Expected: 5 commits, in this order (most recent first):

1. `scaffold: add vault CLAUDE.md ...`
2. `scaffold: pin deep-research skill as submodule`
3. `scaffold: add wiki-research and recall slash-command skills`
4. `scaffold: add entity/concept/synthesis page templates`
5. `scaffold: initialize <vault-name> repo (gitignore, README, .research)`

If the count or order differs, report to the user; some scaffolding step may have been merged or split.
```

Replace with:

```
Expected: 7 commits, in this order (most recent first):

1. `scaffold: add vault CLAUDE.md ...`
2. `scaffold: add SessionStart hook for qmd freshness + vendor-update nudge`
3. `scaffold: pin deep-research and obsidian-skills as submodules`
4. `scaffold: add wiki-research, recall, and update-vendors skills`
5. `scaffold: add entity/concept/synthesis page templates`
6. `scaffold: initialize <vault-name> repo (gitignore, README, .research, raw)`

(Plus an initial empty commit if `git init` produced one — usually 6 commits total in newer git versions, 7 if older.)

If the count or order differs significantly, report to the user; some scaffolding step may have been merged or split.
```

- [ ] **Step 4: Update Phase 7 Task 7.2 smoke-test step 1 to mention raw/ scope**

Find this exact text:

```
- Phase 2 runs the qmd query, gets ~0 results (the vault has no domain content yet), routes to Phase 4.
```

Replace with:

```
- Phase 2 runs the qmd query at the `wiki` scope, gets ~0 results (the vault has no domain content yet), routes to Phase 4.
```

- [ ] **Step 5: Update Part IV failure modes table**

Find this exact text in Part IV:

```
| Failure | Phase | Recovery |
|---|---|---|
| `git init` fails (permission, existing repo) | 1.1 | Check directory permissions; if a `.git/` already exists, surface to user — do not auto-delete. |
| Submodule clone fails (network, URL typo) | 4.1 | Verify network. Re-run `git submodule update --init --recursive`. If the URL in `.gitmodules` is wrong, edit it manually and re-add. |
| `qmd: command not found` after install | 6.2 | Verify `bun install -g` placed the binary on PATH; the user may need to restart the shell or add bun's global bin to `$PATH`. |
| `qmd embed` fails (out of disk, model download fails) | 6.3 | Surface the qmd error verbatim; defer to qmd upstream documentation. The vault is still usable on Read+Grep fallback. |
| `qmd collection add` indexes dotfolders | 6.3 | Drop the collection (`qmd collection remove <vault-name>`) and re-add per top-level topic folder (deferred until topics exist; for now, the framework files are an acceptable cost). |
| `wiki-research` skill not listed in Claude Code | 6.4 step 3 | Verify `.claude/skills/wiki-research/SKILL.md` exists and has valid YAML frontmatter. Restart Claude Code. |
| Deep-research submodule clone succeeds but `SKILL.md` is missing | 4.1 | Check the submodule SHA; the upstream repo's HEAD may have moved the file. Pin to a known-good SHA. |
| Index corruption | post-Phase 7 | Suggest `qmd update` or `qmd embed -f`; user runs. |
| qmd MCP crash mid-session | post-Phase 7 | One-shot retry; on second failure, fall back to Read+Grep for remainder of session; surface "qmd MCP unavailable, on grep fallback" once. |
```

Replace with:

```
| Failure | Phase | Recovery |
|---|---|---|
| `git init` fails (permission, existing repo) | 1.1 | Check directory permissions; if a `.git/` already exists, surface to user — do not auto-delete. |
| Submodule clone fails (network, URL typo) | 4.1, 4.2 | Verify network. Re-run `git submodule update --init --recursive`. If the URL in `.gitmodules` is wrong, edit it manually and re-add. |
| obsidian-skills submodule populates but nested `SKILL.md` files don't trigger skill discovery | 4.2 step 3 | Apply the wrapper-SKILL.md fallback (thin SKILL.md files at `.claude/skills/<name>/SKILL.md` that delegate to nested paths). |
| Obsidian CLI install fails | 4.5.1 | Surface the install-helper output verbatim. Lint pass falls back to home-grown wikilink resolution. |
| SessionStart hook errors | 4.5.2 | Hook always exits 0; check `.research/.session-start.log` for stderr. Common cause: `qmd status --json` flag not supported by older qmd — update qmd via `/update-vendors`. |
| `qmd: command not found` after install | 6.2 | Verify install placed the binary on PATH; the user may need to restart the shell or add the package manager's global bin to `$PATH`. Re-run the install-helper recipe. |
| Install-helper README parse fails | 6.2 step 2 | Surface to user; offer to fall back to the releases page or to a known last-good install command for the tool. |
| `qmd embed` fails (out of disk, model download fails) | 6.3 | Surface the qmd error verbatim; defer to qmd upstream documentation. The vault is still usable on Read+Grep fallback. |
| `qmd collection add` indexes dotfolders | 6.3 | Drop the collection (`qmd collection remove <vault-name>`) and re-add per top-level topic folder + `raw/` (deferred until topics exist; for now, the framework files are an acceptable cost). |
| `wiki-research` skill not listed in Claude Code | 6.4 step 3 | Verify `.claude/skills/wiki-research/SKILL.md` exists and has valid YAML frontmatter. Restart Claude Code. |
| Deep-research or obsidian-skills submodule clone succeeds but expected `SKILL.md` is missing | 4.1, 4.2 | Check the submodule SHA; the upstream repo's HEAD may have moved the file. Pin to a known-good SHA. |
| Index corruption | post-Phase 7 | Suggest `qmd update` or `qmd embed -f`; user runs. |
| qmd MCP crash mid-session | post-Phase 7 | One-shot retry; on second failure, fall back to Read+Grep for remainder of session; surface "qmd MCP unavailable, on grep fallback" once. |
| `/update-vendors` reports network errors | post-Phase 7 | User can skip remaining checks or retry; pinned versions remain in place. |
```

- [ ] **Step 6: Update Part IV rollback section**

Find this exact text:

```
## Rollback (un-instantiating a vault)

If the framework turns out to be a bad fit:

1. Remove the qmd plugin: `claude plugin uninstall qmd`
2. Remove the vault's qmd collection: `qmd collection remove <vault-name>`
3. Optionally `bun remove -g @tobilu/qmd`
4. The vault remains intact as a directory of markdown files. It is portable to any other tool (Obsidian, plain text editor, a different search engine) without modification.

The vault's content is the source of truth; qmd is a derived index.
```

Replace with:

```
## Rollback (un-instantiating a vault)

If the framework turns out to be a bad fit:

1. Remove the qmd plugin: `claude plugin uninstall qmd`
2. Remove the vault's qmd collection: `qmd collection remove <vault-name>`
3. Optionally uninstall qmd via the same package manager used to install it (the install-helper recipe documented which one).
4. Optionally uninstall the Obsidian CLI similarly.
5. The vault remains intact as a directory of markdown files. It is portable to any other tool (Obsidian, plain text editor, a different search engine) without modification.

The vault's content is the source of truth; qmd is a derived index. The submodules under `.claude/skills/` are also derived (vendored copies); deleting `.claude/` entirely leaves the wiki content untouched.
```

- [ ] **Step 7: Validate full RUNBOOK.md**

```bash
cd /Users/alepar/AleCode/llm-wiki

# Sanity counts
wc -l RUNBOOK.md
grep -c '^## Phase ' RUNBOOK.md      # expected: ~9 (0,1,2,3,4,4.5,5,6,7,8)
grep -c '^# Part ' RUNBOOK.md        # expected: 4 (I, II, III, IV)
grep -c '^### Task ' RUNBOOK.md      # expected: at least 25

# No unbound vault-name placeholders outside intentional contexts
# (template content has <vault-name>; the instantiating session binds it)
grep -c '<vault-name>' RUNBOOK.md    # expected: many (templates) — just a sanity check

# All new components mentioned
grep -n 'three retrieval scopes' RUNBOOK.md
grep -n 'install-helper' RUNBOOK.md
grep -n 'update-vendors' RUNBOOK.md
grep -n 'obsidian-skills' RUNBOOK.md
grep -n 'SessionStart' RUNBOOK.md
grep -n 'auto-commit' RUNBOOK.md     # case-insensitive needed:
grep -ni 'auto-commit' RUNBOOK.md
grep -n 'UltraDeep' RUNBOOK.md
grep -n 'obsidian unresolved' RUNBOOK.md

# Invariants: should be 9 numbered items
grep -c '^[0-9]\. \*\*' RUNBOOK.md
```

Expected:
- `wc -l` shows 1900–2400 lines (vs 1611 in base).
- 9+ Phase sections.
- 4 Part headers.
- 25+ Task subsections.
- Each new-feature grep returns matches.
- 9+ numbered invariants.

If any check fails, fix the corresponding section before committing.

- [ ] **Step 8: Commit RUNBOOK.md**

```bash
cd /Users/alepar/AleCode/llm-wiki
git add RUNBOOK.md
git commit -m "$(cat <<'EOF'
docs: add RUNBOOK.md (consolidated llm-wiki vault instantiation runbook)

Faithful integration of the family/docs/plans base runbook with all design
decisions from the meta-design spec:

- Three retrieval scopes (wiki/raw/hybrid) with new raw/ folder for ingested
  source documents.
- Two pinned submodules: deep-research (existing) and obsidian-skills (new,
  kepano) with defuddle (web→clean MD ingest) and obsidian-markdown
  (authoring syntax) sub-skills.
- Three first-party committed skills: wiki-research, recall, update-vendors.
- Auto-commit policy after user-approved writes (Invariant 9).
- SessionStart hook for qmd index freshness + weekly vendor-update nudge.
- Install-helper recipe replaces hardcoded install commands; agent fetches
  upstream README, parses methods, ranks by friction, recommends best for
  environment, defaults to print-and-wait.
- Obsidian CLI integration: install via helper recipe, used for
  `obsidian unresolved` in lint.
- wiki-research default deep-research mode flipped to UltraDeep.
- Invariant 4 revised (qmd update/embed agent-allowed; collection add /
  ingest still user-only).
- Invariant 7 split (raw/ ingest permitted; non-raw external sources still
  cited by URL only).

Source: docs/superpowers/specs/2026-05-09-llm-wiki-meta-design.md

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
git log --oneline
```

Expected: 5 commits in log:
1. RUNBOOK.md (this commit)
2. CLAUDE.md + README.md
3. prior-art.md
4. .gitignore
5. design spec

---

## Task 12: Final cross-cutting validation

**Files:**
- Read-only: all four target files

Validates that all four artifacts are coherent with each other and with the spec. No commits unless an issue surfaces.

- [ ] **Step 1: Confirm all artifacts are present and committed**

```bash
cd /Users/alepar/AleCode/llm-wiki
git status                                    # expected: clean
git log --oneline | wc -l                     # expected: 5
ls -la README.md CLAUDE.md RUNBOOK.md docs/prior-art.md
```

Expected: clean tree; 5 commits; all four files exist and are non-empty.

- [ ] **Step 2: Cross-reference check — README and CLAUDE point at the right things**

```bash
grep 'RUNBOOK.md' /Users/alepar/AleCode/llm-wiki/README.md
grep 'RUNBOOK.md' /Users/alepar/AleCode/llm-wiki/CLAUDE.md
grep 'docs/prior-art.md' /Users/alepar/AleCode/llm-wiki/README.md
grep 'docs/prior-art.md' /Users/alepar/AleCode/llm-wiki/CLAUDE.md
grep 'CLAUDE.md' /Users/alepar/AleCode/llm-wiki/README.md
```

Expected: each grep returns at least one match.

- [ ] **Step 3: Spec coverage check — every spec section maps to an artifact**

Open the spec at `/Users/alepar/AleCode/llm-wiki/docs/superpowers/specs/2026-05-09-llm-wiki-meta-design.md` alongside the four output files. For each spec section, confirm an output covers it:

| Spec section | Covered by |
|---|---|
| §1 Summary, §2 Goals/non-goals | CLAUDE.md (Identity, Philosophy) + README.md (intro) |
| §3 Approach (faithful integration) | RUNBOOK.md structure (Parts I–IV preserved with insertions) |
| §4 Repo layout | Actual files present in repo |
| §5.1 Three retrieval scopes | RUNBOOK.md Part II.8 + Phase 5 vault `CLAUDE.md` "Retrieval scopes" section |
| §5.2 raw/ folder | RUNBOOK.md Part II.9 + Phase 1.5 + raw/README.md template |
| §5.3 Synthesis citation forms | RUNBOOK.md Part II.2 (sources frontmatter) |
| §6 Procedure changes | RUNBOOK.md Phases 1, 1.5, 3, 4, 4.5, 5, 6, 7 |
| §7 Auto-commit policy | RUNBOOK.md Invariant 9 + Phase 5 vault `CLAUDE.md` "Auto-commit" section |
| §8 SessionStart hook | RUNBOOK.md Phase 4.5 (session-start.sh + settings.json) |
| §9 Install-helper recipe | RUNBOOK.md Phase 6 Task 6.2 + reproduced in vault `CLAUDE.md` retrieval-primitives |
| §10 Vendor autoupdate | RUNBOOK.md Phase 3 update-vendors SKILL.md + Phase 5 vault `CLAUDE.md` "Vendor dependency updates" |
| §11 Invariant changes | RUNBOOK.md Part I.3 (invariants 4, 7 revised; 9 added) |
| §12 Skills landscape | RUNBOOK.md Phase 3 + Phase 4 + vault `CLAUDE.md` Skills section |
| §13 Build sequence | This plan (executed) |
| §14 Open questions | RUNBOOK.md mentions submodule pin-to-main with `/update-vendors` for bumps; nested-skill-discovery fallback documented |
| §15 Sources/credits | CLAUDE.md "Sources / credits" + docs/prior-art.md |

If any row's mapping is missing, return to the corresponding task and add the content. No changes needed if all rows map.

- [ ] **Step 4: Final sanity grep — no leaked TODO/TBD markers**

```bash
cd /Users/alepar/AleCode/llm-wiki
grep -rn 'TBD\|TODO\|FIXME\|XXX' --include='*.md' . || echo "no markers"
```

Expected: `no markers` (the `--include='*.md'` should not flag anything in the spec or plan documents either, but if it does, those are existing files and acceptable — the focus is on RUNBOOK.md, CLAUDE.md, README.md, prior-art.md).

If markers appear in any of the four target files, fix and amend the relevant commit.

- [ ] **Step 5: Verify external URLs reachable (optional, network-dependent)**

```bash
for url in \
  https://github.com/199-biotechnologies/claude-deep-research-skill \
  https://github.com/kepano/obsidian-skills \
  https://github.com/tobilu/qmd \
  https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f \
  https://github.com/NiharShrotri/llm-wiki \
  https://github.com/ekadetov/llm-wiki ; do
  echo -n "$url: "
  curl -sfo /dev/null -w '%{http_code}\n' "$url" --max-time 5
done
```

Expected: each URL returns `200`. If any return `404`, the corresponding section in CLAUDE.md / RUNBOOK.md / prior-art.md needs the URL corrected.

(No commit; this is verification only.)

---

## Self-Review Notes

After completing Tasks 1–12, the implementation matches the spec. Key spec requirements coverage:

- **§5 doc model:** raw/ folder + three scopes — Tasks 6, 7, 10.
- **§6 procedure changes:** all phase modifications — Tasks 7–11.
- **§7 auto-commit:** Invariant 9 + vault CLAUDE.md section — Tasks 5, 10.
- **§8 SessionStart hook:** session-start.sh + settings.json — Task 9.
- **§9 install-helper:** Phase 6 + reproduced in vault CLAUDE.md — Tasks 10, 11.
- **§10 vendor autoupdate:** update-vendors SKILL.md + vault CLAUDE.md section + SessionStart nudge — Tasks 8, 9, 10.
- **§11 invariants:** 4 revised, 7 split, 9 added — Task 5.
- **§12 skills landscape:** wiki-research updated to UltraDeep, update-vendors added, obsidian-skills submodule, lint with `obsidian unresolved` — Tasks 8, 9, 10.

No placeholder steps; every `Edit` instruction has the exact `old_string` from the base runbook and the exact `new_string` to substitute.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-05-09-llm-wiki-meta-repo.md`. Two execution options:

**1. Subagent-Driven (recommended)** — Dispatch a fresh subagent per task; review between tasks; fast iteration. Best fit here because each task is self-contained and the parent agent's context stays clean for review.

**2. Inline Execution** — Execute tasks in this session using executing-plans; batch execution with checkpoints for review.

Which approach?
