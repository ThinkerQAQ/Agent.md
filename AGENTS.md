# AGENTS.md — Engineering Workflow

**English (default)** · [简体中文](AGENTS_ZH.md)

Use this as a practical checklist for agent-assisted software work. **Follow the user's explicit instructions and project-specific rules first.** Adapt effort to risk: a question needs no PR; a production migration needs evidence and a rollback plan.

## 0. Scope and authorization

Read-only review, local edits, Commit, Push, Merge, Release, Deploy, and deleting user data are **separate permissions**. Do not assume authorization for one implies another. README changes (including translations) require explicit authorization for the task.

**Example:** Asked to “fix the bug and open a PR” → change, verify, push, and open the PR; leave Merge/Release/Deploy untouched.

## 1. Issue — define success

Check related issues and PRs. For nontrivial work, record the problem, reproduction, expected behavior, acceptance criteria, and what is out of scope. Small changes do not need an issue.

**Example:** “With the local service stopped, the extension sidebar must still show navigation; service-dependent actions indicate offline; restarting the service restores access without losing SQLite data.”

## 2. Review — inspect before editing

Check the branch, worktree, uncommitted changes, entry points, and the real execution path. Reproduce bugs when possible; separate **observed facts, hypotheses, and unknowns**. Never overwrite another task's changes or inspect unrelated credentials.

**Example (inside the repository):**
```bash
git status --short --branch
git diff --stat
rg -n 'offline|reconnect' .   # Replace with terms from the actual issue.
```
Trace UI → service/Bridge → configuration → storage; record the failing step before editing.

### Code intelligence — adaptive tool selection

Use **`rg` → CodeGraph → LSP → source review → verification** as a *decision guide*, not a mandatory sequence. Choose only the tools justified by the question and the cost of analysis.

1. **Identify the Git workspace.** Resolve the absolute repository/worktree root; inspect `git status --short --branch` and `git worktree list`. Bind searches, indexes, and diagnostics to that specific workspace. A branch name or a shared tool process does not uniquely identify a working tree.
2. **Locate with `rg`.** Search filenames, error messages, routes, identifiers, configuration, and call sites first. Narrow the scope and limit output rather than dumping the entire repository. Text matches are leads, not proof of semantic relationships.
3. **Map with CodeGraph when structure matters.** Use callers/callees, dependencies, symbol relationships, and change-impact queries for cross-module questions. Pass an explicit workspace when supported; check the indexed paths and freshness, especially after edits, branch switches, or file renames. Prefer on-demand, one-shot queries; for structural-only queries, use graph-only mode if available.
4. **Confirm with LSP when semantics matter.** Ask the language server for definitions, references, implementations, types, and diagnostics when text/graph evidence is insufficient. Match the project language/toolchain and the intended workspace. Compatible language-server daemons may share resources, **not** an interchangeable view of different worktrees. Starting a language server alone does not count as a successful semantic query.
5. **Read the authoritative source.** Inspect relevant implementations, callers, configurations, and tests before concluding or editing. Verify critical graph/LSP findings against code and, when appropriate, a build or test; distinguish observed facts, hypotheses, and unknowns.
6. **Report actual evidence.** Name the workspace, tools really used, relevant results, validation status, and unverified areas. Do not claim a tool ran, an index was current, or a call path was verified without evidence.

**Scale the workflow:** a simple fix may need only `rg`, source review, and focused tests; cross-module impact analysis may add CodeGraph; type/interface questions may add LSP; complex changes may warrant all three. If CodeGraph or LSP is unavailable, fall back to `rg` + source inspection + available checks and disclose the limitation. Reuse compatible binaries/caches, but avoid unnecessary persistent indexers or language-server processes. Keep machine-specific executable paths, runtime wrappers, and server setup in local instructions rather than this shared policy.

## 3. Plan — model the domain and choose an approach

For architecture tasks, state the domain in one sentence, identify actors/rules/invariants, then map components and validate success **and failure** flows. Inspect existing modules and maintained libraries before building custom infrastructure; compare fit, cost, license, and compatibility. Keep plans proportional to the change.

**Example:** “The UI can render without a running backend” → UI owns rendering and retry state; backend owns data operations; a timeout shows an offline UI and retries without clearing stored data.

## 4. Change — fix the affected path

Keep concurrent work isolated (branches/worktrees, ports, databases). Fix the root cause across relevant callers, configuration, state transitions, errors, and cleanup; remove replaced paths without unrelated rewrites. Keep the project's existing TOML/YAML/JSON conventions, validate settings, protect secrets, and back up user data consistently before migration.

**Example:** For a configurable port, update the launcher, service, extension, and MCP readers to use one validated setting; then remove the obsolete hard-coded port without changing the user's SQLite path.

## 5. Verify — show real evidence

Choose relevant build, unit/integration, smoke, browser, and target-platform tests. Re-test the original failure, a normal path, recovery, and data integrity. A WSL build **does not** prove Windows installation works. Report **Passed / Failed / Not Tested / N/A** with environment and observed result.

**Example:** “Windows + real extension: sidebar visible while service is stopped (Passed); restart reconnects (Passed); macOS installer (Not Tested).”

## 6. PR — make review reproducible

Review the diff for unrelated files, secrets, and generated artifacts. Include the problem, change, test evidence, risk, and rollback. Honor branch protections and the project's Squash Merge policy; merge only with authorization. Use `Fixes #N` only when the issue is fully resolved.

**Example:** “PR #42: decouple sidebar rendering from service connectivity; Windows offline/reconnect Passed; macOS Not Tested; no database changes; rollback by reverting the commit.”

## 7. Deliver — validate the user journey

Commit, Push, Merge, Release, and Deploy are distinct. For software delivery, check download, install, startup, upgrade, and recovery on actual target platforms. For website changes, build and inspect the local preview before asking for review; production deployment needs separate authorization. Protect existing SQLite data (including WAL) and verify backups before migration.

**Example:** “On Windows, back up and verify the existing SQLite database → install the update → confirm original records and settings → test startup failure and recovery → release only when authorized.”

## 8. Retrospect — state what really happened

Report changed files/PR, verification evidence, user-data impact, untested items, and the exact Merge/Release/Deploy status. Do not call incomplete verification “done.”

**Example:** “PR #42 open, not merged; Windows offline/reconnect Passed; macOS Not Tested; user database unchanged; no release or deployment.”
