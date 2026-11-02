# AGENTS.md — Practical Software Engineering Workflow

**English (default)** · [简体中文](AGENTS_ZH.md)

Use these instructions for agent-assisted software development, debugging, refactoring, and delivery. The goal is **work that can be executed, verified, reviewed, and rolled back**, not mandatory ceremony for every task.

These are general working practices, not mandates for any programming language, configuration format, build system, installer, or tool. The user's explicit request, available permissions, and more specific project-level instructions take precedence. A document alone does not enforce GitHub branch protection, CI, or deployment permissions.

## 0. Choose the workflow and respect authorization

| Task | Minimum workflow | Evidence of completion |
| --- | --- | --- |
| Question / read-only review | Inspect → explain findings | Facts, hypotheses, and unknowns are distinguished; no writes |
| Small change | Review → Change → Verify | Relevant diff and checks pass |
| Bug fix / refactor | Issue (if useful) → Review → Plan → Change → Verify → PR | Root cause, blast radius, and regression evidence are documented |
| Install / migrate / release | Review → Plan → Change → Verify → Deliver | Real target-platform user journey checked, with a recovery plan |

Treat local edits, Commit, Push, Merge, Release, Deploy, and deletion/overwriting of user data as **separate actions**. Do not infer permission for one from permission for another. Do not repeatedly seek approval for operations already authorized; leave unauthorized, high-impact steps ready for review. **Modify README files, including translations, only when the user explicitly authorizes README changes for the task.**

## 1. Issue — define an acceptance-testable problem

Before changing a repository, inspect relevant issues, PRs, and concurrent branches. A trivial change does not need an issue. For tracked work, record **Problem, Expected, Reproduction, Acceptance, Scope**, and what user data must not change.

**Example: extension UI disappears while the local service is offline**

```text
Problem: The sidebar only says "Cannot connect" when the service is stopped.
Reproduction: Stop the test service → open the sidebar → refresh.
Expected: Navigation and settings still render; online-only actions indicate offline.
Acceptance:
  1. On a real Windows browser, navigation and settings remain visible offline.
  2. After restarting the service, reconnecting works without reinstalling.
  3. Existing SQLite data and user settings remain unchanged.
Scope: Extension initialization, connection state, and error handling.
Out of scope: Redesigning the complete UI.
```

If the problem cannot be reproduced, say exactly what was observed and which hypotheses remain unverified. Do not report a hypothesis as a confirmed root cause.

## 2. Review — collect evidence before editing

Check the working directory, branch, worktrees, uncommitted changes, runtime environment, and the **real execution path**. Keep read-only requests read-only. Use authorized terminal, source-search, and browser/device tools; avoid unrelated credentials, cookies, and private data.

**Example: establish a repository baseline (run inside the repository)**

```bash
git status --short --branch
git branch --show-current
git log -5 --oneline
git diff --stat
rg -n 'connect|offline|retry' .   # Illustrative terms; adapt to the codebase.
```

Trace entry point → caller → configuration → state/data → dependencies → error path. For bugs, record before-change evidence such as reproduction steps, timestamps, response status, logs, and screenshots, including the target platform. **Never overwrite another task's uncommitted work.**

## 3. Plan — model the domain, evaluate reuse, define verification

For architecture work, start with **one natural-language sentence defining the domain**. Extract actors, entities, operations, rules, boundaries, and invariants; then map them to components, contracts, dependencies, and data flows. Validate the result using an actual user path **and a failure path**, rather than choosing a framework first.

**Example: decouple the local UI from service availability**

```text
Domain: Users can open the UI and settings even if the local service is stopped.
Entities/states: UI, service, connection state (online/offline/reconnecting).
Invariant: An unavailable service must not prevent UI rendering or erase data.
Boundary: The UI owns presentation/retries; the service owns data operations.
Success: Open UI → probe service → fetch data → render content.
Failure: Open UI → probe times out → render offline UI → retry → recover.
```

