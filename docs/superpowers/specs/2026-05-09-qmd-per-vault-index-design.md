# Per-Vault qmd Index Design Spec

**Date:** 2026-05-09
**Status:** Approved (in brainstorming session)
**Next step:** Hand off to writing-plans skill for implementation plan.

---

## 1. Summary

Today the LLM-wiki runbook assumes every vault shares a single global qmd index at `~/.cache/qmd/index.sqlite`. Multiple vaults coexist as named collections inside that one store. With upstream commit `b77559223025cbcff3f992df0bf01147497c3bab` ("fix mcp --index store selection"), qmd's MCP server now correctly honors index-selection inputs — making true per-vault indexes practical.

This spec evolves the runbook so each vault gets its own index file at `<vault-root>/.qmd/index.sqlite`, configured via committed files at the vault root. Vaults are now fully isolated: changes in one don't pollute another's index, and a vault repo carries its own qmd configuration.

## 2. Goals and non-goals

**Goals:**
- Each vault has a dedicated qmd index file at `<vault-root>/.qmd/index.sqlite`.
- Per-vault qmd config (collection name, ignore patterns) lives in a committed file.
- The qmd MCP server (launched by the marketplace `tobi/qmd` plugin or any equivalent) automatically uses the per-vault index when a session opens in the vault.
- CLI invocations from the user, hooks, and scripts use the per-vault index without needing to remember a flag every time.
- Older qmd installs are detected and the user is routed to upgrade.

**Non-goals:**
- Removing the marketplace plugin. It stays installed.
- Replacing qmd with another retriever.
- Shared-cache optimizations (de-duping embeddings across vaults — out of scope).
- Auto-rebuilding the index after a fresh `git clone` of the vault. The user runs the setup commands once on first clone; the SessionStart hook keeps things fresh thereafter.

## 3. Approach

**`.mcp.json` + qmd-native config** (chosen over hook-based env injection and over XDG-cache redirection):

The vault's `.mcp.json` declares a per-server `env:` block with `INDEX_PATH` and `QMD_CONFIG_DIR` set via `${CLAUDE_PROJECT_DIR}` expansion. This shadows the marketplace plugin's qmd MCP server entry per Claude Code's scope precedence (Project > User > Plugin). qmd reads the env vars directly (verified in `tobi/qmd src/store.ts`).

Rejected alternatives:

- **SessionStart hook + env exports.** Verified non-functional: SessionStart fires before MCP servers connect, and `CLAUDE_ENV_FILE` only propagates to Bash tool, not MCP child processes.
- **`XDG_CACHE_HOME` redirect.** Works for index isolation but coarsely relocates the entire qmd cache. Less explicit than `INDEX_PATH`.
- **Per-vault `qmd.yml` at vault root with auto-discovery.** qmd does not auto-discover config at vault root or cwd. Config files only load via `--config <path>` or via `${QMD_CONFIG_DIR}/<indexName>.yml`.

## 4. The three vault-root files

### 4.1 `.mcp.json` (committed)

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

`${VAR:-default}` is qmd's documented expansion form. The `:-.` defensive default keeps the config parseable when run outside Claude Code (e.g., `qmd mcp` invoked directly for debugging).

### 4.2 `.qmd/index.yml` (committed)

qmd auto-loads `${QMD_CONFIG_DIR}/<indexName>.yml` where `<indexName>` defaults to `index`. Filename therefore is `index.yml`:

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

`<vault-name>` is the kebab-case vault identity captured at Phase 0.

### 4.3 `.gitignore` additions

```
# Per-vault qmd index (regenerable from the wiki content)
/.qmd/index.sqlite
/.qmd/*.sqlite-wal
/.qmd/*.sqlite-shm
```

Everything else under `.qmd/` (the YAML, plus any future state qmd writes there) stays committed. The `.qmd/` directory itself is **not** gitignored.

## 5. CLI invocations

Manual qmd commands and the SessionStart hook use env-prefix form:

