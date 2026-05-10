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

## 7. tobi/qmd (search backend)

**URL:** https://github.com/tobi/qmd

**Summary.** Hybrid retrieval CLI + MCP — BM25 + vector embeddings + LLM rerank against local markdown collections. Single-file SQLite index per collection. Ships an official Claude Code plugin (`tobi/qmd`) that exposes the index over MCP.

**Adopted as the assumed retrieval backend.** Adoption preceded this survey — qmd is the load-bearing primitive that makes the LLM-wiki pattern work at scale (Karpathy's gist explicitly calls it out as the recommended retriever). The survey of alternatives (lunr, ripgrep, sqlite-fts, embedding-only stores) was not conducted because:

- qmd is the only retriever the surveyed authors converged on.
- Hybrid (BM25 + vector + rerank) outperforms any single-approach retriever for the curated-wiki + ingested-source mix this pattern produces.
- The official MCP plugin makes Claude Code integration trivial.

**Trade-offs noted:**
- One collection per vault — fine for this scope, doesn't cleanly support a multi-vault workspace if one ever materializes.
- Index lives in the vault repo at `<vault-root>/.qmd/index.sqlite` (gitignored) — must be rebuilt after a fresh clone. (Originally lived in `~/.cache/qmd/`; per-vault layout adopted post-`b775592` upstream fix that made `INDEX_PATH` env work for the qmd MCP server.)
- Mutating ops (`qmd ingest`, `qmd collection add`) are user-run only by policy (Invariant 4).
- Hard floor on qmd version: post-`b775592` (commit `b77559223025cbcff3f992df0bf01147497c3bab`, 2026-05-09). Older versions silently fall back to the global index for MCP queries, breaking per-vault isolation.

## 8. 199-biotechnologies/claude-deep-research-skill (deep-research submodule)

**URL:** https://github.com/199-biotechnologies/claude-deep-research-skill

**Summary.** Claude Code skill for multi-source web research with citation tracking. Supports Quick / Standard / Deep / UltraDeep modes; produces structured markdown reports with source-attributed claims.

**Adopted as the deep-research submodule** pinned at `.claude/skills/deep-research/`. Like qmd, adoption preceded the survey — the wiki-research workflow needs a thorough web-research primitive, and this skill is the most actively maintained option in May 2026. The wiki-research playbook calls into it whenever vault coverage is insufficient to answer a question (default mode: UltraDeep).

**Trade-offs noted:**
- API key requirements vary by deep-research mode — surfaced verbatim if missing.
- Output schema may evolve upstream — pinned by submodule SHA; bumped via `/update-vendors`.
- Only the markdown report is consumed by wiki-research; HTML/PDF artifacts are ignored.

## Cross-cutting takeaways

1. NiharShrotri's three retrieval scopes — adopted.
2. kepano's `obsidian-markdown` and `defuddle` skills — adopted as a submodule.
3. ekadetov's auto-commit pattern — adopted (after-user-approved-write only).
4. ekadetov's SessionStart hook — adopted (read-only freshness check + idempotent index update).
5. Official `obsidian unresolved` — adopted for lint.
6. Vendor autoupdate philosophy — first-party design, applied to qmd, obsidian-cli, deep-research, obsidian-skills.
7. Of five "llm-wiki" sources, exactly one (NiharShrotri) is implemented end-to-end and one (ekadetov) is packaged as a reusable artifact. The two gists are sketches.
