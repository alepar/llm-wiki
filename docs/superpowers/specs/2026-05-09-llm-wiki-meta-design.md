# LLM-Wiki Meta-Repo Design Spec

**Date:** 2026-05-09
**Status:** Approved (in brainstorming session)
**Next step:** Hand off to writing-plans skill for implementation plan.

---

## 1. Summary

This repository (`llm-wiki/`) is a meta-project that holds the canonical runbook for instantiating LLM-friendly markdown wikis (the "llm-wiki" pattern, originated by Karpathy). The runbook is a single self-sufficient file fed verbatim to a fresh Claude Code session along with a chosen domain — that session scaffolds a working vault.

This spec captures the design for evolving an existing 1600-line base runbook (at `/Users/alepar/AleCode/family/docs/plans/2026-05-08-llm-wiki-vault-instantiation.md`) by integrating prior-art research, kepano's Obsidian skills, official Obsidian CLI, three retrieval scopes, auto-commit policy, a session-start hook, an install-helper recipe, and a vendor-autoupdate policy.

## 2. Goals and non-goals

**Goals:**
- Produce a single consolidated runbook (`RUNBOOK.md`) that can be handed verbatim to a fresh Claude session and produce a working vault end-to-end.
- Integrate research-validated additions from prior art (Karpathy gist, kennyg gist, NiharShrotri/llm-wiki, ekadetov/llm-wiki, kepano/obsidian-skills) and from the Obsidian-CLI landscape.
- Establish an autoupdate philosophy for every vendored dependency in produced vaults.
- Document philosophy/sources/credits in `CLAUDE.md` (this meta-repo) and human handoff in `README.md`.

**Non-goals:**
- Actually instantiating a vault using the runbook (separate session, separate working dir).
- Building tooling beyond markdown + skills + qmd (no Python packages, no FastAPI UIs, no MCP servers of our own).
- Maintaining a copy of upstream submodule content — submodules stay submoduled.
- Backwards-compatibility shims with the existing `family/docs/plans/` runbook — that file becomes the historical base; this repo's `RUNBOOK.md` supersedes it.

## 3. Approach

**Faithful integration** (chosen over restructure or layered approaches):

Take the existing 1600-line runbook as the base. Insert new sections inline, modify existing sections where invariants change, preserve the four-part Part I / Part II / Part III / Part IV structure. Result: ~2200 lines, still a single self-sufficient file.

Rejected alternatives:
- **Restructure** (reorganize taxonomy): risks losing the carefully-crafted coherence of the working base.
- **Layered** (base + appendix): breaks the "give this single file to a fresh Claude session" property that's load-bearing for the use case.

## 4. Repo layout (the meta-repo)

```
llm-wiki/
├── README.md                # Human-facing: how to instantiate a wiki using RUNBOOK.md
├── CLAUDE.md                # Agent-facing: philosophy, sources, principles for editing RUNBOOK.md
├── RUNBOOK.md               # The consolidated, self-sufficient instantiation runbook
├── docs/
│   ├── prior-art.md         # Research notes: what we adopted/rejected from each source, why
│   └── superpowers/specs/
│       └── 2026-05-09-llm-wiki-meta-design.md   # This spec
└── .gitignore
```

**Notes:**
- `RUNBOOK.md` lives at root in caps to advertise its primacy alongside README.md and CLAUDE.md.
- `CLAUDE.md` here describes **the meta-repo**, not the produced vault. It captures: identity, philosophy, sources/credits, rules for editing RUNBOOK.md without breaking invariants.
- `docs/prior-art.md` is provenance — why each design decision was made, citing originating sources.

## 5. Document model (changes vs base runbook)

### 5.1 Three retrieval scopes

New section in vault `CLAUDE.md` (Part II.8 of the runbook). Three scopes for qmd queries against the vault:

| Scope | Indexes | When to use |
|---|---|---|
| `wiki` | curated typed pages (entity, concept, synthesis, plus domain-extended types) | Synthesizing answers; trust hierarchy lookups |
| `raw` | source documents under `raw/` | Citing primary sources; finding original quotes |
| `hybrid` | both `wiki` and `raw` | Default for general questions; fallback when scope is unclear |