```bash
INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd qmd <subcommand> [args]
```

Examples:
- `INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd qmd collection add . --name <vault-name> --mask '**/*.md'`
- `INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd qmd embed`
- `INDEX_PATH=$(pwd)/.qmd/index.sqlite QMD_CONFIG_DIR=$(pwd)/.qmd qmd status`

The vault `CLAUDE.md` retrieval-primitives section documents an optional shell helper users may add to their rc:

```bash
qmd-vault() { INDEX_PATH="$PWD/.qmd/index.sqlite" QMD_CONFIG_DIR="$PWD/.qmd" qmd "$@"; }
```

…dropping the env prefix to `qmd-vault status`, etc. The env-prefixed form remains canonical in the runbook.

## 6. SessionStart hook changes

`.claude/hooks/session-start.sh` exports the env vars at the top:

```bash
export INDEX_PATH="$PWD/.qmd/index.sqlite"
export QMD_CONFIG_DIR="$PWD/.qmd"
```

(`$PWD` resolves to the vault root because hooks fire there.)

All subsequent `qmd status`/`qmd update`/`qmd embed` calls in the hook inherit the env. No other hook logic changes.

## 7. Marketplace plugin interaction

The user keeps `tobi/qmd` plugin installed. The vault's project-scoped `.mcp.json` shadows the plugin's qmd server entry by name, per Claude Code's documented scope precedence (Project > User > Plugin).

**Side effects:**
- Other plugin features (commands, hooks shipped by the plugin) remain globally available.
- First-time session in a vault triggers Claude Code's "approve project MCP config" prompt. Documented in README and Phase 7 hand-off summary.

## 8. Version floor

`b77559223025cbcff3f992df0bf01147497c3bab` (referred to as `b775592` in this spec) is the hard floor. Older qmd ignores `INDEX_PATH` in MCP and silently falls back to the global index — vault isolation breaks without user-visible signal.

**Detection** is added to Phase 6.1: after `qmd status` succeeds, parse `qmd --version` against a known floor. If older, route to install-helper (Phase 6.2) for upgrade. Until qmd has a release tag containing the fix, the floor is enforced via build-date check (commit `b775592` was 2026-05-09).

**`update-vendors` skill** gains a per-vendor "minimum version" concept. The skill's vendor table for qmd records the floor; when the user runs `/update-vendors`, qmd's current version is compared against the floor and an upgrade is flagged as required (not optional) if below.

## 9. Verification

Phase 7 Task 7.2 smoke-test gains a step: "ask the agent in a fresh Claude Code session to call qmd MCP `status` and report the index path; expect `<vault-root>/.qmd/index.sqlite`." Confirms env propagation end-to-end.

Phase 6.4 success criteria add:
- `<vault-root>/.qmd/index.sqlite` exists and is non-empty after `qmd embed`.
- `<vault-root>/.qmd/index.yml` exists and is committed.
- `.mcp.json` exists and is committed; `${CLAUDE_PROJECT_DIR:-.}/.qmd/index.sqlite` resolves correctly.

## 10. Build sequence (RUNBOOK.md changes)

Modifications, all in `RUNBOOK.md`:

