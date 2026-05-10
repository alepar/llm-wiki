# Per-Vault qmd Index Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update `RUNBOOK.md` so that every produced vault gets its own qmd index at `<vault-root>/.qmd/index.sqlite`, configured via committed `.mcp.json` and `.qmd/index.yml` at the vault root. Plus small companion edits to `docs/prior-art.md` and `README.md`.

**Architecture:** Series of targeted Edit operations on `RUNBOOK.md` (~11 edits across Parts I, III, IV) plus 2 small edits to other files. Tasks 1–7 modify files without committing; Task 8 lands a single commit covering all changes (per spec §10).

**Tech Stack:** Markdown only. No code, no tests.

**Working directory:** `/Users/alepar/AleCode/llm-wiki/`

**Reference files:**
- Spec: `/Users/alepar/AleCode/llm-wiki/docs/superpowers/specs/2026-05-09-qmd-per-vault-index-design.md`
- Target: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md` (2096 lines, current)

**Commit policy:** All file modifications across Tasks 1–7 are uncommitted intermediate state. Task 8 makes one commit with the message specified in its step. Do not run `git add` or `git commit` in Tasks 1–7.

---

## Task 1: Update Part I (architecture diagram + sidecar paragraph + Invariant 4)

**Files:**
- Modify: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md`

Updates the Layer-3 cell of the architecture diagram, the sidecar paragraph after the diagram, and adds a clarifying sentence to Invariant 4.

- [ ] **Step 1: Update the Layer-3 architecture cell**

Find this exact text in RUNBOOK.md:

```
│ Layer 3: qmd (CLI + MCP plugin, stdio transport)           │
│   One collection: the vault. Index at                      │
│   ~/.cache/qmd/index.sqlite.                               │
└────────────────────────────────────────────────────────────┘
```

Replace with:

```
│ Layer 3: qmd (CLI + MCP plugin, stdio transport)           │
│   One collection per vault. Per-vault index at             │
│   <vault-root>/.qmd/index.sqlite (driven by .mcp.json      │
│   + .qmd/index.yml committed at the vault root).           │
└────────────────────────────────────────────────────────────┘
```

- [ ] **Step 2: Add a per-vault-index sentence to the sidecar paragraph**

Find this exact text:

```
Source documents ingested for long-term grounding land in a vault-root `raw/` folder, indexed by qmd alongside the wiki content. The vault therefore has three retrieval scopes: `wiki` (curated typed pages), `raw` (source documents), and `hybrid` (both).
```

Replace with:

```
Source documents ingested for long-term grounding land in a vault-root `raw/` folder, indexed by qmd alongside the wiki content. The vault therefore has three retrieval scopes: `wiki` (curated typed pages), `raw` (source documents), and `hybrid` (both).

The qmd index is per-vault: SQLite at `<vault-root>/.qmd/index.sqlite` (gitignored), wired via committed `.mcp.json` (sets `INDEX_PATH` and `QMD_CONFIG_DIR` env vars for the qmd MCP server) and `.qmd/index.yml` (qmd's per-collection config). No cross-vault index contamination.
```

- [ ] **Step 3: Append clarifying sentence to Invariant 4**

Find this exact text:

```
4. **Vault-content-changing qmd operations are never agent-run.** `qmd collection add` (initial setup) and `qmd ingest <url>` (pulls external content) are user-run only; agent prints the commands and waits. Index-only operations `qmd update` and `qmd embed` are idempotent and don't change vault content — the agent runs these automatically (typically from the SessionStart hook). qmd install is agent-guided via the install-helper recipe (Phase 6) and user-executed by default; the user may opt the agent in to executing the install per session.
```

Replace with:

```
4. **Vault-content-changing qmd operations are never agent-run.** `qmd collection add` (initial setup) and `qmd ingest <url>` (pulls external content) are user-run only; agent prints the commands and waits. Index-only operations `qmd update` and `qmd embed` are idempotent and don't change vault content — the agent runs these automatically (typically from the SessionStart hook). qmd install is agent-guided via the install-helper recipe (Phase 6) and user-executed by default; the user may opt the agent in to executing the install per session. The vault binds qmd to its own per-vault index via `INDEX_PATH` / `QMD_CONFIG_DIR` env vars in `.mcp.json` (committed); the agent never edits `.mcp.json` mid-session.
```