Before building generic infrastructure, examine existing dependencies and mature maintained components. Compare capabilities, integration effort, licenses, security, compatibility, maintenance, and operational cost. Write custom code only for a demonstrated gap; avoid unnecessary services, interfaces, events, or configuration.

**Minimum plan:** root cause or key hypothesis; impacted modules; behavior and data that must not change; success/failure tests; rollback path; authorization boundaries. Small low-risk changes may skip a written plan.

## 4. Change — isolate work and repair the complete affected path

Use independent branches or worktrees for concurrent tasks. Isolate shared ports, databases, processes, build outputs, and caches. Reuse an existing Docker/Compose setup when a repeatable multi-service environment is needed; do not force containers onto simple tasks.

**Example: isolate a branch (illustrative path and branch)**

```bash
git fetch origin
git worktree add ../app-offline-fix -b fix/offline-ui origin/main
cd ../app-offline-fix
# Edit, build, and verify in the isolated worktree.
git diff --check
git status --short
```

Address the root cause across the **affected path**: callers, configuration, state transitions, timeouts/retries, resource cleanup, compatibility, logging, and migration. Once the replacement works, remove superseded code, duplicate paths, and misleading settings. Do not expand a repair into an unrelated rewrite.

- **Configuration:** Keep the project's existing TOML/YAML/JSON model unless there is an authorized migration. Confirm every reader, writer, precedence rule, and schema validation. Avoid hard-coded personal paths, ports, tunnel identifiers, or hidden defaults. Keep secrets out of source, examples, and logs.
- **Data:** Before migrating SQLite, identify its actual path, WAL state, and user-owned files. Use a consistent online backup or stop the database cleanly; verify the backup can be read. Never replace a populated database with an empty one.
- **Logging:** Record meaningful states, duration, failures, and correlation IDs using the existing logging system; never log credentials or unrelated personal information.
- **Concurrency:** Give shared resources clear ownership. Parallel work should not mutate the same file or live database without coordination.

## 5. Verify — demonstrate results with evidence

Validate **real user journeys**. Select relevant static checks, builds, unit tests, integration tests, browser checks, and target-platform installs based on change risk. Do not require every test category for every task or categorically skip relevant tests.

**Example: Go backend with a local UI (only if the project supports these commands)**

```bash
go test ./...             # Useful where applicable; not an end-to-end substitute.
go build ./...
git diff --check
# After starting the service in its real environment:
curl --fail http://127.0.0.1:18080/health
```

Re-run the original failing case, then check a normal journey, an important failure/recovery journey, and user-data integrity. **A successful Linux/WSL build does not prove Windows installation works; an emulated browser does not prove an actual extension works.**

Use only these statuses: **Passed** (observed evidence), **Failed** (executed but failed), **Not Tested**, or **N/A**. Record the environment, action/command, observed result, and evidence. Never mark a guess as Passed.

**Example verification matrix — illustrative only, not actual project test results**

| Check | Environment/action | Status | Evidence |
| --- | --- | --- | --- |
| Build | WSL, `go build ./...` | Passed | Exit code 0 |
| Offline UI | Windows extension with test service stopped | Passed | Navigation visible; online-only actions disabled |
| Reconnection | Restart service and retry in Windows | Passed | State changes to online |
| macOS installation | No macOS test device | Not Tested | Do not claim support was verified |

## 6. PR — make changes inspectable and reproducible

Create a PR for normal code changes. Reference relevant issues and explain the problem, root cause, scope, test evidence, risks, and rollback. Review the diff for secrets, user data, generated files, and unrelated edits before pushing.

**Example: commit and push (only when Push is authorized)**

```bash
git diff --check
git status --short
git add path/to/changed-file
git commit -m "fix: keep UI available while service is offline"
git push -u origin fix/offline-ui
# Open a PR using an available GitHub tool and attach verification evidence.
```

**PR description example**

```text
Problem / Reproduction:
Root Cause:
Changes / Out of Scope:
Verification (environment, steps, Passed/Failed/Not Tested):
Data & Compatibility:
Risk / Rollback:
Related Issue: #N
```

