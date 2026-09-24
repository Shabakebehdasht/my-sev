---
name: git-workflow
description: "Git pitfalls: nested repos, staging, identity."
version: 1.1.0
author: Sydney
license: MIT
metadata:
  hermes:
    tags: [git, workflow, pitfalls, staging, nested-repos, commit, session-init]
    category: software-development
---

# Git Workflow

## When to Use

Use when committing, pushing, branching, or resolving git-related issues in any
project. Also use at session start to initialize the working environment (git
sync, MCP tool verification, reading project docs).

Pitfalls and procedures for everyday git operations that fall outside standard
`gh` CLI workflows (for gh-specific flows see the `github` skill).

## Standing Rules

- Always check `git status` and `git remote -v` before staging or pushing.
- Configure per-repo identity before first commit if global config is absent.
- Never assume a branch name — read it from `git branch --show-current`.
- Verify MCP tools are operational at session start, not mid-task.

## Session Initialization

For projects with MCP tooling (Laravel Boost, Context7, CodeGraph, GitHub MCP),
verify all tools at session start before doing development work. Do not assume
they are running — probe each one. Fix failures immediately so the toolchain is
ready before the first real task.

### Verification order

1. `git remote -v` + `git branch --show-current` + `git status`
2. Sync current branch: `git fetch upstream <base-branch> && git merge upstream/<base-branch> --no-edit`
3. Read project AGENTS.md (if present)
4. Probe each MCP tool with a lightweight call (e.g. `application_info`, `list_pull_requests`)
5. Fix any broken tools before proceeding

See `references/session-init.md` for project-specific initialization procedures
(h-dashboard, and patterns for future projects).

## Pitfalls

### Nested `.git` directories when copying content

When `cp -r` (or similar) a directory that contains its own `.git` into another repo,
`git add` detects the nested `.git` and stages only a submodule reference (mode 160000),
not the actual files. Clones of the outer repo will not contain the copied content.

**Fix — remove nested `.git` before staging:**

```bash
cp -r /source/dir target/inside/repo
rm -rf target/inside/repo/.git
git add target/inside/repo/
```

If already staged as submodule:

```bash
git rm -r --cached target/inside/repo/
rm -rf target/inside/repo/.git
git add target/inside/repo/
git commit -m "Add dir contents (fix nested repo)"
```

### Unintended file changes in feature commits

When committing feature work, run `git diff --stat` before staging to catch
unintended modifications to config/meta files (e.g. `AGENTS.md`, `.hermes.md`)
that were modified by tooling or agents during the session but are not part of
the feature.

**Prevention:** `git add -p` or selective `git add <paths>` instead of
`git add -A`. Review `git diff --staged --stat` before committing.

**Fix — restore from upstream before amend:**

```bash
git show upstream/beta:AGENTS.md > AGENTS.md
git add AGENTS.md
git commit --amend --no-edit
```

### Missing git identity on fresh clones

New clones may lack both global and per-repo `user.name`/`user.email`.
Commits fail with `empty ident name`.

**Fix — set per-repo config before first commit:**

```bash
git config user.email "user@users.noreply.github.com"
git config user.name "username"
```

Detect from `gh auth status` or set manually.

## Verification

- `git status` shows no unexpected submodule entries.
- `git diff --cached --stat` shows actual file additions, not just mode changes.
- Commit and push succeed without warnings about embedded repos.
- All configured MCP tools respond to probe calls.