- [ ] **Step 4: Validate Part I changes**

```bash
cd /Users/alepar/AleCode/llm-wiki
grep -n 'Per-vault index at' RUNBOOK.md         # expected: 1 match
grep -n 'INDEX_PATH' RUNBOOK.md                  # expected: at least 1 match
grep -n '~/.cache/qmd/index.sqlite' RUNBOOK.md   # expected: 0 (we replaced the only one in Layer 3)
```

(No commit. Tasks 1–7 are uncommitted; Task 8 commits.)

---

## Task 2: Update Phase 1 (.gitignore + README template + new scaffolding task for .mcp.json and .qmd/index.yml)

**Files:**
- Modify: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md`

Adds sqlite ignores to the `.gitignore` template, adds `.qmd/` to the README Layout section, inserts a new Task 1.7 scaffolding `.mcp.json` and `.qmd/index.yml`, and renumbers the commit task to 1.8.

- [ ] **Step 1: Update `.gitignore` template (Phase 1 Task 1.2 Step 1)**

Find this exact text:

```
Write `<vault-root>/.gitignore` with verbatim content:

```
# OS / editor state
.DS_Store
*.swp
.idea/
.vscode/

# Optional Obsidian local state — comment out if the user shares it across machines
.obsidian/workspace*.json
```
```

Replace with:

```
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
```

- [ ] **Step 2: Update README template Layout section to mention `.qmd/`**

Find this exact text:

```
- `.claude/skills/` — first-party skills (`wiki-research`, `recall`, `update-vendors`) and the pinned submodules (`deep-research`, `obsidian-skills`).
- `.claude/hooks/` — SessionStart hook for qmd freshness checks.
- `raw/` — ingested source documents (web articles, papers); indexed by qmd as the `raw` retrieval scope.
- `.research/` — ephemeral deep-research artifacts (kept for traceability, not indexed).
- `<topic>/<subtopic>/` — actual wiki pages, organized by domain.
```

Replace with:

```
- `.claude/skills/` — first-party skills (`wiki-research`, `recall`, `update-vendors`) and the pinned submodules (`deep-research`, `obsidian-skills`).
- `.claude/hooks/` — SessionStart hook for qmd freshness checks.
- `.qmd/` — per-vault qmd config (`index.yml`) and SQLite index (`index.sqlite`, gitignored).
- `.mcp.json` — Claude Code MCP config; binds the qmd MCP server to this vault's index.
- `raw/` — ingested source documents (web articles, papers); indexed by qmd as the `raw` retrieval scope.
- `.research/` — ephemeral deep-research artifacts (kept for traceability, not indexed).
- `<topic>/<subtopic>/` — actual wiki pages, organized by domain.
```

- [ ] **Step 3: Insert new Task 1.7 (Scaffold per-vault qmd config) before existing Task 1.6, renumber the commit task**

Find this exact text:

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

Replace with:

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
```

- [ ] **Step 4: Validate Phase 1 changes**

```bash
cd /Users/alepar/AleCode/llm-wiki
grep -n 'Per-vault qmd index (regenerable' RUNBOOK.md      # expected: at least 1 (in .gitignore template)
grep -n '`.mcp.json` — Claude Code MCP config' RUNBOOK.md  # expected: 1
grep -n '### Task 1.6 — Scaffold per-vault qmd config' RUNBOOK.md  # expected: 1
grep -n '### Task 1.7 — First commit' RUNBOOK.md           # expected: 1
grep -c '### Task 1\.6 — Create' RUNBOOK.md                # expected: 0 (old 1.6 was renamed)
grep -n 'gitignore, README, .research, raw, qmd config' RUNBOOK.md  # expected: 1
```

(No commit.)

---

## Task 3: Update Phase 4.5 SessionStart hook (export INDEX_PATH and QMD_CONFIG_DIR)