Implementation: qmd path filters or per-scope queries. wiki-research playbook explicitly says which scope to use at each phase.

### 5.2 New folder: `raw/`

Vault root gains a `raw/` folder. Holds ingested clean source documents — defuddle output from web articles, manually placed papers/blog posts/repo docs.

**Properties:**
- Committed to git (vault stays portable).
- Indexed by qmd (gives the `raw` scope its content).
- Minimal frontmatter only. No `type:` field — `raw/` documents are not wiki pages:

  ```yaml
  source_url: https://...
  ingested_at: YYYY-MM-DD
  kind: article | paper | doc | repo | post
  ```

- Citable from synthesis pages as `qmd://<vault-name>/raw/<slug>.md`.

**Lint** excepts `raw/` from page-type rules (no `type:` requirement, no `date_updated:` requirement).

### 5.3 Synthesis citation forms

Synthesis `sources:` frontmatter accepts three citation forms (was two):

```yaml
sources:
  - qmd://<vault-name>/<path>            # vault page
  - qmd://<vault-name>/raw/<slug>.md     # ingested raw doc (NEW)
  - https://...                          # non-ingested external URL
```

### 5.4 Frontmatter additions (declined)

The following were considered and rejected:
- `source_count:` on entity/concept pages (kennyg). Adds maintenance burden without clear value.
- `log.md` append-only audit trail (Karpathy). Replaced by git commit history (which we get for free with auto-commit, see §7).

## 6. Procedure changes (vs base runbook)

### 6.1 Modified phases

| Phase | Change |
|---|---|
| 1 | Adds `raw/.gitkeep` + `raw/README.md` describing what belongs there. |
| 3 | Adds new committed skill `update-vendors` alongside `wiki-research` and `recall`. |
| 4 | Adds **second** submodule: `kepano/obsidian-skills` at `.claude/skills/obsidian-skills/`. |
| 5 | Vault `CLAUDE.md` template gains: three-retrieval-scopes section, auto-commit policy, vendor-autoupdate policy, lint pass using `obsidian unresolved`, references to defuddle and obsidian-markdown skills. |
| 6 | qmd install replaced by generic install-helper recipe (see §9). Phase reorders: 6.1 detect → 6.2 install-helper → 6.3 setup → 6.4 verify. |

### 6.2 New phases

| Phase | Purpose |
|---|---|
| 1.5 | Bootstrap `raw/` folder. |
| 4.5 | Obsidian CLI install (uses install-helper recipe). Scaffold SessionStart hook. |

### 6.3 Auto-commit shape (procedure-wide)

Every "agent writes a file" step gains a tail: agent runs `git add <files>` (chunk-staged via `git add -p` if other unrelated changes are present in the working tree) and commits with a topic-shaped message, then reports the commit SHA. One commit per approved-write turn — no batching across turns; multi-page writes approved in one turn become one commit.

## 7. Auto-commit policy

In vault `CLAUDE.md` and the wiki-research playbook:

