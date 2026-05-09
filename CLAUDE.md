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