**Files:**
- Modify: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md`

Adds two `export` lines at the top of the session-start.sh script content so all subsequent `qmd` invocations in the hook bind to the per-vault index.

- [ ] **Step 1: Add export lines to session-start.sh template**

Find this exact text:

```
set -u
LOG=".research/.session-start.log"
mkdir -p "$(dirname "$LOG")" 2>/dev/null

log_err() { printf '%s\n' "$*" >> "$LOG" 2>/dev/null; }

# 1. qmd freshness check
```

Replace with:

```
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
```

- [ ] **Step 2: Validate Phase 4.5 changes**

```bash
cd /Users/alepar/AleCode/llm-wiki
grep -n 'export INDEX_PATH="\$PWD/.qmd/index.sqlite"' RUNBOOK.md   # expected: 1
grep -n 'export QMD_CONFIG_DIR="\$PWD/.qmd"' RUNBOOK.md            # expected: 1
```

(No commit.)

---

## Task 4: Update Phase 5 vault `CLAUDE.md` template — Retrieval primitives section

**Files:**
- Modify: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md`

Rewrites the Setup commands and Update commands blocks to use the env-prefixed form, adds a new "Per-vault index" subsection, and updates the agent-automation policy to mention the env vars.

- [ ] **Step 1: Replace the Setup commands and Update commands blocks**

