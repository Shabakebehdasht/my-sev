---
name: issue-driven-development
description: "GitHub issues to PR: sync, fan out, gates, commit-push-PR."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [github, issues, workflow, parallel, testing, pr]
    category: software-development
---

# Issue-Driven Development Workflow

Procedure for taking GitHub issues from fetch to merged-ready PR.

## When to Use

User provides GitHub issue numbers and asks to implement them, or says "انجام بده" followed by issue links, ending with commit/push/PR instructions.

## Procedure

### 1. Sync branch from upstream

```bash
cd <repo> && git fetch upstream beta && git merge upstream/beta --no-edit
```

Never hardcode fork URLs — read `git remote -v` first.

### 2. Fetch issue details

```bash
gh issue view <num> --repo <owner>/<repo> --json title,body
```

Batch multiple issues in one `execute_code` call to save round-trips.

### 3. Fan out implementation

For independent issues, dispatch parallel subagents via `delegate_task`. Each subagent gets:
- Exact file paths to modify
- The schema/data it needs inline (don't make it re-query)
- Verification commands to run after changes
- Instruction to return a summary of what changed

Group related issues in one subagent when they touch the same files.

### 4. Quality gates (before every commit)

```bash
php vendor/bin/pint --dirty          # code style
vendor/bin/phpstan analyse --no-progress  # static analysis
XDEBUG_MODE=off php artisan test --parallel  # full test suite
```

All three must pass. If PHPStan errors reference deleted files, regenerate baseline: `vendor/bin/phpstan analyse --generate-baseline`.

### 5. Commit and push

```bash
git add -A && git status --short  # verify staged files
git commit -m "type: description — closes #<issue>"
git push origin <branch>
```

### 6. Create PR

```bash
gh pr create --repo <upstream-owner>/<repo> --base beta \
  --head <fork-owner>:<branch> --title "..." --body "..."
```

GitHub MCP `create_pull_request` may auth-fail — fall back to `gh` CLI.

### 7. Monitor CI

```bash
gh pr checks <num> --repo <owner>/<repo>
```

Blocking checks: Pint, Tests & Coverage, PHPStan. Mutation Testing is non-blocking. Checks take 2-5 min to appear after push.

## Pitfalls

- **`Sanctum::actingAs()` mock tokens break ability middleware.** The Sanctum Guard resolves the user from session first, wraps with `TransientToken` (no abilities), so `ability:` middleware sees null `currentAccessToken()` → 401. Create real tokens via `$user->createToken('test', $abilities)` and authenticate with Bearer header instead.

- **`Route::middleware()` with multiple args needs array syntax.** `middleware('a', 'b')` fails PHPStan with "invoked with 2 parameters, 1 required". Use `middleware(['a', 'b'])`.

- **Route ability middleware nests inside role_or_permission middleware.** Tests need BOTH: the token ability AND the Spatie permission. Passing one without the other returns 403 — check which layer rejected before debugging.

- **Scheduler job dispatch tests check `event->description`, not `event->command`.** `$schedule->job(new SomeJob)` sets the description to the job class FQN; artisan commands set `command`.

- **Deleting a Blade component requires deleting its test file.** Test referencing a deleted component throws `ComponentNotFoundException` — grep tests for the component name before removing.

- **PostgreSQL sequences after seeding with explicit IDs.** Call `SELECT setval('table_id_seq', COALESCE((SELECT MAX(id) FROM table), 1))` to avoid duplicate key errors in subsequent tests.

- **CI check timing.** GitHub checks show `pending` immediately but take 2-5 min to start. Poll every 60s, not immediately after push.

## Parallelization pattern

For N independent issues, dispatch N subagents in one `delegate_task` call. Each gets:
- Self-contained goal with exact file paths and verification commands
- Inline schema/data so it doesn't re-query
- Instruction to report what changed

After all complete: run full quality gates yourself (subagents may miss cross-file interactions), then commit-push-PR as one unit.