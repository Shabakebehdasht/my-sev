# Verifying a Pull Request (inbound review)

When the user asks to "check PR #N" / "verify this PR" / "صحت عملکرد PR رو بررسی
کن": your job is to REPRODUCE the PR's claims. A green CI badge and a written
verification table in the PR body are claims, not evidence. GitHub mechanics
(posting comments, submitting the verdict) live in the `github` skill; this file
is the verification procedure.

## 1. Fetch without disturbing the session's branch

```bash
git fetch <canonical-repo-url> pull/N/head:pr-N   # adds a local ref, no remote changes
```

Check it out in a **worktree**, never with `git checkout` — the working branch,
its remotes and uncommitted state must stay untouched:

```bash
git worktree add --detach /path/to/wt pr-N
cp .env /path/to/wt/.env     # gitignored; the suite needs APP_KEY
```

For a Laravel/PHP checkout, prepare the harness FIRST (vendor copy +
`composer dump-autoload -o`, `public/build` symlink, `config:clear` /
`route:clear`) — see `laravel-livewire` → "Running the Suite in an Isolated
Checkout". Skipping this produces failures that look like PR defects but are
harness gaps.

## 2. Read the PR body as a list of claims to test

Each claim needs a command that would contradict it if false:

| Claim | Disproof command |
|---|---|
| test counts ("N passed") | run the project's own entrypoint (`composer test`) |
| "file X unmodified" | `git diff base...head -- X` (empty) |
| "baseline: deletions only" | `git diff base...head -- <generated-file> \| grep -c '^+'` → 0 |
| lint / type gate | run the EXACT CI command — `pint --test`, never `pint --dirty` (a no-op on a clean tree, makes the gate vacuous) |
| API/payload "byte-identical" | run the API contract test file, untouched by the PR |

## 3. Two separate evidence tracks

1. `gh pr checks <N> --repo <owner>/<repo>` — what CI actually did.
2. Your own local run — the FULL suite via the project entrypoint, not only the
   tests the PR mentions; the interesting regressions are in the ones it omits.

Never report CI green or a PR claim as verified without a fresh read/run.

## 4. Classify every failure before reporting it

- Missing build artifact, stale autoloader, absent local env, missing gitignored
  config → **harness**: fix it, re-run, and say so in the report.
- Survives a prepared checkout → **finding on the PR**.

Label which kind each failure was, so the author is not sent chasing your
environment.

## 5. Pin every finding with a runnable proof

A finding backed only by reading code is a suggestion until it reproduces:

- behaviour → a throwaway Pest/Livewire probe test (write it, run it, **delete it**);
- type suspicion (`in_array(..., true)` vs what the DB layer returns) → a
  `gettype()` dump through tinker;
- "really zero additions" claims → a diff-count command;
- client-writable state → `Livewire::test('c')->set('p', 'v')->assertSet('p', 'v')`,
  then re-render with `->html()` to prove the value is USED, not merely stored.

## 6. Report honestly

Format: **Critical / Warning / Suggestion / Looks Good**, one line each, every
claim backed by command output. Anything you could not execute (browser
install blocked, tool unavailable) is reported as **UNVERIFIED** — never inherit
the PR's own claim for a check you did not run.

## Pitfalls

- **`git checkout` of the PR branch in the session repo** — use a worktree; the
  session's current branch must stay checked out.
- **Reviewing the diff alone** — prose review confirms nothing; every
  behavioural claim needs a contradictable command.
- **Reporting an environment failure as a PR defect** — classify first, report
  second.
- **Probes left behind** — a throwaway test file inside the worktree pollutes
  `git status`; delete it before the final report.
- **Cleanup** — `git worktree remove /path/to/wt` plus the temp local branch when
  done (the copied `vendor/` is the expensive part).