Find this exact text:

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
```

Replace with:

```
**Per-vault index (this vault's qmd binding):**

The qmd MCP server in this vault is bound to a per-vault index by `.mcp.json` at the vault root, which sets two env vars when the MCP server spawns:

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
```

- [ ] **Step 2: Update the agent-automation policy to mention env-binding**

Find this exact text:

```
**Agent automation policy:**

- **Always agent-callable:** `qmd query`, `qmd search`, `qmd get`, `qmd multi-get`, `qmd status`, `qmd update`, `qmd embed` (the last two are idempotent index ops; the SessionStart hook runs them automatically when stale).
- **Agent-callable on per-session opt-in:** qmd install (via install-helper recipe).
- **Never agent-run:** `qmd collection add`, `qmd ingest <url>` (vault-scope-changing).
```

Replace with:

```
**Agent automation policy:**

- **Always agent-callable** (with `INDEX_PATH` / `QMD_CONFIG_DIR` env from `.mcp.json` for MCP, or from the SessionStart hook for CLI): `qmd query`, `qmd search`, `qmd get`, `qmd multi-get`, `qmd status`, `qmd update`, `qmd embed` (the last two are idempotent index ops; the SessionStart hook runs them automatically when stale).
- **Agent-callable on per-session opt-in:** qmd install (via install-helper recipe).
- **Never agent-run:** `qmd collection add`, `qmd ingest <url>` (vault-scope-changing). The agent also never edits `.mcp.json` mid-session — that file is the per-vault index binding and is committed.
```

- [ ] **Step 3: Validate Phase 5 changes**

```bash
cd /Users/alepar/AleCode/llm-wiki
grep -n 'Per-vault index (this vault' RUNBOOK.md            # expected: 1
grep -n 'qmd-vault()' RUNBOOK.md                            # expected: 1
grep -c 'INDEX_PATH=\$(pwd)/.qmd/index.sqlite' RUNBOOK.md   # expected: at least 6 (in setup + update blocks)
grep -n "agent also never edits .mcp.json" RUNBOOK.md       # expected: 1
```

(No commit.)

---

## Task 5: Update Phase 6 (version-floor check + env-prefixed setup commands + new success criteria) and Phase 7 (smoke test)

**Files:**
- Modify: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md`

Adds a version-floor check to Phase 6.1, replaces the Phase 6.3 setup-commands block with env-prefixed form, adds new success criteria to Phase 6.4, and adds an MCP env-propagation smoke test step to Phase 7.

- [ ] **Step 1: Add version-floor check to Phase 6.1**

Find this exact text:

```
### Task 6.1 — Detect qmd state

- [ ] **Step 1: Run the freshness check (read-only — agent runs this)**

```bash
qmd status 2>&1
```

Branch on the result:

- **Case A — `qmd: command not found`** → continue to Task 6.2 (install).
- **Case B — `qmd status` runs but reports no collection matching `<vault-name>`** → skip to Task 6.3 (setup).
- **Case C — `qmd status` reports a collection matching `<vault-name>`** → unexpected for a fresh repo. Surface to the user: "A qmd collection named `<vault-name>` already exists from a previous setup. Skip qmd setup and proceed to Task 6.4 verification?" Wait for explicit confirmation.
```

Replace with:

```
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
```

- [ ] **Step 2: Replace Phase 6.3 setup-commands block with env-prefixed form**

Find this exact text:

```
### Task 6.3 — Print setup commands

- [ ] **Step 1: Print verbatim** (with `<vault-root>`, `<vault-name>`, `<identity-statement>` substituted):

> Run these in `<vault-root>`:
>
> ```bash
> cd <vault-root>
>
> # Add the vault as a qmd collection
> qmd collection add . --name <vault-name> --mask '**/*.md'
>
> # Initial context entries (root + collection-scoped, both with the identity statement)
> qmd context add /                  "<identity-statement>"
> qmd context add qmd://<vault-name> "<identity-statement>"
>
> # Embed (downloads ~2.1GB GGUF models on first install; subsequent runs are fast)
> qmd embed
> ```
```

Replace with:

```
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
```

- [ ] **Step 3: Update Phase 6.3 verification (Step 2)**

Find this exact text:

```
- [ ] **Step 2: Wait for confirmation, then verify**

After the user confirms:

```bash
qmd status 2>&1
```

Verify:
- A collection named `<vault-name>` is listed.
- The collection's source path matches `<vault-root>`.
- The indexed-file count is plausible for a fresh vault (~6–12 files: CLAUDE.md, README.md, three `.templates/*.md`, `raw/README.md`, plus the `wiki-research/playbook.md` and the three first-party SKILL.md files — depending on whether the dotfolder mask traversal includes them).
```

Replace with:

```
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
```

- [ ] **Step 4: Add new Phase 6.4 success criteria for per-vault binding**

Find this exact text:

```
### Task 6.4 — Verify the four success criteria

- [ ] **Step 1: `qmd status` returns the collection**

```bash
qmd status 2>&1 | grep -i '<vault-name>'
```

Expected: a non-empty match line.

- [ ] **Step 2: A query against the empty(-ish) vault returns without erroring**

```bash
qmd query --collection <vault-name> --json "test" 2>&1
```

Expected: valid JSON, no traceback, no `Error:` prefix on stderr. The result list may be empty or contain the framework files — both are fine.
```

Replace with:

```
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
```

- [ ] **Step 5: Add MCP env-propagation smoke test to Phase 7 Task 7.2**

Find this exact text:

```
- Phase 2 runs the qmd query at the `wiki` scope, gets ~0 results (the vault has no domain content yet), routes to Phase 4.
```

Replace with:

```
- Phase 2 runs the qmd query at the `wiki` scope, gets ~0 results (the vault has no domain content yet), routes to Phase 4.

Additionally, before invoking wiki-research, verify the MCP server is bound to the per-vault index:

> Ask the agent in the fresh Claude Code session: "Call the qmd MCP `status` tool and report the index path it shows."
>
> Expected: the agent reports an index path of the form `<vault-root>/.qmd/index.sqlite`. If the path is `~/.cache/qmd/index.sqlite` (or any non-vault path), the `${CLAUDE_PROJECT_DIR}` expansion in `.mcp.json` failed to resolve — surface to the user, fall back to writing absolute paths into `.mcp.json` (substituting `<vault-root>` directly), and re-run.
```

- [ ] **Step 6: Validate Phase 6 + 7 changes**

```bash
cd /Users/alepar/AleCode/llm-wiki
grep -n 'Step 2: Verify qmd version floor' RUNBOOK.md           # expected: 1
grep -n 'b77559223025cbcff3f992df0bf01147497c3bab' RUNBOOK.md   # expected: at least 1
grep -n 'Per-vault index file exists' RUNBOOK.md                # expected: 1
grep -n 'per-vault index file OK' RUNBOOK.md                    # expected: 1
grep -n 'Call the qmd MCP `status` tool' RUNBOOK.md             # expected: 1
grep -c 'INDEX_PATH=\$(pwd)/.qmd/index.sqlite' RUNBOOK.md       # expected: at least 12 (Phase 5 + Phase 6 invocations)
```

(No commit.)

---

## Task 6: Update Part IV failure modes + `update-vendors` SKILL.md (version-floor concept)

**Files:**
- Modify: `/Users/alepar/AleCode/llm-wiki/RUNBOOK.md`

Adds three new failure-mode rows and a version-floor section to the `update-vendors` SKILL.md template.

- [ ] **Step 1: Add three new rows to the Part IV failure modes table**

Find this exact text (the row in Part IV that mentions "qmd MCP crash mid-session"):

```
| qmd MCP crash mid-session | post-Phase 7 | One-shot retry; on second failure, fall back to Read+Grep for remainder of session; surface "qmd MCP unavailable, on grep fallback" once. |
```

Replace with:

```
| qmd MCP crash mid-session | post-Phase 7 | One-shot retry; on second failure, fall back to Read+Grep for remainder of session; surface "qmd MCP unavailable, on grep fallback" once. |
| qmd < `b775592` silently uses global index | 6.1 | The version-floor check in 6.1 catches this; if missed, the Phase 7 MCP smoke test catches it via wrong index path. Recovery: bump qmd via install-helper (Task 6.2) or `/update-vendors`. |
| First-time MCP approval prompt confuses user on fresh session | 7.x | Claude Code prompts to approve the project's `.mcp.json`. Documented behavior. Tell user to approve; the prompt happens once per project per machine. |
| `${CLAUDE_PROJECT_DIR}` not expanded by older Claude Code | 7.x (smoke test) | Phase 7 smoke test catches this — the qmd MCP `status` tool reports the literal `${CLAUDE_PROJECT_DIR}` string (or `./.qmd/index.sqlite` after `:-` fallback) as the index path instead of the absolute vault path. Recovery: substitute `<vault-root>` directly into `.mcp.json` at scaffold time and re-commit. Surface the Claude Code version requirement (`${VAR}` expansion in `.mcp.json` env was added with `code.claude.com/docs/en/mcp` doc'd support). |
```

- [ ] **Step 2: Add version-floor concept to the `update-vendors` SKILL.md template**

Find this exact text inside the update-vendors SKILL.md content:

```
## Vendors covered

- **qmd** (CLI tool) — installed via package manager.
- **Obsidian CLI** (CLI tool) — installed via package manager.
- **deep-research** (git submodule at `.claude/skills/deep-research/`).
- **obsidian-skills** (git submodule at `.claude/skills/obsidian-skills/`).
```

Replace with:

```
## Vendors covered

- **qmd** (CLI tool) — installed via package manager. **Minimum version:** post-`b775592` (`b77559223025cbcff3f992df0bf01147497c3bab`, 2026-05-09 — fixes MCP `--index` plumbing). Older versions silently break per-vault index isolation.
- **Obsidian CLI** (CLI tool) — installed via package manager.
- **deep-research** (git submodule at `.claude/skills/deep-research/`).
- **obsidian-skills** (git submodule at `.claude/skills/obsidian-skills/`).

**Minimum-version policy:** for any vendor with a documented floor (currently only qmd), the skill's per-dep verdict in Phase 4 marks an upgrade as **REQUIRED** (not optional) when the installed version is below the floor. The user can still skip, but the framework flags this as a known-broken state.
```

- [ ] **Step 3: Update update-vendors Phase 4 verdict block to mark version-floor failures**

Find this exact text inside update-vendors SKILL.md:

```
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
```

Replace with:

```
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
```

- [ ] **Step 4: Validate Part IV + update-vendors changes**

```bash
cd /Users/alepar/AleCode/llm-wiki
grep -n 'silently uses global index' RUNBOOK.md          # expected: 1 (in failure modes table)
grep -n 'First-time MCP approval prompt' RUNBOOK.md      # expected: 1
grep -n '${CLAUDE_PROJECT_DIR}' RUNBOOK.md               # expected: at least 4 (.mcp.json + failure modes + spec ref)
grep -n 'Minimum version' RUNBOOK.md                     # expected: 1 (in update-vendors template)
grep -n 'BELOW MINIMUM FLOOR' RUNBOOK.md                 # expected: 1
```

(No commit.)

---

## Task 7: Update `docs/prior-art.md` and `README.md`

**Files:**
- Modify: `/Users/alepar/AleCode/llm-wiki/docs/prior-art.md`
- Modify: `/Users/alepar/AleCode/llm-wiki/README.md`

Adds a note to the qmd section of prior-art and a bullet to README's "What you get".

- [ ] **Step 1: Add per-vault note to prior-art.md section 7 (qmd)**

Find this exact text in `/Users/alepar/AleCode/llm-wiki/docs/prior-art.md`:

```
**Trade-offs noted:**
- One collection per vault — fine for this scope, doesn't cleanly support a multi-vault workspace if one ever materializes.
- Index lives in `~/.cache/qmd/`, not in the vault repo — must be rebuilt after a fresh clone.
- Mutating ops (`qmd ingest`, `qmd collection add`) are user-run only by policy (Invariant 4).
```

Replace with:

```
**Trade-offs noted:**
- One collection per vault — fine for this scope, doesn't cleanly support a multi-vault workspace if one ever materializes.
- Index lives in the vault repo at `<vault-root>/.qmd/index.sqlite` (gitignored) — must be rebuilt after a fresh clone. (Originally lived in `~/.cache/qmd/`; per-vault layout adopted post-`b775592` upstream fix that made `INDEX_PATH` env work for the qmd MCP server.)
- Mutating ops (`qmd ingest`, `qmd collection add`) are user-run only by policy (Invariant 4).
- Hard floor on qmd version: post-`b775592` (commit `b77559223025cbcff3f992df0bf01147497c3bab`, 2026-05-09). Older versions silently fall back to the global index for MCP queries, breaking per-vault isolation.
```

- [ ] **Step 2: Add a bullet to README.md "What you get"**

Find this exact text in `/Users/alepar/AleCode/llm-wiki/README.md`:

```
- **qmd hybrid retrieval** — BM25 + vector + LLM rerank — with three scopes (wiki / raw / hybrid).
```

Replace with:

```
- **qmd hybrid retrieval** — BM25 + vector + LLM rerank — with three scopes (wiki / raw / hybrid). Per-vault index at `.qmd/index.sqlite` (no cross-vault contamination).
```

- [ ] **Step 3: Validate**

```bash
cd /Users/alepar/AleCode/llm-wiki
grep -n 'b77559223025cbcff3f992df0bf01147497c3bab' docs/prior-art.md  # expected: 1
grep -n 'Per-vault index at' README.md                                 # expected: 1
grep -n 'Index lives in the vault repo' docs/prior-art.md              # expected: 1
```

(No commit.)

---

## Task 8: Final validation + single commit

**Files:**
- Read-only checks across all modified files

Final cross-cutting validation, then ONE commit covering all changes from Tasks 1–7.

- [ ] **Step 1: Cross-cutting validation**

```bash
cd /Users/alepar/AleCode/llm-wiki

# Working tree should have changes to RUNBOOK.md, README.md, docs/prior-art.md only
git status
git diff --stat

# Spec coverage spot-checks
grep -c 'INDEX_PATH' RUNBOOK.md                           # expected: at least 15
grep -c 'QMD_CONFIG_DIR' RUNBOOK.md                       # expected: at least 8
grep -c '.qmd/index.sqlite' RUNBOOK.md                    # expected: at least 8
grep -c 'b77559223025cbcff3f992df0bf01147497c3bab' RUNBOOK.md  # expected: at least 1
grep -c '${CLAUDE_PROJECT_DIR' RUNBOOK.md                 # expected: at least 3 (.mcp.json + failure mode + maybe more)
grep -c 'Per-vault' RUNBOOK.md                            # expected: at least 4

# No leaked editing artifacts
grep -rn 'TBD\|TODO\|FIXME\|XXX' --include='*.md' \
  RUNBOOK.md README.md CLAUDE.md docs/prior-art.md || echo "no markers"

# RUNBOOK.md still has correct top-level structure
grep -c '^# Part ' RUNBOOK.md             # expected: 4
grep -c '^## Phase ' RUNBOOK.md           # expected: at least 9 (was 18 before; some are inside skill content; sanity check)
grep -c '^### Task ' RUNBOOK.md           # expected: at least 25

# Tasks 1.6 / 1.7 renaming
grep -c '### Task 1\.6 — Scaffold per-vault qmd config' RUNBOOK.md  # expected: 1
grep -c '### Task 1\.7 — First commit' RUNBOOK.md                   # expected: 1
```

If any expected count is off, return to the corresponding task and inspect what was missed.

- [ ] **Step 2: Single commit covering all Task 1–7 changes**

```bash
cd /Users/alepar/AleCode/llm-wiki
git add RUNBOOK.md README.md docs/prior-art.md
git commit -m "$(cat <<'EOF'
docs: per-vault qmd index (leverages b775592 upstream fix)

Each produced vault now gets its own qmd index at
<vault-root>/.qmd/index.sqlite (gitignored) instead of sharing the
global ~/.cache/qmd/index.sqlite. Two committed files at vault root
drive it:

- .mcp.json with env block (INDEX_PATH + QMD_CONFIG_DIR via
  ${CLAUDE_PROJECT_DIR} expansion) shadows the marketplace plugin's
  qmd MCP server entry per Claude Code's scope precedence (Project >
  User > Plugin).
- .qmd/index.yml is qmd's auto-loaded per-collection config
  (loaded from $QMD_CONFIG_DIR/<indexName>.yml).

Index sqlite + WAL/SHM are gitignored; everything else under .qmd/
stays committed. Hard floor on qmd post-b775592
(b77559223025cbcff3f992df0bf01147497c3bab, 2026-05-09). The
update-vendors skill marks below-floor versions as REQUIRED (not
optional) upgrades.

Phase 1 gains a new Task 1.6 scaffolding the .qmd/index.yml and
.mcp.json (old Task 1.6 commit becomes 1.7). Phase 4.5 SessionStart
hook exports INDEX_PATH and QMD_CONFIG_DIR. Phase 5 vault CLAUDE.md
template documents the per-vault binding and uses env-prefixed
commands. Phase 6 qmd setup adds version-floor check and uses
env-prefixed setup commands. Phase 7 smoke test verifies MCP picks
up the env-driven per-vault index.

Source: docs/superpowers/specs/2026-05-09-qmd-per-vault-index-design.md

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
git log --oneline | head -5
```

- [ ] **Step 3: Confirm commit landed**

```bash
git status   # expected: clean
git log -1 --stat
```

Expected: clean tree; one new commit with three files modified (RUNBOOK.md, README.md, docs/prior-art.md).

---

## Self-Review Notes

After Tasks 1–8, spec coverage:

- §3 Approach — `.mcp.json` + qmd-native config: Tasks 2, 3, 4.
- §4 vault-root files: Task 2 (Phase 1 scaffolding adds `.mcp.json` and `.qmd/index.yml`; `.gitignore` updated).
- §5 CLI invocations env-prefixed: Tasks 4 (Phase 5 vault CLAUDE.md template) and 5 (Phase 6.3 setup commands).
- §6 SessionStart hook: Task 3.
- §7 Marketplace plugin shadowing: covered by `.mcp.json` content (Task 2) + agent-automation policy text (Task 4).
- §8 Version floor: Tasks 5 (Phase 6.1 detection) and 6 (update-vendors REQUIRED-upgrade marking).
- §9 Verification: Task 5 (Phase 6.4 success criteria + Phase 7 smoke test).
- §10 Build sequence: this entire plan.
- §11 Open questions: Part IV failure-modes table covers the `${CLAUDE_PROJECT_DIR}` non-expansion case (Task 6).

No placeholder steps; every Edit has exact `old_string` from the current RUNBOOK.md state and exact `new_string` to substitute.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-05-09-qmd-per-vault-index.md`. Two execution options:

**1. Subagent-Driven (recommended)** — Dispatch a fresh subagent per task; review between tasks; fast iteration. Best fit here because each task is mechanical (specific Edits with exact anchors) and the parent agent's context stays clean for review.

**2. Inline Execution** — Execute tasks in this session using executing-plans; batch execution with checkpoints for review.

Which approach?
