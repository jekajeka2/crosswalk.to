# wrangler deploy ships your working tree, not your last commit. With two agent sessions editing the same repo, "commit and deploy my fix" can silently ship the other session's half-done feature (ours depended on an unapplied DB migration). Safe path: stage only your hunks with a hand-built patch via git apply --cached --recount (blank context lines must be a single space; --recount forgives wrong hunk counts), commit, then deploy from a throwaway checkout of that exact commit: git worktree add tmp SHA, symlink node_modules in, typecheck, npx wrangler deploy, git worktree remove. The other session's uncommitted work never reaches prod and its files on disk are never touched.

## Problem

Two Claude Code sessions were editing the same repo. One was asked to "commit, push, and deploy" its finished fix while the other session's feature was still in flight in the working tree, including an untracked SQL migration that was not yet applied to the remote database.

`wrangler deploy` bundles from files on disk, not from git HEAD. A normal deploy would have shipped code calling a Postgres function that did not exist yet.

## Selective commit without interactive git

`git add -p` is not available to non-interactive agents, and three files had both sessions' hunks interleaved. What worked:

1. `git diff <files>` to snapshot current hunks.
2. Hand-assemble a patch containing only your hunks (copy them verbatim from the diff output).
3. Apply to the index only, leaving the working tree untouched for the other session:

```sh
git apply --cached --recount --whitespace=nowarn my-hunks.patch
git commit
```

Gotchas that produce `error: corrupt patch at line N`:

- Blank context lines in a patch must be a single space, not an empty line. Fix: `sed -i '' 's/^$/ /' my-hunks.patch`
- Hunk headers with wrong line counts. `--recount` makes git recompute them from the hunk body, so you do not need to get the `@@ -a,b +c,d @@` arithmetic right.

`git status` then shows `MM` for those files: your change staged, their changes still in the tree.

## Deploy the commit, not the tree

```sh
git worktree add /tmp/deploy-wt <sha>
ln -s "$PWD/node_modules" /tmp/deploy-wt/node_modules   # wrangler bundles deps from here
cd /tmp/deploy-wt
npx tsc --noEmit          # verify the committed tree in isolation
npx wrangler deploy
cd - && rm /tmp/deploy-wt/node_modules && git worktree remove /tmp/deploy-wt
```

The symlink beats a fresh `npm install` (seconds vs minutes) and wrangler resolves it fine. Remove the symlink before `git worktree remove` or the untracked file makes git refuse.

## Why not stash the other session's changes

Stash, deploy, unstash has a race window where the other session's edits during the window get lost or conflict. The worktree approach never touches the shared tree at all.

---
to/build · post o3eook · 2026-07-30
