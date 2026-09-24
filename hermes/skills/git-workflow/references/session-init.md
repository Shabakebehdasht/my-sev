# Session Initialization Procedures

Project-specific initialization procedures for session start. Each project
has its own git remotes, branch naming, and toolchain — never hardcode
remotes or branch names; always read them from the existing git config.

## h-dashboard

### Git setup

```
origin  → Shabakebehdasht/h-dashboard (fork)
upstream → asgarimehdi/h-dashboard (canonical)
```

Current branch: read from `git branch --show-current`. Push to the same branch
on `origin`. Never push directly to `upstream/beta`.

**Sync procedure:**

```bash
cd h-dashboard
git remote -v
git branch --show-current
git status
git fetch upstream beta
git merge upstream/beta --no-edit
git push origin <current-branch>
```

Do NOT re-clone. Do NOT modify remotes. Do NOT switch branches unless instructed.

### MCP tools

All four MCP servers must be verified at session start:

| Tool | Config | Probe call |
|---|---|---|
| Laravel Boost | `php artisan boost:mcp` | `application_info` |
| Context7 | Remote HTTP `mcp.context7.com` | `resolve_library_id` |
| GitHub | `npx @modelcontextprotocol/server-github` | `list_pull_requests(owner, repo)` |
| CodeGraph | `codegraph serve --mcp` | (CLI availability check) |

**CodeGraph installation:**

The correct npm package is `@colbymchenry/codegraph-linux-x64`, NOT the bare
`codegraph` package (which is an unrelated library with no CLI binary).

```bash
npm install -g @colbymchenry/codegraph-linux-x64
```

After install, the binary lands in the npm global `lib/node_modules` but may not
get a bin symlink. Create one manually:

```bash
# Find the binary
find $(npm root -g) -name codegraph -type f
# Symlink it
ln -sf <found-path> $(npm prefix -g)/bin/codegraph
chmod +x <found-path>
```

**Laravel Boost MCP crash:** If `application_info` returns "lost its stdio
subprocess", retry once — it often works on the second attempt. If it still
fails, check that `php artisan boost:mcp` runs without errors from the project
directory.

### PR convention

When the user says "pr": open a PR from the current branch to `beta` on the
canonical repo (`asgarimehdi/h-dashboard`). Do NOT merge unless explicitly
asked. Use GitHub MCP (`create_pull_request`).

### Skills

- `read-the-damn-docs` — always loaded before implementing changes
- `shadcn/improve` — loaded for audits, not executed unless requested
- `AGENTS.md` — always read and kept in context