1. **Part I.1 (the shape)** — sidecar paragraph mentions per-vault index at `.qmd/index.sqlite`; layout diagram cell labels `.qmd/`.
2. **Part I.3 Invariant 4** — clarifying sentence: per-vault index is bound via `INDEX_PATH` env in `.mcp.json`; agent never edits `.mcp.json` mid-session.
3. **Phase 1.3 README template** — Layout section gains `.qmd/` line; mentions per-vault index briefly.
4. **Phase 1 `.gitignore` template** — adds the three sqlite-related ignores.
5. **New Phase 1.7** — scaffold `.qmd/index.yml` and `.mcp.json` with verbatim content from Sections 4.1–4.2 (substituting `<vault-name>`). Old Task 1.7 (commit) becomes 1.8.
6. **Phase 4.5.2 SessionStart hook script** — top of script gains the two `export` lines from Section 6.
7. **Phase 5 vault `CLAUDE.md` template** — Retrieval primitives: rewrite "Setup commands" + "Update commands" with env-prefix form; add a short "Per-vault index" subsection explaining the file layout (`.mcp.json`, `.qmd/index.yml`, `.qmd/index.sqlite`).
8. **Phase 6 (qmd setup)** — Section 2 of this spec specifies the changes: 6.1 gains version-floor check; 6.3 uses env-prefixed form; 6.4 adds the three new success-criterion checks.
9. **Phase 7 Task 7.2 smoke test** — adds the MCP env-propagation check from Section 9.
10. **Part IV failure modes table** — three new rows:
    - "qmd < b775592 silently uses global index" → recovery: bump via install-helper or `/update-vendors`.
    - "First-time MCP approval prompt confuses user" → recovery: documented in README and hand-off.
    - "`${CLAUDE_PROJECT_DIR}` not expanded (older Claude Code)" → recovery: substitute absolute path; surface the Claude Code version requirement.
11. **`update-vendors` SKILL.md** (inside RUNBOOK.md Phase 3 Task 3.4 content) — adds version-floor concept; flags qmd < b775592 as required upgrade.

**Files in this repo to update:**
- `RUNBOOK.md` — all the above, ~11 Edit operations.
- `docs/prior-art.md` — section 7 (qmd) gains a note about per-vault indexes being possible post-`b775592`.
- `README.md` — "What you get" bullet list adds "per-vault qmd index (no cross-vault contamination)".
- `CLAUDE.md` — no change.

## 11. Open questions / deferred

- **Claude Code version floor for `${VAR}` expansion.** The expansion is documented but there's no documented introduction date. If a user has very old Claude Code, they may see literal `${CLAUDE_PROJECT_DIR}` strings as paths and qmd will fail to find the index file. Phase 6.4 verification catches this (the index won't be at the expected path). Mitigation: documented as a failure mode (#10).
- **Index name vs collection name.** The qmd config file is `<indexName>.yml` (default `index`). The `collections:` block names a collection `<vault-name>`. These are deliberately different concepts: the index is the SQLite file; the collection is the named scope inside it. The runbook documents this distinction so users don't conflate them.
- **Multi-collection-per-vault.** Some users may want multiple collections within one vault (e.g., `wiki` vs `raw` as separate collections instead of path-filtered scopes). Out of scope for this spec — `.qmd/index.yml` is structured to allow adding collection entries later without re-scaffolding.

## 12. Sources

- **tobi/qmd commit `b77559223025cbcff3f992df0bf01147497c3bab`** — fix mcp --index store selection. Enables MCP env-based index selection.
- **tobi/qmd `src/store.ts` `getDefaultDbPath`** — confirms `INDEX_PATH` and `XDG_CACHE_HOME` env vars.
- **tobi/qmd `src/collections.ts` `getConfigDir`/`getConfigFilePath`** — confirms `QMD_CONFIG_DIR` env var and `${QMD_CONFIG_DIR}/<indexName>.yml` lookup.
- **Claude Code MCP docs** (https://code.claude.com/docs/en/mcp) — confirms `${VAR}` and `${VAR:-default}` expansion in `.mcp.json` `command`, `args`, `env`, `url`, `headers`. Confirms scope precedence (Project > User > Plugin).
- **Claude Code Hooks docs** (https://code.claude.com/docs/en/hooks) — confirms SessionStart fires before MCP connect and `CLAUDE_ENV_FILE` is Bash-tool-scoped.
- **Wide real-world `${CLAUDE_PROJECT_DIR}` usage** in `.mcp.json` env blocks across Claude Code projects on GitHub (FlorianBruniaux/ccboard, schlessera/ppt-slide-agent, luminarylane/fal-mcp-server, etc.).