1. Agent shows a **diff summary** before any write (file path, +/- line counts, key frontmatter fields touched, slug name for new pages). Full unified diff available on request — not shown by default.
2. User approves explicitly ("yes", "looks good", or directed edits).
3. Agent writes the file(s).
4. Agent runs `git add <files-just-written>` (chunk-staged with `git add -p` if other sessions' uncommitted hunks are present — concurrent-sessions discipline preserved).
5. Agent commits with message scoped to the topic of the write (e.g., `wiki: add synthesis "what is the best ratio for X"`).
6. Agent reports the commit SHA back to the user.

**Boundary:** one commit per approved-write turn. No batching across turns.

**Rationale:** aligns with the user's "git history replaces log.md" choice — every meaningful action becomes a commit; `git log` is the audit trail.

## 8. SessionStart hook

`.claude/settings.json` references `.claude/hooks/session-start.sh` (POSIX shell, committed to vault).

**Behavior, in order:**

1. Run `qmd status` for the vault's collection.
2. Branch:
   - **Fresh** → silent. Exit 0, no output.
   - **Stale** (index older than newest `.md` mtime, or new files since last index) → run `qmd update`. If many new files (>10% delta) or new files lack embeddings, also run `qmd embed`. Print one-line summary (`qmd: refreshed index, N files updated`).
   - **qmd not installed** → print one-line guidance pointing at `CLAUDE.md` retrieval-primitives section. Don't fail the session.
   - **No collection for this vault** → print one-line guidance pointing at Phase 6 setup. Don't fail.
3. **Vendor-update nudge** (see §10): if `.research/.deps-last-checked` is older than 7 days (or missing), append `Vendor deps last checked N days ago — run /update-vendors to review.` Never auto-runs.
4. Hook never blocks session start (always exits 0 even on errors). stderr cached to `.research/.session-start.log` for inspection.

**Constraints:**
- Cross-platform-safe (`/usr/bin/env bash`, no GNU-only flags).
- Read-only on the vault — never edits markdown.
- `qmd update` and `qmd embed` are the only "mutating" ops it runs; permitted under revised Invariant 4 (see §11) since they're idempotent index ops that don't change vault content.

## 9. Install-helper recipe (qmd, Obsidian CLI, future tools)

Generic recipe used in Phases 4.5 (Obsidian CLI) and 6.2 (qmd). Inline in the runbook (not a separate skill — YAGNI; factor into a skill only if a third tool is added).

**Inputs:** repo URL, expected binary name, vault context.

**Behavior:**

1. **Fetch latest install guidance.** Agent fetches `https://raw.githubusercontent.com/<owner>/<repo>/main/README.md` (falls back to releases page if README parsing fails).
2. **Parse install methods** from the README. Build a list of `{method, command, platform-applicability, notes}` tuples. Methods to look for: `brew`, `scoop`, `winget`, `apt`, `cargo install`, prebuilt-binary download (`curl ... | sh`), `bun install -g`, `npm install -g`, source build.
3. **Detect environment** (read-only):
   - OS: `uname -s` (Darwin / Linux / MINGW)
   - Arch: `uname -m`
   - Package managers present: `command -v brew`, `command -v scoop`, `command -v winget`, `command -v apt`, `command -v cargo`, `command -v bun`, `command -v npm`
4. **Rank candidates** by friction (lowest first):
   1. Platform-native package manager with autoupdate (brew / scoop / winget / apt)
   2. Prebuilt binary via release-page download
   3. Language toolchain manager (cargo / bun / npm) — if already present
   4. Source build — last resort
5. **Present recommendation** with top choice + up to 2 alternatives. Example output:
   ```
   Recommended (mac, brew installed): `brew install tobilu/qmd/qmd`
   Alternatives:
     - prebuilt binary: `curl -fsSL https://github.com/tobilu/qmd/releases/.../install.sh | sh`
     - bun: `bun install -g @tobilu/qmd`

   Run the recommended command? [agent-runs / I'll-run-it / show-other-options]
   ```
6. **Default branch:** print and wait for user to run. If user explicitly opts in ("you run it"), agent executes the recommended command (per-session opt-in; see Invariant 4).
7. **Verify post-install:** re-run version probe (`qmd status`, `obsidian --version`). If still failing, surface the upstream error verbatim and stop.

## 10. Vendor autoupdate policy

**Philosophy:** every vendored dependency offers autoupdate guidance. SHAs/versions stay pinned for reproducibility, but the agent surfaces "update available" signals and produces ready-to-run upgrade commands with change summaries. The user always pulls the trigger.

**Components covered (in produced vaults):**
- `qmd` (CLI tool)
- Obsidian CLI (CLI tool)
- `deep-research` (git submodule)
- `obsidian-skills` (git submodule)
- Any future vendored dep added to the runbook

**Mechanism: a committed `update-vendors` skill** at `.claude/skills/update-vendors/SKILL.md`. First-party, scaffolded by Phase 3 of the runbook. Invocable directly (`/update-vendors`) or referenced from the SessionStart hook's weekly nudge.

**Per-dep behavior:**
1. **Detect current version.**
   - Submodules: `git -C .claude/skills/<name> rev-parse HEAD` and `git -C ... describe --tags --always`.
   - CLI tools: `qmd --version`, `obsidian --version`.
2. **Detect latest available.**
   - Submodules: `git ls-remote <url> HEAD` and `git ls-remote --tags <url>`.
   - CLI tools: fetch upstream README/releases page; parse latest version (reuse install-helper parsing).
3. **Build change summary.**
   - Submodules: upstream commit log between pinned SHA and HEAD (`git log --oneline pinned_sha..HEAD`), capped at ~20 lines with notice if more.
   - CLI tools: release notes for new version (parse from releases page or CHANGELOG).
4. **Present per-dep verdict** with options: yes / show-full-diff / skip.
5. **Apply approved updates.**
   - Submodules: `git submodule update --remote <path>` then commit.
   - CLI tools: print/run install command via install-helper recipe.

**SessionStart integration:** weekly nudge in the hook (see §8), never blocking, never auto-running.

**Scope decision:** the `update-vendors` skill ships in produced vaults only, NOT in this meta-repo. The meta-repo has no submodules of its own; manual maintainer-driven bumps of pinned SHAs in `RUNBOOK.md` are sufficient.

## 11. Invariant changes (vs base runbook)

Numbering follows the base runbook's Part I.3 list.

| # | Original | Revised |
|---|---|---|
| 1 | Three core page types: entity, concept, synthesis | **Unchanged.** |
| 2 | Page type lives in frontmatter, never folder path | **Unchanged.** |
| 3 | Synthesis pages are write-once | **Unchanged.** |
| 4 | qmd mutating operations are never agent-run | **Revised.** Agent may automatically run `qmd update` and `qmd embed` (idempotent index ops that don't change vault content). Agent never auto-runs `qmd collection add` (initial setup) or `qmd ingest <url>` (pulls external content). Install is agent-guided, user-executed by default; user may opt the agent in per-session via the install-helper recipe. |
| 5 | Topic-first folders, never type-first | **Unchanged.** |
| 6 | `.research/` is exclusively for deep-research output | **Unchanged.** |
| 7 | External web sources are not copied into the vault | **Split.** (a) "External web sources are not copied into the vault as wiki pages" — still true. (b) NEW: "Sources may be ingested into `raw/` as cleaned-up markdown via the defuddle skill when long-term grounding is desirable. Synthesis pages still cite via `https://...` for non-ingested sources or `qmd://<vault-name>/raw/...` for ingested ones." |
| 8 | Concurrent-sessions discipline | **Unchanged.** |
| 9 | NEW | **Auto-commit follows user-approved writes.** Diff summary always shown before write. Once user approves, agent commits. One commit per approved-write turn. Full diffs available on request but not shown by default. |

## 12. Skills landscape (in produced vaults)

**First-party committed (scaffolded verbatim by runbook):**
- `wiki-research` — disciplined research loop. Default deep-research mode is **UltraDeep** (revised — was Standard); override only if user explicitly says "quick research" or "standard research".
- `recall` — raw qmd debug query wrapper.
- `update-vendors` — vendor-autoupdate skill (NEW).

**Submoduled (pinned per vault):**
- `deep-research` → `199-biotechnologies/claude-deep-research-skill` (existing).
- `obsidian-skills` → `kepano/obsidian-skills` (NEW). Provides nested sub-skills: `defuddle` (web→clean MD for `raw/` ingest), `obsidian-markdown` (wikilink/callout/property syntax reference), and others (`obsidian-bases`, `json-canvas`, `obsidian-cli`) which are not advertised but accessible.

**Sub-skill discovery path:** if Claude Code's skill discovery walks nested submodule folders, sub-skills are auto-discovered at `.claude/skills/obsidian-skills/<name>/SKILL.md`. If not, Phase 4 includes a fallback: scaffold thin wrapper SKILL.md files at `.claude/skills/<name>/SKILL.md` that delegate via path. Phase 4 verification step picks the working path at scaffold time.

**Obsidian CLI (installed user tool, not committed):** used for two narrow purposes:
1. `obsidian unresolved` in the lint pass — broken-wikilink detection. Replaces the home-grown lint pass 2.
2. `obsidian search` as fallback retrieval when qmd is unavailable (extends the fallback chain alongside grep).

NOT used for: writes (CLI's `create`/`daily` bypassed; agent writes files directly), search-as-primary (qmd wins), frontmatter manipulation (handled by skills + lint).

## 13. Build sequence

1. Write this spec → `docs/superpowers/specs/2026-05-09-llm-wiki-meta-design.md` (this file).
2. Self-review the spec inline.
3. User reviews the spec — gate before further work.
4. Hand off to writing-plans skill. The implementation plan covers:
   a. Write `docs/prior-art.md` from the research notes (Karpathy / kennyg / NiharShrotri / ekadetov / kepano + obsidian-cli landscape).
   b. Write `CLAUDE.md` (meta-repo identity, sources/credits, philosophy, runbook-editing rules).
   c. Write `README.md` (human handoff guide).
   d. Write `RUNBOOK.md` by integrating the base runbook + all design decisions in this spec.
   e. Sanity-check pass on `RUNBOOK.md`: every invariant change applied, every new phase inserted, every reference resolves, no orphaned `<placeholders>`.
   f. Commit(s) — topic-shaped: one for prior-art, one for CLAUDE.md+README, one for RUNBOOK.md.
5. Validation gates before declaring done:
   - `RUNBOOK.md` has zero unbound placeholders besides the three runtime-bound ones (`<vault-name>`, `<vault-root>`, `<identity-statement>`) and intentional template-internal placeholders (e.g., `<topic>`, `<Entity Name>`).
   - All nine invariants (eight revised + one new) restated coherently.
   - Submodule URLs in `RUNBOOK.md` are real and reachable.

## 14. Open questions / deferred items

These are intentional deferrals to implementation time, not unresolved design questions:

- **Submodule SHA pinning.** `RUNBOOK.md` pins to `main` initially with a note to bump to a tag/SHA on first instantiation. The `update-vendors` skill provides the bump mechanism going forward.
- **Nested-submodule skill discovery.** Whether Claude Code's skill discovery walks `.claude/skills/obsidian-skills/<name>/SKILL.md` paths is verified at scaffold time (Phase 4). Fallback (wrapper SKILL.md files) is in the runbook either way.
- **Actually instantiating a vault.** Out of scope for this work; happens in a separate session in another working dir.

## 15. Sources / credits

- **Andrej Karpathy** — original "llm-wiki" gist; three-folder split (raw / wiki / schema) and ingest/query/lint workflow framing.
- **kennyg** — second "llm-wiki" gist; concrete frontmatter conventions; `confidence:` on concepts.
- **NiharShrotri** ([github](https://github.com/NiharShrotri/llm-wiki)) — three retrieval scopes (wiki / raw / hybrid); `sources:` array as canonical provenance handle.
- **ekadetov** ([github](https://github.com/ekadetov/llm-wiki)) — Claude Code skill packaging; SessionStart hook bootstrap; auto-commit pattern.
- **Steph Ango / kepano** ([github](https://github.com/kepano/obsidian-skills)) — `defuddle` and `obsidian-markdown` skills; agent-skills format for Obsidian.
- **tobilu / qmd** — hybrid retrieval (BM25 + vector + LLM rerank).
- **199-biotechnologies / claude-deep-research-skill** — multi-source web research with citation tracking.
- **Obsidian (official CLI)** — `obsidian unresolved` for blessed wikilink resolution.
- **Base runbook** — `/Users/alepar/AleCode/family/docs/plans/2026-05-08-llm-wiki-vault-instantiation.md` (Alexey Parfenov, 2026-05-08); the foundation this spec evolves.
