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

This repo's runbook synthesizes those approaches with kepano's [obsidian-skills](https://github.com/kepano/obsidian-skills), the official [Obsidian CLI](https://obsidian.md/cli), and [tobi/qmd](https://github.com/tobi/qmd) for retrieval.

See [`docs/prior-art.md`](./docs/prior-art.md) for what was adopted from each source. See [`CLAUDE.md`](./CLAUDE.md) for design philosophy.