Use `Fixes #N` only when merging truly satisfies the entire issue; otherwise use `Refs #N`. Honor repository-required checks, branch protection, and **Squash Merge** when configured. Merge only when authorized. Delete branches/worktrees only after confirming they are merged and unused.

## 7. Deliver — shipping is more than compiling

**Commit, Push, Merge, Release, and Deploy are distinct states.** Execute and report each separately according to the user's authorization. Before release, verify download/install paths and choose a package, script, or installer based on product needs.

**Example: upgrade an app that stores data in SQLite**

```text
1. On the target platform, identify the installed version and real data path.
2. Take a consistent backup and verify it can be opened.
3. Install the new version and verify startup and configuration migration.
4. Compare data records, important actions, and UI with the baseline.
5. Test service stopped, port conflict, and expected recovery behavior.
6. If migration fails, stop and use the verified rollback; never delete data to retry.
7. Mark target-platform validation Passed only after actually executing it.
```

After an authorized batch of website changes, run the project's actual build, start a local preview, validate affected routes and rendered pages, and preserve preview access until review is complete where feasible. Production deployment requires separate authorization. Change README files only when expressly authorized; use reproducible installation, configuration, run, and verification instructions without unnecessarily pinning fast-changing versions.

## 8. Retrospect — report precise delivery status

Report changed files, branch/PR/commit, checks marked Passed/Failed/Not Tested, effect on user data, Merge/Release/Deploy status, and unresolved risks. Do not say “all done” if a required platform or workflow remains untested.

**Example handoff**

```text
Code: PR #42 opened, not merged.
Changes: Offline UI, reconnection state, error messages.
Verified: Windows offline/recovery Passed; Go build Passed.
Not tested: macOS installation.
Data: Existing SQLite was neither migrated nor deleted.
Shipping: No Release or Deploy performed.
```

## 9. Three complete execution examples

The following are **hypothetical agent workflows**, not reports of actual tests performed on a repository.

### Case A — The entire UI disappears when the service is offline

1. **Review:** Trace UI initialization and service probing; stop only the test service under your control; reproduce the blank UI and capture errors; inspect related issues and PRs.
2. **Plan:** Separate initial rendering from remote data fetch. Model `offline / connecting / online`, disabling only operations requiring service access.
3. **Change:** Fix initialization, retry, and failure branches. Search for the same assumption in other affected routes rather than adding a second competing state machine.
4. **Verify:** Use a real browser to test offline startup, online startup, reconnection, timeouts, and unchanged user data. Record each result honestly.
5. **PR:** Attach before/after reproduction steps, logs, and untested-platform notes. Do not merge without authorization.

### Case B — Move a hard-coded port into project configuration

1. **Review:** Search constants in service startup, launchers, extensions, Web UI, and MCP entry points. Identify the actual current configuration file and precedence.
2. **Plan:** Retain the project's configuration format. Define and validate `listen_address`, ensure every entry point consumes the same effective value, and specify port-conflict handling.
3. **Change:** Migrate all relevant readers and writers; remove obsolete hard-coded paths while preserving necessary backward-compatible migration.
4. **Verify:** Start and restart on a non-default port; connect the extension and MCP; check malformed port errors; verify data paths and credentials remain unchanged.
5. **PR:** Include a configuration example and test matrix. If README changes are not authorized, note them for later without editing.

### Case C — Change a website project page for user review

1. **Review:** Find the real site repository, route, reusable components, and any concurrent work on the same files.
2. **Plan:** Reuse the existing article/project components instead of duplicating the component system for one page.
3. **Change:** Adjust only in-scope layout, content, and styles; account for translated routes and image resources.
4. **Verify:** Build locally, start preview, and inspect desktop/narrow layouts, images, navigation, and links in the rendered route.
5. **PR:** Provide the preview URL and screenshots or explicit inspection results. A local preview is not production deployment.

---

**A useful rule allows another agent, given only the task, repository, and this document, to determine what to check first, how to change it, how to verify it, and when to stop.** Add concrete steps where that is not possible rather than more abstract slogans.
