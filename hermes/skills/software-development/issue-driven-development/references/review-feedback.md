# Responding to PR Review (superseding PR)

Use when review feedback exists on a PR whose head branch you cannot push to —
usually someone else's fork — and the user asks for the changes plus a NEW PR.
The fixes ship as your own branch off the reviewed PR's head, then a fresh PR
against the base repo.

## 1. Read every review surface

```bash
# Verdict state + narrative body + PR metadata (base repo)
gh pr view N --repo <base-owner>/<repo> \
  --json title,state,headRefName,baseRefName,author,reviews

# Inline comments, including ```suggestion fences
gh api repos/<base-owner>/<repo>/pulls/N/comments --paginate \
  | jq -r '.[] | "=== \(.user.login) | \(.path):\(.line // .original_line)\n\(.body)"'
```

`gh pr view --comments` / `--json comments` returns ONLY issue-level comments.
The blocking change is normally an inline comment carrying a `suggestion`
fence and the reasoning sits in `reviews[].body` (state `CHANGES_REQUESTED`) —
read both endpoints or you implement a fraction of what was asked.

Turn the feedback into an item list: code fix, regression test, doc wording.
Address the reviewer's minor notes too; an unanswered note is what lengthens
the next round.

## 2. Get the code without touching remotes

```bash
git fetch <base-repo-url> pull/N/head:refs/heads/pr-N   # PR refs live on the BASE repo
git checkout -b <fix-branch> pr-N
```

Fetching `pull/N/head` from your fork fails for cross-fork PRs, and adding a
remote to work around it mutates repo configuration the project forbids
touching — fetch by URL instead.

## 3. Prove the claim before applying the fix (RED first)

Add the reviewer's suggested regression test to the UNFIXED tree and run it:
expect RED, apply the fix, expect GREEN. Keep both results.

Pitfall — the reviewer's root-cause prose can be partly WRONG while the
suggested fix is still right: they trace one code path, another branch may
already satisfy some of their own examples. Let the failing assertion, not
their narrative, decide what your commit message and PR body claim.

## 4. One commit per independent fix

When two fixes add tests to the SAME test file, avoid interactive `git add -p`:
remove the second test block with your patch tool, commit fix one, re-add the
block, commit fix two. Each commit then stands alone with a green suite and
bisect stays honest.

## 5. Gates, and baseline what you did not touch

Run the repo's documented gates (from its `AGENTS.md`; for Laravel typically
`vendor/bin/pint --test`, `vendor/bin/phpstan analyse`, full test command).

- A failure or "risky" warning is not yours until you check it on the base ref:
  `git show <base-owner>:<base-branch>:<path>` (or `git show upstream/<base>:<path>`)
  and compare. If identical, say so in the PR body — never report a green run
  while hiding a pre-existing warning.
- Never edit a test file while a suite that reads it is running: kill the run,
  edit, rerun, or the result covers an inconsistent tree.
- Put a long suite in the background writing to a log file, then read the log's
  summary line — an exit code alone does not tell you the pass/risky counts.

## 6. Push to your fork, open the PR against the base repo

```bash
git push -u origin <fix-branch>
gh repo view <your-fork>/<repo> --json isFork,parent   # head must be a fork of base
gh pr create --repo <base-owner>/<repo> --base <base> \
  --head <your-fork>:<fix-branch> --title "..." --body "..."
```

Body shape:

- Open with `Supersedes #N.` plus the issue keyword the original PR carried
  (`Closes #<issue>`) so merging still closes the issue.
- One bullet per review item: what was asked → what changed → which test covers it.
- Verification numbers from YOUR run (test counts, lint, static analysis), with
  pre-existing failures named.
- Keep the original PR's good analysis below the review section instead of
  making reviewers re-read the old PR.

## 7. Stop where the instruction stopped

Do not close the reviewed PR or merge anything unless the user says so — the
commit/supersede call belongs to the reviewer. Report the new PR URL plus the
verification results.
