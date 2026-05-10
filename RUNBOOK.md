# LLM-Wiki Vault Instantiation Runbook

> A single self-sufficient procedure for spinning up a new markdown wiki vault, designed to be fed verbatim to a fresh Claude Code session along with a chosen domain. The session collects three inputs (vault name, identity statement, target directory), then mechanically scaffolds a git-tracked vault with qmd hybrid retrieval, a pinned deep-research submodule, and committed `wiki-research` + `recall` slash-command skills.
>
> **No other file needs to be consulted.** Every literal template, every command, every invariant is reproduced inline. The companion design spec (`docs/superpowers/specs/2026-05-09-llm-wiki-meta-design.md`) contains rationale and alternatives but is not required for execution.

## How to use this runbook

Open a fresh Claude Code session in any directory. Give it this file. Say:

> Execute the vault instantiation runbook at `<path-to-this-file>`.

The session will:

1. Read this file end-to-end.
2. Create a TodoWrite list with one item per task.
3. Run **Phase 0** to ask the human for `<vault-name>`, `<identity-statement>`, `<vault-root>`.
4. Run **Phases 1–5** to scaffold files inside `<vault-root>`, committing per phase.
5. Run **Phase 6** to print qmd install/setup commands for the human to execute (the agent never auto-runs qmd's mutating operations).
6. Run **Phase 7** to verify the four success criteria from Part III.

The vault is operational at the end of Phase 7. The first real wiki page is the human's domain content and gets written in a separate session via `/wiki-research`.

---

# Part I: What you're building

## I.1 The shape

A self-contained git repository whose entire content is markdown — typed pages organized into topic-first folders, plus a few committed dotfiles for skills and templates. There is no application server, no database. qmd indexes the markdown for sub-second hybrid retrieval (BM25 + vector + LLM rerank); the `tobi/qmd` Claude Code plugin exposes that index over MCP. A `deep-research` git submodule extends retrieval to multi-source web research when the vault has gaps.

```
┌────────────────────────────────────────────────────────────┐
│ Layer 1: Workflow prose (vault CLAUDE.md, wiki-research    │
│   playbook)                                                │
│   Intent only. "Search the vault first." "Tier results    │
│   by trust level." "Cross-check before writing." Tool-     │
│   agnostic, hand-editable.                                 │
└────────────────────────────────────────────────────────────┘
                           │ (consults)
                           ▼
┌────────────────────────────────────────────────────────────┐
│ Layer 2: Retrieval primitives section in vault CLAUDE.md   │
│   Translates intent to mechanism. Owns: collection name,   │
│   query selection, fallback chain, install/setup/update    │
│   sub-workflow, freshness signal, citation convention,     │
│   "never run automatically" rule.                          │
└────────────────────────────────────────────────────────────┘
                           │ (invokes)
                           ▼
┌────────────────────────────────────────────────────────────┐
│ Layer 3: qmd (CLI + MCP plugin, stdio transport)           │
│   One collection per vault. Per-vault index at             │
│   <vault-root>/.qmd/index.sqlite (driven by .mcp.json      │
│   + .qmd/index.yml committed at the vault root).           │
└────────────────────────────────────────────────────────────┘
```

Two sidecar submodules pin at `.claude/skills/`:
- `deep-research` — invoked by `wiki-research` when the vault lacks coverage. Output lands in `.research/<topic-slug>_<YYYYMMDD>/` (excluded from indexing by being dot-prefixed) and the curated synthesis page lands in the vault normally.
- `obsidian-skills` (kepano) — provides `defuddle` (web→clean markdown for `raw/` ingest) and `obsidian-markdown` (wikilink/callout/property syntax reference for authoring).

Source documents ingested for long-term grounding land in a vault-root `raw/` folder, indexed by qmd alongside the wiki content. The vault therefore has three retrieval scopes: `wiki` (curated typed pages), `raw` (source documents), and `hybrid` (both).

The qmd index is per-vault: SQLite at `<vault-root>/.qmd/index.sqlite` (gitignored), wired via committed `.mcp.json` (sets `INDEX_PATH` and `QMD_CONFIG_DIR` env vars for the qmd MCP server) and `.qmd/index.yml` (qmd's per-collection config). No cross-vault index contamination.

## I.2 What this vault is NOT

- **Not a coach, journaling system, or project tracker.** No periodic files (no daily diary, no weekly retro), no goals, no habits. If the use case wants those, this framework is the wrong starting point.
- **Not multi-actor.** No `users/<name>/`, no per-actor private state. Single-vault, single-domain.
- **Not auto-mutating.** The agent never auto-edits without showing a diff first; never auto-supersedes; never silently rewrites a synthesis answer. Every write requires explicit user approval.
- **Not lifecycle-driven.** No "session start" context-load step. The agent searches when asked. The qmd freshness check (Part II.4) is the one exception, and it's silent on the happy path.
- **No behavioral guardrails by default.** No "never says" patterns, no tone register rules. The vault is a librarian. Domains that need behavioral rules add their own section in the resulting `CLAUDE.md`.

## I.3 Invariants the executor must preserve

These are framework-invariant. The executor must not relax them based on perceived domain need:

1. **Three core page types: `entity`, `concept`, `synthesis`.** Domains may add additional types alongside these, but never in place of them.
2. **Page type lives in frontmatter, never folder path.** No `entities/`, `concepts/`, or `synthesis/` top-level folders.
3. **Synthesis pages are write-once.** Never edit an existing synthesis page to change the answer. Write a new synthesis at a new slug; set the old page's `superseded_by:` to point at the new one.
4. **Vault-content-changing qmd operations are never agent-run.** `qmd collection add` (initial setup) and `qmd ingest <url>` (pulls external content) are user-run only; agent prints the commands and waits. Index-only operations `qmd update` and `qmd embed` are idempotent and don't change vault content — the agent runs these automatically (typically from the SessionStart hook). qmd install is agent-guided via the install-helper recipe (Phase 6) and user-executed by default; the user may opt the agent in to executing the install per session. The vault binds qmd to its own per-vault index via `INDEX_PATH` / `QMD_CONFIG_DIR` env vars in `.mcp.json` (committed); the agent never edits `.mcp.json` mid-session.
5. **Topic-first folders, never type-first.** Page paths are `<topic>/<subtopic>/<slug>.md`, not `entities/<slug>.md`.
6. **`.research/` is exclusively for deep-research output.** Not for working notes, scratchpads, drafts. A page being filled in over multiple sessions is a normal wiki page, not a `.research/` artifact.
7. **External web sources are not copied into the vault as wiki pages.** When long-term grounding is needed, sources may be ingested into `raw/` as cleaned-up markdown via the defuddle skill. Synthesis pages cite ingested sources by `qmd://<vault-name>/raw/<slug>.md` and non-ingested sources by `https://...`. The vault stays portable and human-readable; `raw/` documents have minimal frontmatter (no `type:` field) so they're never confused with wiki pages.
8. **Concurrent-sessions discipline.** Treat any uncommitted changes the executor did not author as belonging to another session. Use `git add -p` to stage only own hunks.
9. **Auto-commit follows user-approved writes.** Agent shows a diff summary before any write (file path, +/- line counts, key frontmatter fields touched). User approves explicitly. Agent writes, then commits with a topic-shaped message and reports the SHA. One commit per approved-write turn — no batching across turns. Full unified diff is available on user request but not shown by default. Git history is the audit trail; there is no separate `log.md`.

---

# Part II: Document model

## II.1 The three core page types

| Type | When to use | Required frontmatter |
|---|---|---|
| `entity` | A specific proper noun: a tool, a person, an organization, a model, a product, a dataset, a place. | `type`, `kind`, `date_updated` |
| `concept` | A *kind of thing*: an abstraction, a technique, a category, a methodology. | `type`, `date_updated` |
| `synthesis` | An answer to a specific question, citing other pages and external sources. **Write-once.** | `type`, `question`, `answered_at`, `superseded_by`, `sources`, `date_updated` |

Heuristic: if the subject is a single proper noun, prefer entity. If it's a kind of thing, prefer concept. **When unsure, prefer concept.**

## II.2 Frontmatter contracts

Every page MUST have:

```yaml
type: entity | concept | synthesis
date_updated: YYYY-MM-DD
```

**Entity** also requires:

```yaml
kind: tool | person | org | model | repo | dataset | product | place | <other domain-specific>
aliases: []          # true synonyms only — see II.5
homepage: <URL or empty>
```

**Concept** also accepts (optional but conventional):

```yaml
confidence: low | medium | high
related: [[...]]      # wikilinks to related concepts
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

The synthesis frontmatter is contractual: the lint pass (Part III Phase 5) checks all five required fields are present and well-formed, and `superseded_by:` resolves to a real synthesis page in the vault when set.

## II.3 Wikilinks

Use `[[name]]` for vault-internal references. Resolution is **case-insensitive** and treats spaces, hyphens, and underscores as equivalent — `[[DIY string alignment procedure]]` resolves to `diy-string-alignment-procedure.md`. This matches Obsidian's documented behavior; users opening the vault as an Obsidian vault get free preview.

External links: `[label](https://...)`.

For an explicit vault-internal URL (e.g., when citing from outside the vault), use `[label](qmd://<vault-name>/<path>)`.

## II.4 Subfolder placement rule

Within a topic, choose the subfolder by what the page is about:

| The page is about… | Goes in… |
|---|---|
| A specific use case | `<topic>/<use-case>/` |
| A vendor / brand / product entity | `<topic>/parts/` |
| A DIY tool entity | `<topic>/tools/` |
| Tied to one specific platform, not use-case-driven | `<topic>/<platform-name>/` |
| None of the above (cross-cutting) | `<topic>/` root |

The vault `CLAUDE.md` includes domain-specific examples after this table. The rule itself is invariant.

## II.5 Aliases

Aliases on entity pages are for **true synonyms only** — acronyms, alternate names, foreign phrasings (e.g., "ZN6" for the BRZ chassis page; "GPT-4" for the OpenAI o1-preview entity if the page is canonically titled differently). Do NOT add aliases that are just phrasing variants of the H1 or filename. If a target page accumulates more than ~3 aliases, prefer rewriting inbound links to a canonical form.

## II.6 Synthesis supersession

To update a synthesis answer:

1. Write a new synthesis page with the updated answer at a new slug.
2. Set the old page's `superseded_by:` to the new page's `qmd://<vault-name>/<path>`.
3. Mention the supersession in the new page's body ("supersedes [[old-slug]] — what changed: …").

Synthesis history stays queryable; no silent semantic drift.

Entity and concept pages, by contrast, are freely editable in place. Bump `date_updated` on every meaningful edit.

## II.7 Extending with new page types (optional, per-domain)

Domains may add additional page types beyond entity / concept / synthesis. Examples: `recipe` for cooking; `position` for finance tracking holdings; `routine` for fitness; `protocol` for medical research.

To add a new type:

1. Add `.templates/<type>.md` with the type's frontmatter starter.
2. Add a row to the page-type table in the vault `CLAUDE.md`, naming required frontmatter.
3. Add a lint rule for the new required fields.
4. Update the wiki-research playbook trust hierarchy if the new type slots above or below the defaults.

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

The procedure has 9 phases. Each phase ends with at least one git commit (so the work is recoverable mid-instantiation if anything goes wrong).

| Phase | Purpose | Mutates |
|---|---|---|
| 0 | Capture inputs | none |
| 1 | Empty repo bootstrap + per-vault qmd config | `<vault-root>/`, `git init`, `.gitignore`, `README.md`, `.research/.gitkeep`, `raw/.gitkeep`, `raw/README.md`, `.qmd/index.yml`, `.mcp.json` |
| 2 | Page type templates | `.templates/{entity,concept,synthesis}.md` |
| 3 | Committed skills | `.claude/skills/{recall,wiki-research,update-vendors}/...` |
| 4 | Pinned submodules (deep-research, obsidian-skills) | `.gitmodules`, `.claude/skills/{deep-research,obsidian-skills}/` |
| 4.5 | Obsidian CLI + SessionStart hook | `.claude/hooks/session-start.sh`, `.claude/settings.json` |
| 5 | Vault `CLAUDE.md` | `CLAUDE.md` (+ optional domain extensions) |
| 6 | qmd setup (install-helper recipe; printed; user-run) | qmd's local cache, not the vault |
| 7 | Verification | none |
| 8 | Hand-off (deferred: first real page) | none |

## Phase 0 — Capture inputs

### Task 0.1 — Collect vault name, identity, target dir

Ask the human, one question at a time, validating each:

- [ ] **Step 1: Vault name**

> What kebab-case name should the new vault use? This becomes the qmd collection name and the host of all `qmd://<name>/...` URLs in synthesis pages. It can't be cheaply renamed later — pick deliberately. Examples: `cooking-notes`, `quant-investing`, `physics-lab`.

Validate: matches `^[a-z][a-z0-9-]*[a-z0-9]$` (lowercase, kebab-case, no leading/trailing dash). On failure, surface the rule and ask again.

- [ ] **Step 2: Identity statement**

> Write a one-paragraph identity statement (3–5 sentences) describing what this vault is for. It will appear at the top of the vault's CLAUDE.md and as the qmd root context. Naming the domain, the kinds of pages this vault holds, and any non-goals (e.g., "not a journal", "not for personal notes") helps the agent stay on-task in future sessions.

Validate: 80–600 characters, non-empty. If too short, push for more detail; if too long, ask for a tighter version.

- [ ] **Step 3: Target directory**

> Where on disk should the new vault repo live? Provide an absolute path. The directory should not yet exist (or should be empty). Common pattern: `~/AleCode/<vault-name>` if you keep multiple vaults siblings.

Validate: absolute path. If the directory exists and is non-empty, ask the user to confirm overwrite or pick a different path.

- [ ] **Step 4: Confirm**

Echo back:

```
Vault name:        <vault-name>
Vault root:        <vault-root>
Identity statement:
<identity-statement>
```

Ask: "Confirm these before I start scaffolding?" Wait for explicit yes / no / edit. Re-loop on edits.

For the rest of the runbook, every command block uses these literal substitutions:
- `<vault-name>` → the kebab-case string from Step 1
- `<vault-root>` → the absolute path from Step 3
- `<identity-statement>` → the prose paragraph from Step 2

Do not invent additional placeholders later that aren't bound here.

---

## Phase 1 — Empty repo bootstrap

### Task 1.1 — Create directory and `git init`

- [ ] **Step 1: Create the directory**

```bash
mkdir -p <vault-root>
cd <vault-root>
```

Expected: directory exists; `cd` succeeds.

- [ ] **Step 2: Initialize git**

```bash
git init
git status
```

Expected: `Initialized empty Git repository...`, then `On branch main` (or `master`) with `No commits yet`.

If the default branch is `master`:

```bash
git branch -M main
```

### Task 1.2 — Write `.gitignore`

- [ ] **Step 1: Create `.gitignore`**

Write `<vault-root>/.gitignore` with verbatim content:

```
# OS / editor state
.DS_Store
*.swp
.idea/
.vscode/

# Optional Obsidian local state — comment out if the user shares it across machines
.obsidian/workspace*.json

# Per-vault qmd index (regenerable from the wiki content)
/.qmd/index.sqlite
/.qmd/*.sqlite-wal
/.qmd/*.sqlite-shm
```

Note on `.research/`: artifacts are committed by default (the audit trail for synthesis pages references in-vault paths). To exclude `.research/` from git, add `.research/` here — but synthesis citations to deep-research artifacts will dangle on a fresh clone.

### Task 1.3 — Write `README.md`

- [ ] **Step 1: Create `README.md`**

Write `<vault-root>/README.md` with this content (substituting `<vault-name>` and `<identity-statement>`):

````markdown
# <vault-name>

<identity-statement>

This is a curated markdown wiki. The vault's operating rules are in [`CLAUDE.md`](./CLAUDE.md). Search is via [qmd](https://github.com/tobi/qmd); follow the setup commands the agent prints on first session.

## Layout

- `CLAUDE.md` — vault schema and workflow rules, auto-loaded by Claude Code.
- `.templates/` — frontmatter starters for entity / concept / synthesis pages.
- `.claude/skills/` — first-party skills (`wiki-research`, `recall`, `update-vendors`) and the pinned submodules (`deep-research`, `obsidian-skills`).
- `.claude/hooks/` — SessionStart hook for qmd freshness checks.
- `.qmd/` — per-vault qmd config (`index.yml`) and SQLite index (`index.sqlite`, gitignored).
- `.mcp.json` — Claude Code MCP config; binds the qmd MCP server to this vault's index.
- `raw/` — ingested source documents (web articles, papers); indexed by qmd as the `raw` retrieval scope.
- `.research/` — ephemeral deep-research artifacts (kept for traceability, not indexed).
- `<topic>/<subtopic>/` — actual wiki pages, organized by domain.

## After cloning

```
git submodule update --init --recursive
```

Then run the qmd setup commands the vault `CLAUDE.md` prints on the first Claude Code session.
````

### Task 1.4 — Create `.research/.gitkeep`

- [ ] **Step 1: Create the directory and stub file**

```bash
mkdir -p .research
touch .research/.gitkeep
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

### Task 1.6 — Scaffold per-vault qmd config

- [ ] **Step 1: Create the .qmd/ directory and write .qmd/index.yml**

```bash
mkdir -p .qmd
```

Write `<vault-root>/.qmd/index.yml` verbatim (substituting `<vault-name>`):

```yaml
collections:
  <vault-name>:
    path: .
    pattern: "**/*.md"
    ignore:
      - .qmd/**
      - .research/**
      - .claude/**
      - .templates/**
```

- [ ] **Step 2: Write .mcp.json at the vault root**

Write `<vault-root>/.mcp.json` verbatim (no substitutions; `${CLAUDE_PROJECT_DIR}` is expanded by Claude Code at MCP-server-spawn time):

```json
{
  "mcpServers": {
    "qmd": {
      "command": "qmd",
      "args": ["mcp"],
      "env": {
        "INDEX_PATH": "${CLAUDE_PROJECT_DIR:-.}/.qmd/index.sqlite",
        "QMD_CONFIG_DIR": "${CLAUDE_PROJECT_DIR:-.}/.qmd"
      }
    }
  }
}
```

- [ ] **Step 3: Verify the files**

```bash
test -f .qmd/index.yml && echo "index.yml OK"
test -f .mcp.json && echo ".mcp.json OK"
grep -c 'CLAUDE_PROJECT_DIR' .mcp.json   # expected: 2
grep -c '<vault-name>' .qmd/index.yml    # expected: 1 (after substitution)
```

### Task 1.7 — First commit

- [ ] **Step 1: Stage and commit**

```bash
git add .gitignore README.md .research/.gitkeep raw/.gitkeep raw/README.md .qmd/index.yml .mcp.json
git commit -m "scaffold: initialize <vault-name> repo (gitignore, README, .research, raw, qmd config)"
git status
```

Expected: clean tree after commit.

---

## Phase 2 — Page type templates

### Task 2.1 — `.templates/entity.md`

- [ ] **Step 1: Create directory and file**

```bash
mkdir -p .templates
```

Write `<vault-root>/.templates/entity.md` verbatim:

```markdown
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

### Task 2.2 — `.templates/concept.md`

- [ ] **Step 1: Create file**

Write `<vault-root>/.templates/concept.md` verbatim:

```markdown
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

### Task 2.3 — `.templates/synthesis.md`

- [ ] **Step 1: Create file**

Write `<vault-root>/.templates/synthesis.md` verbatim:

````markdown
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
````

### Task 2.4 — Commit

- [ ] **Step 1: Stage and commit**

```bash
git add .templates/
git commit -m "scaffold: add entity/concept/synthesis page templates"
```

---

## Phase 3 — Committed skills

### Task 3.1 — `.claude/skills/recall/SKILL.md`

- [ ] **Step 1: Create directory and file**

```bash
mkdir -p .claude/skills/recall
```

Write `<vault-root>/.claude/skills/recall/SKILL.md` verbatim:

````markdown
---
name: recall
description: Direct qmd query against the vault's local index — for debugging
  retrieval, inspecting raw scores, or one-off lookups outside the
  wiki-research workflow. Use when you want raw qmd output. wiki-research
  uses qmd internally; do not route normal vault retrieval through this
  skill.
---

# /recall — direct qmd queries

A thin user-facing wrapper around the qmd CLI that lets you run hybrid
retrieval against the local qmd index without going through the
wiki-research loop. Useful for:

- Debugging retrieval quality ("why did qmd not surface this page?")
- Inspecting raw scores via `--explain`
- One-off lookups outside a research session

The wiki-research skill does not call `/recall` — it goes through the qmd
MCP path defined in the vault `CLAUDE.md`. `/recall` exists for the human
operator.

## Subcommands

| Invocation | Wraps | Use |
|---|---|---|
| `/recall <query>` | `qmd query --json "<query>"` | Hybrid retrieval, default rank |
| `/recall --explain <query>` | adds `--explain` | Surface RRF + rerank score traces |
| `/recall search <query>` | `qmd search --json "<query>"` | Pure BM25, no LLM |
| `/recall get <path-or-docid>` | `qmd get <arg> --full` | Retrieve full document content |
| `/recall status` | `qmd status` | Index health check |

## Anti-features

- **No mutating ops.** `collection add`, `embed`, `ingest`, and `update` are
  excluded by design. Run those directly via the commands the wiki
  `CLAUDE.md` prints in its retrieval-primitives section.
- **wiki-research does not call `/recall` itself.** Vault retrieval goes
  through the MCP path. `/recall` is user-only.
- **No query rewriting.** The user's input goes to qmd verbatim. The point
  is to see raw qmd behavior — translation belongs in wiki-research, not
  in the debugging tool.

## When qmd is unavailable

If `qmd` is not installed or the index is empty, `/recall` reports the
failure verbatim. It does not fall back to grep — the fallback chain is
for the wiki-research workflow, not for debug tooling that expects qmd
specifically.
````

### Task 3.2 — `.claude/skills/wiki-research/SKILL.md`

- [ ] **Step 1: Create directory and file**

```bash
mkdir -p .claude/skills/wiki-research
```

Write `<vault-root>/.claude/skills/wiki-research/SKILL.md` verbatim:

```markdown
---
name: wiki-research
description: Use when the user asks to research a topic in the vault, answer a question requiring sources, or fill a gap in vault knowledge — runs qmd-first retrieval, optional seeded web deep-research via the deep-research submodule, contradiction reconciliation, and vault page updates
---

# wiki-research

Disciplined research loop for a qmd-backed wiki vault. Search the vault first,
fall back to the web, cross-check, then write structured wiki pages with
citations.

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

## Default mode

When invoking the deep-research skill (Phase 5 of the playbook), the default mode is **UltraDeep**. Override only if the user explicitly says "quick research" or "standard research" in their original ask. The seed brief (Phase 4) explains the choice with one line: "vault-quality grounding wants thorough sourcing".

## Workflow

The full phase-by-phase procedure lives in `playbook.md`. Read it before
starting. High-level:

1. Receive query (clarify if vague — at most one question).
2. Run `qmd query` (MCP preferred, CLI fallback) and bucket results by
   frontmatter `type:` into the trust tiers above.
3. **Coverage check**: skip web research only if a non-stale synthesis page
   directly answers the question. Otherwise continue.
4. Build a seed brief for `deep-research` containing the question, vault
   evidence, and known gaps.
5. Invoke the `deep-research` skill (UltraDeep mode by default — see "Default mode" section above) with the seed
   brief; override its output path to `.research/<topic-slug>_<YYYYMMDD>/`.
6. Cross-check findings against the vault. **Stop and ask the user on any
   contradiction.**
7. Update existing entity/concept pages (with diff approval), propose new
   ones (with approval), write a synthesis page citing all sources.

## Hard rules

- The SessionStart hook handles index freshness automatically — do NOT run
  `qmd update` after each individual write. Running it ad-hoc at session end
  or to settle a known stale state is acceptable but not required.
- Synthesis pages have `question:`, `answered_at:`, `superseded_by:`
  (initially `null`), and a complete `sources:` list of every URL cited in
  the body.
- Wikilinks: `[[name]]` for vault-internal references, full URLs for external
  citations. `qmd://<vault>/<path>` only when an explicit URL form is needed.
- If `.claude/skills/deep-research/SKILL.md` is missing, the submodule is not
  populated. Tell the user to run `git submodule update --init --recursive`
  from the vault root and stop.

## Examples

See `examples.md` for worked traces (optional file; if present, lists three
cases: vault-only shortcut, full pipeline with one contradiction, full
pipeline introducing new entities).
```

### Task 3.3 — `.claude/skills/wiki-research/playbook.md`

- [ ] **Step 1: Create file**

Write `<vault-root>/.claude/skills/wiki-research/playbook.md` verbatim:

`````markdown
# wiki-research playbook

The seven phases of a research run. Follow in order. Do not skip phases unless
explicitly noted.

## Phase 0 — Preflight

Verify the deep-research submodule is populated:

```
ls .claude/skills/deep-research/SKILL.md
```

If missing, stop and tell the user:
> The `deep-research` submodule is not populated. From the vault root, run:
> ```
> git submodule update --init --recursive
> ```

Verify qmd availability per the vault `CLAUDE.md` retrieval-primitives section
(run `qmd status` or `mcp__qmd__status`). On Case A or B, accept
Read+Grep fallback for the rest of this run; surface the degraded mode once.

Determine the vault name from `qmd status`'s collection list. The vault name
hosts `qmd://<vault-name>/<path>` URLs; you'll cite them in synthesis
`sources:`.

## Phase 1 — Receive query

If the user's request is unambiguous (a clear question or topic), proceed.

If vague ("research X" with no scope, or a topic that could mean several
things), ask **at most one** clarifying question. Otherwise commit to a
reasonable interpretation and proceed — the user can correct later.

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

## Phase 3 — Coverage check

The bar for skipping web research is **high**: only skip when a tier-1
synthesis page exists whose `question:` directly matches the user's query
and is fresh.

**Direct match**: the synthesis page's `question:` field, read literally,
answers the same thing the user is asking. Paraphrasing is fine; semantic
equivalence is the standard.

**Freshness**: `(today - answered_at) <= 180` days by default. Override
threshold from the vault `CLAUDE.md` if the domain demands. If stale, do
NOT skip — re-run the research.

If a direct, fresh synthesis exists:

1. Quote the page's short answer (top of body).
2. Cite the page (`qmd://<vault-name>/<path>`) and any sources from its
   `sources:` frontmatter.
3. Ask the user: "This is already answered in `<page>`. Want me to use that,
   or run fresh research anyway?"
4. If the user accepts the existing answer, you are done. Do not write
   anything new.
5. If the user wants fresh research, continue to phase 4.

Otherwise, continue.

## Phase 4 — Seed brief

Compose a brief for the deep-research skill. Keep it under ~600 words.
Structure:

```
QUESTION:
<the user's question, verbatim>

EXISTING VAULT EVIDENCE:
- [tier 1 page title] (qmd://...): <one-line summary + key claim or quote>
- [tier 2 page title] (qmd://...): <one-line summary>
... (only the most relevant 3–6 items)

EXISTING WEB SOURCES IN QMD:
- <https://...> — <one-line relevance note>
...

OPEN QUESTIONS / KNOWN GAPS:
- <thing the vault doesn't cover or where confidence is low>
- <claim that needs verification>

INSTRUCTIONS FOR DEEP-RESEARCH:
- Use the vault evidence as a starting point, not as ground truth.
- Pay particular attention to <gaps>.
- Flag any source that contradicts the vault claims above.
```

## Phase 5 — Invoke deep-research

Default mode: **UltraDeep**. Override only if the user explicitly says
"quick research" or "standard research" in their original ask. Vault-quality
grounding wants thorough sourcing — the seed brief should mention this so
the deep-research skill calibrates accordingly.

### Phase 5a — search-cli precheck (if applicable)

If the implementer's environment uses `search-cli` as deep-research's
multi-provider web backend, run the precheck before invoking deep-research:

```bash
search update --check
search config check
for p in brave serper exa tavily perplexity jina; do
  printf '%-12s ' "$p:"
  search search -q "ping" -p "$p" -c 1 --json --quiet 2>/dev/null \
    | python3 -c "import json,sys;d=json.load(sys.stdin);print('OK' if d.get('status')=='success' and d.get('results') else 'FAIL')"
done
```

Stop and tell the user before invoking deep-research if any provider returns
FAIL or the version is out of date.

If `search-cli` is not in use, omit phase 5a.

### Phase 5b — Invoke

Invoke via the `Skill` tool with the seed brief; embed the output-path
override at the top of the prompt so the skill writes into the in-vault
workspace instead of `~/Documents/...`:

```
Skill(deep-research, "OUTPUT PATH OVERRIDE: write all artifacts into
.research/<topic-slug>_<YYYYMMDD>/ relative to the vault root. Do not use
the default ~/Documents/... location.

<seed brief from phase 4>")
```

If the skill version in use does not honor the prompt-level output-path
override, let it write to its default location, then move the artifacts
into `.research/<topic-slug>_<YYYYMMDD>/` after the run completes — the
synthesis page must cite the in-vault path, not `~/Documents/...`.

Capture the markdown report path. Use the report findings, not the
HTML/PDF artifacts.

If the deep-research skill fails (missing API keys, search-cli not
installed, etc.), surface the error to the user verbatim and stop.

## Phase 6 — Cross-check & contradictions

Read the deep-research markdown report. For each major claim in the report,
classify it against vault evidence:

- **Agreement**: web finding matches an existing vault claim. Cite both in
  the synthesis.
- **Extension**: web finding adds detail not previously in the vault.
  Capture for new pages or page updates.
- **Contradiction**: web finding disagrees with a vault claim. **Stop and
  surface to the user before any writes.**

For each contradiction, present:
- The vault claim (with the page path and the relevant quote).
- The web claim (with the source URL and quote).
- A short read of which seems more credible and why.

Ask the user how to resolve each one. Outcomes per item:

- **Trust vault** — drop the web claim from the synthesis.
- **Trust web** — supersede the affected synthesis page (write a new one,
  set old `superseded_by:`) or update the affected entity/concept page (with
  diff approval; phase 7).
- **Mark uncertain** — capture in synthesis under `## Open questions`; on
  the affected concept page, lower `confidence:` to `low` and note the
  disagreement.
- **Supersede** — only valid for synthesis pages; never directly edit them.

Apply each resolution in phase 7. Do not proceed until every contradiction
has a resolution.

## Phase 7 — Write/update vault pages

Strict order:

### 7a. Update existing entity/concept pages

For each entity/concept page touched by the research:

1. Read the current file.
2. Compose the proposed edit (typically: refine description, add a property,
   link a new related concept, refresh `date_updated`).
3. Show the unified diff to the user.
4. On approval, write the file and update `date_updated` to today.

### 7b. Propose new entity/concept pages

For major new things surfaced (a new tool, dataset, person, technique),
draft a page using the matching template under `.templates/`:

- New thing with a proper noun → an entity page from `.templates/entity.md`.
  Place per the placement rule (typically `<topic>/parts/` for vendor
  entities or `<topic>/tools/` for DIY tools).
- New abstraction or technique → a concept page from `.templates/concept.md`.
  Place per the placement rule.
- Unsure → prefer concept.

Show each draft to the user. On approval, write.

### 7c. Write the synthesis page

Frontmatter (per `.templates/synthesis.md`):

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

Body structure:

```markdown
# <The question>

<Short answer in 1–3 sentences. State the bottom line directly.>

## Reasoning

<Detailed reasoning. Use [[wikilinks]] to entities/concepts. Cite web sources
inline as [n] with a footnote section, or with the URL in parentheses on
first reference.>

## Open questions

<Anything not resolved. Include items where the user chose "mark uncertain"
in phase 6.>

## Detailed report

<Optional. Relative path to the deep-research artifact under `.research/...`
if the user wants the long-form linked.>
```

Place the synthesis page per the placement rule — typically in the
topic's root, not a subfolder, since syntheses are usually cross-cutting.

Write. Do NOT run `qmd update` — the session-end reminder handles that.

### 7d. Final summary

Tell the user:
- Files written (paths).
- Sources cited count (vault / web).
- Any unresolved items still flagged in `## Open questions` of the synthesis.
- Reminder: at session end, run `qmd update` to refresh the index.

## Failure modes

- **qmd unavailable**: surface the degraded mode once at phase 2; proceed
  with grep fallback. The wiki-research loop still works, just slower.
- **deep-research submodule empty**: caught in phase 0.
- **deep-research returns nothing useful**: tell the user; offer to retry
  with a different mode (Quick / Deep / UltraDeep) or to write a synthesis
  page based on vault evidence alone.
- **User rejects every diff**: respect that — do not write a synthesis
  page either. The research output is conversational only.
`````

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

- **qmd** (CLI tool) — installed via package manager. **Minimum version:** post-`b775592` (`b77559223025cbcff3f992df0bf01147497c3bab`, 2026-05-09 — fixes MCP `--index` plumbing). Older versions silently break per-vault index isolation.
- **Obsidian CLI** (CLI tool) — installed via package manager.
- **deep-research** (git submodule at `.claude/skills/deep-research/`).
- **obsidian-skills** (git submodule at `.claude/skills/obsidian-skills/`).

**Minimum-version policy:** for any vendor with a documented floor (currently only qmd), the skill's per-dep verdict in Phase 4 marks an upgrade as **REQUIRED** (not optional) when the installed version is below the floor. The user can still skip, but the framework flags this as a known-broken state.

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

For deps below their minimum-version floor (see "Vendors covered" above), prefix the verdict with **REQUIRED** so the user knows the framework considers this a known-broken state:

```
qmd: <current-version> → <latest-version> (BELOW MINIMUM FLOOR — REQUIRED upgrade)
   reason: per-vault index isolation broken without b775592 fix
   summary:
   <change summary>

   update? [yes / show-full-diff / skip — note: skipping leaves the vault in a known-broken state]
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

---

## Phase 4 — Deep-research submodule

### Task 4.1 — Add the submodule

- [ ] **Step 1: Add and initialize**

```bash
git submodule add https://github.com/199-biotechnologies/claude-deep-research-skill.git .claude/skills/deep-research
git submodule update --init --recursive
```

Expected output: clone progress, ending with `done.`. The submodule is now populated.

- [ ] **Step 2: Verify**

```bash
ls .claude/skills/deep-research/SKILL.md
test -f .claude/skills/deep-research/SKILL.md && echo "OK"
cat .gitmodules
```

Expected:
- File path printed.
- `OK`.
- `.gitmodules` content:

```
[submodule ".claude/skills/deep-research"]
    path = .claude/skills/deep-research
    url = https://github.com/199-biotechnologies/claude-deep-research-skill.git
```

(Indentation may use a tab instead of four spaces — both are git-canonical.)

If `SKILL.md` is missing, the submodule did not populate. Diagnose: check network, check the URL in `.gitmodules`, retry `git submodule update --init --recursive` (add `-v` for verbose).

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

# Bind subsequent qmd CLI invocations to this vault's per-vault index.
# (The MCP server gets the same env via .mcp.json; this covers the hook's
# own qmd status / qmd update / qmd embed calls.)
export INDEX_PATH="$PWD/.qmd/index.sqlite"
export QMD_CONFIG_DIR="$PWD/.qmd"

# 1. qmd freshness check
if ! command -v qmd >/dev/null 2>&1; then
  printf 'qmd unavailable — see CLAUDE.md retrieval-primitives section for setup\n'
elif ! qmd status >/dev/null 2>&1; then
  printf 'qmd: status check failed — see Phase 6 setup or .research/.session-start.log\n'
  qmd status 2>>"$LOG"
else
  # Check for staleness: any .md newer than the qmd index (portable across BSD/GNU find)
  if [ "$(uname -s)" = "Darwin" ] || [ "$(uname -s)" = "FreeBSD" ]; then
    newest_md=$(find . -name '*.md' -not -path './.*' -not -path './node_modules/*' -type f -exec stat -f '%m' {} \; 2>/dev/null | sort -rn | head -1)
  else
    newest_md=$(find . -name '*.md' -not -path './.*' -not -path './node_modules/*' -type f -printf '%T@\n' 2>/dev/null | sort -rn | head -1)
  fi
  index_mtime=$(qmd status --json 2>/dev/null | grep -o '"last_updated":[^,}]*' | head -1 | tr -d ' "' | cut -d: -f2-)
  if [ -n "$newest_md" ] && [ -n "$index_mtime" ] && [ "$(printf '%.0f' "$newest_md")" -gt "$(printf '%.0f' "$index_mtime" 2>/dev/null || echo 0)" ]; then
    qmd update >/dev/null 2>>"$LOG" && printf 'qmd: refreshed index (run `qmd embed` if many new files were added)\n'
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

### Task 5.1 — Write the vault `CLAUDE.md`

This is the largest single file in the scaffold. Reproduce the template below into `<vault-root>/CLAUDE.md`, applying these substitutions inline:

| Placeholder | Substitute with |
|---|---|
| `<Identity heading>` | An H1 like `# <vault-name> — Wiki Schema` (replace dashes in the vault-name with spaces and title-case if natural) |
| `<identity statement, one paragraph>` | The captured `<identity-statement>` from Phase 0 |
| `<vault-name>` | Every literal occurrence (~10 places: in command blocks, citation conventions, qmd:// URLs) |
| `<domain-specific examples>` | A short bullet list of 3–6 examples specific to the chosen domain (see Step 2 below) |

- [ ] **Step 1: Write the file**

Create `<vault-root>/CLAUDE.md` from this template:

````markdown
# <Identity heading>

<identity statement, one paragraph>

This is a curated knowledge wiki. qmd handles search; this file describes the
maintenance discipline.

## Layout

Topic-first folders, created on demand. No fixed top-level list — each topic
folder appears when its first page does.

- `<topic>/<subtopic>/`     — see Placement rule below for where to put a new page
- `raw/`                    — ingested source documents (web articles, papers); minimal
                              frontmatter; indexed by qmd as the `raw` retrieval scope
- `.templates/`             — frontmatter templates (entity, concept, synthesis)
- `.claude/skills/`         — first-party skills (wiki-research, recall, update-vendors)
                              and pinned submodules (deep-research, obsidian-skills)
- `.claude/hooks/`          — SessionStart hook for qmd freshness
- `.research/`              — ephemeral deep-research artifacts (NOT indexed)
- `.docs/` (optional)       — design notes, plans, internal docs

Page type (entity / concept / synthesis) lives only in frontmatter, not in the
folder path.

## Placement rule (where the page file goes)

Within a topic, choose the subfolder by what the page is about:

| Page is about… | Goes in… |
|---|---|
| A specific use case | `<topic>/<use-case>/` |
| A vendor / brand / product entity | `<topic>/parts/` |
| A DIY tool entity | `<topic>/tools/` |
| Tied to one specific platform, not use-case-driven | `<topic>/<platform-name>/` |
| None of the above (cross-cutting) | `<topic>/` root |

<domain-specific examples>

Topic folders themselves are created on demand; do not pre-scaffold empty ones.

## When to create which page type

- Single proper noun as the subject → entity.
- A *kind of thing* → concept.
- An answer to a question → synthesis.
- Unsure between entity/concept → prefer concept.

## Required frontmatter

Every page MUST have:
  type:         (entity | concept | synthesis)
  date_updated: (YYYY-MM-DD)

Entity pages also require:    kind:
Synthesis pages also require: question:, answered_at:, superseded_by:, sources:

## Wikilinks

Use [[name]] for short links. Resolution is case-insensitive and treats
spaces, hyphens, and underscores as equivalent — so [[DIY string alignment
procedure]] resolves to diy-string-alignment-procedure.md without needing
an alias. This matches Obsidian's documented behavior.

Aliases are for *true synonyms only* — acronyms, alternate names, foreign
phrasings. Do NOT add aliases that are just phrasing variants of the H1 or
filename. If a target page accumulates more than ~3 aliases, ask whether the
inbound links should be rewritten to a canonical form instead.

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

1. Search the vault first via the qmd MCP tools (or `/recall` for raw inspection)
   to find existing relevant pages.
2. Read the full pages qmd returns (frontmatter + body).
3. If a page already covers the topic, update it (bump date_updated).
   Otherwise, create a new page using the Placement rule above.
4. Show diffs (or full drafts for new pages) to the user before writing.
5. Run the lint checklist (below) before each commit; address all hard
   findings.
6. Commit one topic-shaped commit per session of work. The commit message
   describes the topic; the file list is recoverable via `git show --stat`.
7. After the commit lands, if any indexed paths changed:
   ```
   qmd update
   # (and qmd embed if many new files were added)
   ```
   The agent does NOT run these commands automatically — see Retrieval
   primitives below.

## Concurrent sessions

The vault may already contain uncommitted work from other Claude sessions
when you start. Treat any modification or untracked file you did not write
in this session as belonging to that other session and leave it alone:

- Don't commit, stash, revert, or "clean up" changes you didn't author.
- At commit time, use `git add -p` (chunk staging) to stage only your own
  hunks. Other sessions' hunks stay in the working tree, available to
  whoever is finishing that work.
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

Never edit an existing synthesis page to change the answer. Instead:
- Write a new synthesis page with the updated answer at a new slug.
- Set the old page's `superseded_by:` to the new page's `qmd://<vault-name>/<path>`.
- Mention the supersession in the new page's body.

## What goes into `raw/` (and what `qmd ingest` is NOT used for)

Source documents that the wiki cites for long-term grounding go into the vault's `raw/` folder as cleaned-up markdown — typically via the **defuddle** skill for web articles, or by manual placement for papers/blog posts. They are committed to git (vault stays portable) and indexed by qmd as the `raw` retrieval scope. Synthesis pages cite them via `qmd://<vault-name>/raw/<slug>.md`; non-ingested sources are still cited by their `https://...` URL.

The qmd-native `qmd ingest <url>` command (which adds external URLs to qmd's index without committing markdown to the vault) is **not used by this framework** — it would break vault portability since the index isn't reproducible from the repo alone. Per Invariant 4 the agent never runs it. Users may run it ad-hoc but should prefer the defuddle → `raw/` workflow above.

## Research artifacts stay in the vault

Working artifacts from research runs (deep-research reports, evidence
stores, source registries, draft synthesis, intermediate HTML/PDF) live
inside the vault, not in `~/Documents/` or other external locations. The
vault must be self-contained and portable.

- Research workspace: `.research/<topic-slug>_<YYYYMMDD>/`
  Dotted prefix keeps it out of normal browsing and out of qmd's index.
- Only the curated synthesis page gets ingested into qmd; the `.research/`
  artifacts are kept for traceability but not indexed.
- When invoking the deep-research skill, override its default
  `~/Documents/...` output path and point it at the in-vault `.research/`
  path.

`.research/` is **not** part of the wiki. It exists solely as a dump ground
for raw artifacts produced by the deep-research skill. Do not write anything
else there:
- No iterative work logs, session notes, scratchpads, plans, or drafts-in-
  progress.
- A page that's still being filled in is a normal wiki page, not a
  `.research/` artifact.
- If something genuinely doesn't belong in the vault and isn't deep-research
  output, keep it outside the vault entirely.

## Templates

See .templates/entity.md, .templates/concept.md, .templates/synthesis.md.

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

## Vendor dependency updates

The vault depends on external components: qmd, the official Obsidian CLI, and two pinned submodules (`deep-research`, `obsidian-skills`). All are pinned for reproducibility but autoupdate is a first-class workflow.

Run `/update-vendors` to:
- Detect current vs latest version for each dep.
- Build change summaries (commit logs for submodules, release notes for CLIs).
- Present per-dep verdicts with options: yes / show-full-diff / skip.
- Apply approved updates (for submodules: `git submodule update --remote` + commit; for CLIs: re-run install via the install-helper recipe in this file's retrieval-primitives section).

The SessionStart hook nudges weekly when `.research/.deps-last-checked` is older than 7 days. Never blocks, never auto-runs — only the user pulls the trigger.

## Skill-suppression

Claude Code in a vault session has many available general-purpose skills.
The vault role overrides general-purpose skill suggestions: a knowledge
request should not trigger a feature-dev framing. If the user explicitly
invokes a skill (e.g., `/superpowers:brainstorming`), proceed with that
skill — explicit user instruction takes precedence. Otherwise, stay in
vault mode.

## Retrieval primitives (qmd integration)

Vault retrieval goes through the `tobi/qmd` Claude Code plugin's MCP tools:
`mcp__qmd__query`, `get`, `multi_get`, `status`. CLI fallback is
`qmd <command>` via Bash, with `--json` always.

**Query selection** (intent → CLI form; MCP equivalents take the same params):

| Intent | CLI form |
|---|---|
| Natural-language search | `qmd query --collection <vault-name> --json` |
| Slug or exact-phrase | `qmd search --collection <vault-name> --json` |
| Known file by path | `qmd get <path>` |
| Glob batch retrieve | `qmd multi-get "<pattern>" --json` |

**Session-start freshness signal.** Run `qmd status` once per session and
branch:

- Case A — qmd not installed → fall back to Read+Grep; offer setup.
- Case B — installed, no collection → fall back; offer setup.
- Case C — index stale → recommend `qmd update`; proceed either way.
- Case D — fresh → silent.

**Per-vault index (this vault's qmd binding):**

The qmd MCP server in this vault is bound to a per-vault index by `.mcp.json` at the vault root. Because `.mcp.json` declares a server named `qmd` and Claude Code's scope precedence is Project > User > Plugin, this entry shadows the marketplace plugin's qmd server. MCP tools are therefore exposed as `mcp__qmd__*` (not `mcp__plugin_qmd_qmd__*`). The `.mcp.json` env block sets two env vars when the MCP server spawns:

- `INDEX_PATH=<vault-root>/.qmd/index.sqlite` — absolute path to this vault's SQLite index
- `QMD_CONFIG_DIR=<vault-root>/.qmd` — directory qmd reads `index.yml` (per-collection config) from

For manual CLI invocations (and the SessionStart hook), prefix the same env vars or rely on the hook's exports. Optional shell helper for ergonomics:

```bash
qmd-vault() { INDEX_PATH="$PWD/.qmd/index.sqlite" QMD_CONFIG_DIR="$PWD/.qmd" qmd "$@"; }
```

…then `qmd-vault status`, `qmd-vault embed`, etc. The env-prefixed form below is canonical.

**Setup commands (user-run only, env-prefixed for per-vault index):**

```bash
INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd \
  qmd collection add . --name <vault-name> --mask '**/*.md'
# After this, run a status check and confirm the indexed file count matches
# the count of `.md` files outside dotfolders (raw/ is included by design):
INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd qmd status
# Initial context entries (root + collection-scoped, both with the identity statement):
INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd \
  qmd context add /                  "<identity-statement>"
INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd \
  qmd context add qmd://<vault-name> "<identity-statement>"
INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd qmd embed
```

**Update commands (agent-run automatically from SessionStart hook, also user-runnable):**

```bash
INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd qmd update
INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd qmd embed   # only if many new files were added
```

**Install commands (user-run by default; agent-runnable on per-session opt-in via the install-helper recipe).** The recipe is documented in Phase 6 of the runbook (this is reproduced here for runtime reference):

1. Fetch `https://raw.githubusercontent.com/tobi/qmd/main/README.md` (or the GitHub releases page if the README fetch fails).
2. Parse install methods: `brew`, `scoop`, `winget`, `apt`, `cargo install`, prebuilt-binary download, `bun install -g`, `npm install -g`, source build.
3. Detect environment: `uname -s`, `uname -m`, `command -v` checks for each package manager.
4. Rank by friction (lowest first): platform-native PM with autoupdate (brew/scoop/winget/apt) → prebuilt binary → language toolchain (cargo/bun/npm) → source build.
5. Present recommendation + up to 2 alternatives. Default action: print and wait. On explicit per-session opt-in, agent executes.
6. Verify with `qmd status`.

**Agent automation policy:**

- **Always agent-callable** (with `INDEX_PATH` / `QMD_CONFIG_DIR` env from `.mcp.json` for MCP, or from the SessionStart hook for CLI): `qmd query`, `qmd search`, `qmd get`, `qmd multi-get`, `qmd status`, `qmd update`, `qmd embed` (the last two are idempotent index ops; the SessionStart hook runs them automatically when stale).
- **Agent-callable on per-session opt-in:** qmd install (via install-helper recipe).
- **Never agent-run:** `qmd collection add`, `qmd ingest <url>` (vault-scope-changing). The agent also never edits `.mcp.json` mid-session — that file is the per-vault index binding and is committed.

**Fallback chain.** When qmd is unavailable, fall back for the rest of the
session and surface "running on grep fallback" once:

| Intent | Fallback |
|---|---|
| Search | `Grep -r "<query>"` against vault root, excluding dotfolders |
| Find file by name/glob | `Glob` against vault root |
| Find orphans / broken links | walk vault root, parse frontmatter |

**Citation convention.** Cite hits as `[[file-slug]]`, not full paths. Cite
external sources as `[label](https://...)`.

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

## Failure modes (summary)

- qmd not installed → Read+Grep fallback; offer install via the install-helper recipe in the Retrieval primitives section.
- qmd MCP crash mid-session → one-shot retry; on second failure, fall back for the rest of the session.
- deep-research or obsidian-skills submodule empty → tell the user to run `git submodule update --init --recursive`; do not proceed.
- Index corruption → suggest `qmd update` or `qmd embed -f`; user runs.
- Obsidian CLI unavailable → home-grown wikilink-resolution fallback in the lint pass; surface "on home-grown lint" once.
- SessionStart hook errored → check `.research/.session-start.log`; never blocks the session.
````

- [ ] **Step 2: Fill in the placement-rule examples**

The template has `<domain-specific examples>` directly under the placement-rule table. Replace it with 3–6 worked examples specific to the chosen domain. Format:

```markdown
Examples for this vault:

- `<topic>/<use-case>/` — `<concrete-example>` (e.g. for a cooking vault: `pasta/weeknight-dinners/cacio-e-pepe.md`)
- `<topic>/parts/` — `<concrete-example>` (e.g. for a cooking vault: `pasta/parts/de-cecco-spaghetti.md`)
- `<topic>/tools/` — `<concrete-example>` (e.g. for a cooking vault: `pasta/tools/sodium-citrate-mac.md`)
- `<topic>/<platform-name>/` — `<concrete-example>` (e.g. for a cooking vault: `coffee/chemex/`)
- `<topic>/` root — `<concrete-example>` (e.g. for a cooking vault: `salt-as-a-flavor-multiplier.md`)
```

If unsure what fits the chosen domain, ask the user verbatim:

> What's a concrete example of each placement-rule row in your domain? I need 3–6 examples to fill in CLAUDE.md's placement section. If you can't think of one for a row, leave that row blank — placement rules without concrete examples are still valid.

- [ ] **Step 3: Verify the file**

```bash
head -3 CLAUDE.md
grep -c '<vault-name>' CLAUDE.md
grep -c '<identity statement' CLAUDE.md
grep -c '^## ' CLAUDE.md
```

Expected:
- First line: the H1 (`# <vault-name> — Wiki Schema` or similar).
- `<vault-name>` placeholder count: **0** (all substituted).
- `<identity statement` placeholder count: **0**.
- Level-2 section count: **at least 12**. If significantly fewer (≤8), the template was truncated; recopy.

### Task 5.2 — Optional: domain extensions

If the domain demands extensions per Part II.7, apply them now before commit.

- [ ] **Step 1: Ask the user about extensions**

> Before committing CLAUDE.md, does this domain need any of these extensions?
>
> 1. **Extra page types** — beyond entity / concept / synthesis. (e.g., `recipe`, `position`, `protocol`)
> 2. **Custom subfolder conventions** — beyond `<topic>/parts/`, `<topic>/tools/`, etc. (e.g., `<topic>/positions/`, `<topic>/conditions/`)
> 3. **Custom lint rules** — beyond the five default passes. (e.g., "every protocol cites at least one peer-reviewed source")
> 4. **Behavioral guardrails** — domain-specific "never says" rules.
>
> If yes, name which and provide enough detail to write them in. If no extensions, just say "no extensions" and we proceed.

- [ ] **Step 2: For each named extension, edit `CLAUDE.md`**

- **Extra page types**: add a row to the page-type table; add `.templates/<type>.md` (Step 3 below); add a lint rule for the new required fields.
- **Custom subfolder conventions**: add rows to the placement-rule table.
- **Custom lint rules**: add a numbered pass to the Lint section.
- **Behavioral guardrails**: add a new top-level section above `## Failure modes`.

- [ ] **Step 3: Add custom page-type templates if requested**

For each new type (e.g., `recipe`):

> What's the frontmatter shape for `type: <new-type>` pages? Required fields and any conventional optional fields.

Then write `<vault-root>/.templates/<new-type>.md`:

```markdown
---
type: <new-type>
<required field 1>:
<required field 2>:
date_updated: YYYY-MM-DD
---

# <Page Name>

<Body conventions for this type — one paragraph.>

## <Section 1 from user's spec>

## <Section 2 from user's spec>
```

### Task 5.3 — Commit

- [ ] **Step 1: Stage and commit**

```bash
git add CLAUDE.md
# If Task 5.2 added templates:
git add .templates/<new-type>.md   # for each new type
```

If no extensions:

```bash
git commit -m "scaffold: add vault CLAUDE.md (identity + workflow + lint + qmd integration)"
```

If extensions:

```bash
git commit -m "scaffold: add vault CLAUDE.md with <domain> extensions (<list>)"
```

Where `<list>` names the categories (page-types, subfolder-conventions, lint-rules, guardrails).

---

## Phase 6 — qmd setup (printed; user-run)

The agent never auto-runs qmd's mutating ops. All commands in this phase are printed for the user to execute; the agent waits between steps.

### Task 6.1 — Detect qmd state and version

- [ ] **Step 1: Run the freshness check (read-only — agent runs this)**

```bash
qmd status 2>&1
```

Branch on the result:

- **Case A — `qmd: command not found`** → continue to Task 6.2 (install).
- **Case B — `qmd status` runs but reports no collection matching `<vault-name>`** → continue to Step 2 (version-floor check), then Task 6.3 (setup).
- **Case C — `qmd status` reports a collection matching `<vault-name>`** → unexpected for a fresh repo. Surface to the user: "A qmd collection named `<vault-name>` already exists from a previous setup. Skip qmd setup and proceed to Task 6.4 verification?" Wait for explicit confirmation.

- [ ] **Step 2: Verify qmd version floor (Cases B and C)**

The vault binds qmd to a per-vault index via `INDEX_PATH` in `.mcp.json`. Pre-`b775592` (commit `b77559223025cbcff3f992df0bf01147497c3bab`, 2026-05-09) qmd silently ignores `INDEX_PATH` for the MCP server and falls back to the global index — vault isolation breaks without a user-visible signal. Verify the installed qmd is at or above the floor:

```bash
qmd --version 2>&1
```

If the version is unambiguously older (e.g., a tagged release that predates the fix, or a build date earlier than 2026-05-09), surface the floor requirement and route to the install-helper (Task 6.2) for an upgrade. The user may opt to skip the upgrade for now, but the runbook flags it as required and the produced vault will not have per-vault MCP isolation until upgraded.

If `qmd --version` doesn't include a parseable date or commit, the runbook can't auto-verify; surface this to the user and ask whether the install is recent enough (post-2026-05-09).

### Task 6.2 — Install qmd via the install-helper recipe (Case A only)

The install-helper recipe is the canonical install workflow for any external CLI tool the runbook depends on (qmd here, Obsidian CLI in Phase 4.5, future deps via the same recipe). It avoids hardcoding install commands that go stale as upstream packaging evolves.

- [ ] **Step 1: Fetch latest install guidance**

```bash
curl -fsSL https://raw.githubusercontent.com/tobi/qmd/main/README.md > /tmp/qmd-README.md 2>/dev/null \
  || curl -fsSL 'https://github.com/tobi/qmd/releases/latest' > /tmp/qmd-releases.html
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
Recommended: brew install tobi/qmd/qmd
Alternatives:
  - prebuilt binary: curl -fsSL https://github.com/tobi/qmd/releases/.../install.sh | sh
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

### Task 6.3 — Print setup commands

- [ ] **Step 1: Print verbatim** (with `<vault-root>`, `<vault-name>`, `<identity-statement>` substituted):

> Run these in `<vault-root>`. Each command is env-prefixed with `INDEX_PATH` and `QMD_CONFIG_DIR` so qmd writes to the per-vault index at `<vault-root>/.qmd/index.sqlite` (not the global default).
>
> ```bash
> cd <vault-root>
>
> # Add the vault as a qmd collection (creates .qmd/index.sqlite)
> INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd \
>   qmd collection add . --name <vault-name> --mask '**/*.md'
>
> # Initial context entries (root + collection-scoped, both with the identity statement)
> INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd \
>   qmd context add /                  "<identity-statement>"
> INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd \
>   qmd context add qmd://<vault-name> "<identity-statement>"
>
> # Embed (downloads ~2.1GB GGUF models on first install; subsequent runs are fast)
> INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd qmd embed
> ```

- [ ] **Step 2: Wait for confirmation, then verify**

After the user confirms:

```bash
INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd qmd status 2>&1
```

Verify:
- A collection named `<vault-name>` is listed.
- The collection's source path matches `<vault-root>`.
- The indexed-file count is plausible for a fresh vault (~6–12 files: CLAUDE.md, README.md, three `.templates/*.md`, `raw/README.md`, plus the `wiki-research/playbook.md` and the three first-party SKILL.md files — depending on whether the dotfolder mask traversal includes them).
- The SQLite file lives at `<vault-root>/.qmd/index.sqlite` (not at `~/.cache/qmd/index.sqlite`):
  ```bash
  test -f $(pwd)/.qmd/index.sqlite && echo "per-vault index OK" || echo "ERROR: index landed at the wrong path"
  ```

- [ ] **Step 3: Verify dotfolder exclusion**

If `qmd status` shows substantially more than ~10 files, dotfolders are leaking. Surface:

> The qmd collection includes more files than expected (`<count>` vs ~10). Dotfolders may be getting indexed. List what's indexed:
>
> ```bash
> qmd status --files | head -20
> ```
>
> If `.claude/`, `.templates/`, or `.research/` paths appear, drop the collection and re-add per topic folder. Pause for guidance.

If the count is plausible, proceed.

### Task 6.4 — Verify the success criteria

- [ ] **Step 1: `qmd status` returns the collection (env-prefixed)**

```bash
INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd qmd status 2>&1 | grep -i '<vault-name>'
```

Expected: a non-empty match line.

- [ ] **Step 2: A query against the empty(-ish) vault returns without erroring (env-prefixed)**

```bash
INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd \
  qmd query --collection <vault-name> --json "test" 2>&1
```

Expected: valid JSON, no traceback, no `Error:` prefix on stderr. The result list may be empty or contain the framework files — both are fine.

- [ ] **Step 2.5: Per-vault index file exists**

```bash
test -f $(pwd)/.qmd/index.sqlite && echo "per-vault index file OK"
test -f $(pwd)/.qmd/index.yml    && echo "per-vault index config OK"
test -f $(pwd)/.mcp.json         && echo ".mcp.json OK"
```

Expected: three OK lines.

- [ ] **Step 3: The wiki-research and recall skills are listed**

In a fresh Claude Code session opened in `<vault-root>`, the system reminder should list `wiki-research` and `recall` as user-invocable skills. Ask the user to confirm by checking `/help` or the skill list.

If the skills don't appear, verify the `.claude/skills/<name>/SKILL.md` files exist and have valid frontmatter (`name:` and `description:` keys present).

- [ ] **Step 4: The deep-research submodule's SKILL.md is present**

```bash
ls -la .claude/skills/deep-research/SKILL.md
```

Expected: file exists, non-empty.

If all four pass, proceed to Phase 7. If any fail, diagnose and remediate before continuing.

---

## Phase 7 — Final verification and hand-off

### Task 7.1 — Verify the commit graph

- [ ] **Step 1: Check `git status`**

```bash
git status
```

Expected: `nothing to commit, working tree clean`.

- [ ] **Step 2: Check the commit graph**

```bash
git log --oneline
```

Expected: 6 commits in this order (most recent first), or 7 if `git init` emitted an initial empty commit (uncommon in git ≥ 2.28):

1. `scaffold: add vault CLAUDE.md ...`
2. `scaffold: add SessionStart hook for qmd freshness + vendor-update nudge`
3. `scaffold: pin deep-research and obsidian-skills as submodules`
4. `scaffold: add wiki-research, recall, and update-vendors skills`
5. `scaffold: add entity/concept/synthesis page templates`
6. `scaffold: initialize <vault-name> repo (gitignore, README, .research, raw, qmd config)`

If the count or order differs significantly, report to the user; some scaffolding step may have been merged or split.

### Task 7.2 — Smoke-test wiki-research

- [ ] **Step 1: Pick a smoke-test query**

Ask the user:

> Smoke test: pick a question relevant to your domain that you'd expect this vault to answer eventually but doesn't yet. Example for a cooking vault: "what's the best ratio of salt to water for pasta?". Example for a quant vault: "how do I model tracking error in a direct-indexing portfolio?".

Wait for the smoke-test query.

- [ ] **Step 2: Invoke wiki-research (dry run only)**

```
Skill(wiki-research, "<the smoke-test query>")
```

Expected behavior, **stopping at Phase 5 (don't actually run deep-research)**:

- Phase 0 (preflight) passes — deep-research submodule is populated, qmd is available.
- Phase 1 accepts the query unmodified or asks at most one clarifying question.
- Phase 2 runs the qmd query at the `wiki` scope, gets ~0 results (the vault has no domain content yet), routes to Phase 4.

Additionally, before invoking wiki-research, verify the MCP server is bound to the per-vault index:

> Ask the agent in the fresh Claude Code session: "Call the qmd MCP `status` tool and report the index path it shows."
>
> Expected: the agent reports an index path of the form `<vault-root>/.qmd/index.sqlite`. If the path is `~/.cache/qmd/index.sqlite` (or any non-vault path), the `${CLAUDE_PROJECT_DIR}` expansion in `.mcp.json` failed to resolve — surface to the user, fall back to writing absolute paths into `.mcp.json` (substituting `<vault-root>` directly), and re-run.
- Phase 4 composes a seed brief acknowledging the vault has no existing evidence.
- Phase 5 — **stop**. The brief should be visibly well-formed; the actual deep-research run is for the user's first real session.

Tell the user:

> Smoke test passed. The wiki-research skill is operational on an empty vault. To write your first real page, invoke `/wiki-research <your real query>` in a fresh session — it will do the deep-research run, contradiction reconciliation, and write back to the vault.

### Task 7.3 — Hand-off

- [ ] **Step 1: Print the hand-off summary**

> Vault `<vault-name>` is operational at `<vault-root>`.
>
> Six scaffolding commits are in place (seven on older git). The qmd collection is set up and embedded against the per-vault index at `<vault-root>/.qmd/index.sqlite` (driven by the committed `.mcp.json` and `.qmd/index.yml`). Both submodules (deep-research, obsidian-skills) are pinned. The wiki-research, recall, and update-vendors skills are committed and discoverable. The SessionStart hook is active and the Obsidian CLI is installed.
>
> To write the first real page, either:
> 1. Invoke `/wiki-research <your-question>` and follow the skill's loop end-to-end; or
> 2. Manually write a page using the templates under `.templates/` and commit.
>
> When you're ready to push to a remote, set up a GitHub repo and add it as `origin`:
>
> ```bash
> gh repo create <vault-name> --private --source=. --remote=origin
> git push -u origin main
> ```
>
> The runbook ends here. The vault now operates per its own `CLAUDE.md` from this point forward.

---

## Phase 8 — Optional: write the first real wiki page (deferred)

This phase is intentionally not part of the runbook. The first real page is the user's domain content, not framework scaffolding — it should be created in a separate session with full attention.

When the user is ready, they invoke `/wiki-research <their first question>` and follow that skill's playbook. The runbook does not script this; the wiki-research skill is the operational primitive for all real wiki work.

---

# Part IV: Failure modes and recovery

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
| qmd < `b775592` silently uses global index | 6.1 | The version-floor check in 6.1 catches this; if missed, the Phase 7 MCP smoke test catches it via wrong index path. Recovery: bump qmd via install-helper (Task 6.2) or `/update-vendors`. |
| First-time MCP approval prompt confuses user on fresh session | 7.x | Claude Code prompts to approve the project's `.mcp.json`. Documented behavior. Tell user to approve; the prompt happens once per project per machine. |
| `${CLAUDE_PROJECT_DIR}` not expanded by older Claude Code | 7.x (smoke test) | Phase 7 smoke test catches this — the qmd MCP `status` tool reports the literal `${CLAUDE_PROJECT_DIR}` string (or `./.qmd/index.sqlite` after `:-` fallback) as the index path instead of the absolute vault path. Recovery: substitute `<vault-root>` directly into `.mcp.json` at scaffold time and re-commit. Surface the Claude Code version requirement: `${VAR}` expansion in `.mcp.json` env blocks is documented at https://code.claude.com/docs/en/mcp; if the user's Claude Code version predates this support, they need to upgrade. |
| `/update-vendors` reports network errors | post-Phase 7 | User can skip remaining checks or retry; pinned versions remain in place. |

## Rollback (un-instantiating a vault)

If the framework turns out to be a bad fit:

1. Remove the qmd plugin: `claude plugin uninstall qmd`
2. Remove the vault's qmd collection: `qmd collection remove <vault-name>`
3. Optionally uninstall qmd via the same package manager used to install it (the install-helper recipe documented which one).
4. Optionally uninstall the Obsidian CLI similarly.
5. The vault remains intact as a directory of markdown files. It is portable to any other tool (Obsidian, plain text editor, a different search engine) without modification.

The vault's content is the source of truth; qmd is a derived index. The submodules under `.claude/skills/` are also derived (vendored copies); deleting `.claude/` entirely leaves the wiki content untouched.

---

# Self-review (post-write)

This runbook merges the design spec (`docs/superpowers/specs/2026-05-09-llm-wiki-meta-design.md`) and the original plan into a single self-sufficient document. Coverage:

- Part I — framework summary, three-layer architecture, what-it-isn't, invariants. From spec §1, §2, §3.5, §5, §6, §7.8.
- Part II — document model, frontmatter contracts, wikilinks, supersession, type extensions. From spec §4.
- Part III — full procedure with all literal templates inline. From spec §13 + the plan.
- Part IV — failure modes and rollback. From spec §15.

Sections of the spec deliberately omitted from the runbook:
- §2.3 ("What this is not") — captured in Part I.2 in condensed form.
- §7.1–7.6 retrieval-primitives prose — relevant content is reproduced inside the vault `CLAUDE.md` template (Phase 5), where it actually needs to live for runtime use.
- §8 deep-research integration — relevant content is in Phase 4 and in the wiki-research playbook (Task 3.3).
- §9 wiki-research skill description — entire skill body is inlined at Task 3.3.
- §10 lint checklist — reproduced inside the vault `CLAUDE.md` template (Phase 5).
- §11 /recall skill description — entire skill body is inlined at Task 3.1.
- §12 instantiation playbook — superseded by Part III of this runbook.
- §13 literal templates — all reproduced inline in Phases 2, 3, 5.
- §14 domain-specific extensions — captured in Part II.7 and Phase 5.2.

The runbook has no unbound placeholders. `<vault-name>`, `<identity-statement>`, `<vault-root>` are bound at Phase 0 Task 0.1; `<new-type>` (if used) is bound at Phase 5.2 Step 3. All other angle-bracketed strings inside template bodies (e.g., `<Entity Name>`, `<the question, verbatim>`, `<topic>`, `<topic-slug>`) are intentional placeholder text inside the templates themselves — they get filled in by the user/agent when writing real wiki pages, not by the runbook executor.

A future Claude Code session given this file alone (no other context, no spec, no plan) can execute Phase 0 → Phase 7 end-to-end and produce a working vault.
