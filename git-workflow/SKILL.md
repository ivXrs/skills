---
name: Git Workflow
description: "Personal Git delivery workflow: typed branches, selective staging, atomic Conventional Commits, push, GitHub PRs via gh, squash merge, and cleanup."
---

# Git Workflow

Deliver the smallest complete change through a typed branch and a PR. Read the diff and the files it touches before proposing anything. Use for PR delivery; for local-only commits, ask before creating a branch or PR.

## Hard Rules

1. Stage exact files only: `git add -- "file"`. No globs, pathspecs, directories, `git add -A`, `git add .`, `git add -u`, `git add --all`, `git add :/`, `git add *`, `git add -p/-i`, `git add --pathspec-from-file`, `git commit -a`, `git commit -am`, `git commit --all`, `git commit --only`, `git commit --include`, or `git commit --patch`. The index must be empty before each commit; after staging, `git diff --cached --name-only` must exactly match the plan.
2. Never force-push or rewrite remote refs. The only allowed pushes are `git push -u origin <branch>` and, after explicit cleanup approval, `git push origin --delete <branch>`. Forbidden push forms: `git push --force`, `-f`, `--force-with-lease`, `--force-if-includes`, `--mirror`, `--prune`, `--all`, `+<refspec>`, `:<refspec>`, `HEAD:main`, and any push to `main`.
3. Never commit on `main` or detached HEAD. Verify `git branch --show-current` before every commit.
4. Two approvals: Gate 1 approves sync/branch/commits/push/PR; Gate 2 approves merge. Plan approval is not merge approval. The only merge command is `gh pr merge <n> --squash ...`; never `--auto`, `gh api`, a web merge, or a push to `main`.
5. Never use `gh pr merge --admin` or bypass branch protection, required reviews, required checks, or a merge queue.
6. `git branch -D` only if all: PR `MERGED`; branch tip equals the Gate 2 SHA; tree clean; branch not in another worktree; explicit confirmation.
7. Never discard work without explicit confirmation. This includes `git reset --hard`, any `git clean -f/-fd/-fdx/-fX`, `git checkout -- <path>`, `git restore <path>`, `git switch -f`, `git stash drop/clear`, `git worktree remove --force`, and any remote ref deletion.
8. No amend/rebase of published commits without approval. Prefer new commits.
9. If `gh` is missing, ask to restart the shell so PATH refreshes. Do not modify PATH.

Hard Rules 1-9 are non-negotiable. Replace every `<...>` placeholder with a concrete value before running a command; never execute a placeholder.

## Conventions

- Base `main`, remote `origin`. `gh repo view` must resolve to the `origin` repository; otherwise stop.
- Branch: `<type>/<slug>`, type from `feat|fix|chore|docs|refactor|test|perf|ci|build|revert`; slug lowercase kebab-case.
- Commit: Conventional Commits in English, atomic, imperative summary, scope optional.
- PR title: same format. Body: `Summary / Changes / Testing / Notes`.
- English artifacts: branch names, commit messages, PR titles/bodies, merge messages. Do not translate code or docs.
- Smallest complete delivery: only files required for the task.

## Workflow

### 0. Preflight
```bash
git status --short --branch
git worktree list
git remote -v
git branch --show-current
git fetch origin
git merge-base --is-ancestor main origin/main
git rev-parse HEAD
gh --version
gh auth status
gh repo view --json nameWithOwner,defaultBranchRef
```
Stop if: not a repo, no `origin`, no `main`, detached HEAD, `gh` missing/unauthenticated, default branch not `main`, `gh repo view` does not match `origin`, local `main` is not an ancestor of `origin/main` (local-only commits), or `main`/the target branch is checked out in another worktree. If already on a non-`main` branch, stop and ask; this workflow starts a new branch from `main`. On any network or auth failure, stop and report; do not retry destructive or merge operations.

### 1. Review
```bash
git status --short
git diff
git diff --cached --name-only
```
Read the changed files and diff. The index must be empty before Phase 3; if anything is staged, stop and ask. Unstage only with approval. Identify changes outside the task and exclude them. If a planned file contains unrelated changes, ask.

### 2. Plan (Gate 1)
Present:
```text
Branch: <type>/<slug> (new from main)
Sync: git fetch origin main; git merge --ff-only origin/main
Commits:
  1. <type>(scope): <summary> — files: <paths>
  2. ...
Push: git push -u origin <branch>
PR: <title> — <full English body>
Merge: gh pr merge <n> --squash --delete-branch --match-head-commit <sha>
Cleanup: git fetch --prune; switch main; fast-forward; verify branch
```
Show the exact English commit messages and full PR body, not just section names. Do not create the branch, sync, commit, push, or open the PR before approval. If the user edits, re-present; no partial execution.

### 3. Sync, branch, commits
```bash
git fetch origin main
git merge --ff-only origin/main
git branch --list <branch>
git ls-remote --heads origin <branch>
git switch --no-guess -c <branch> main
git branch --show-current
```
If the local or remote branch exists, stop and ask. If the remote check fails, stop. If the sync is not fast-forward, stop. If `git status --short` shows paths outside the plan, stop and ask; uncommitted planned changes are expected. Create from `main` explicitly.

For each commit:
```bash
git branch --show-current
git diff --cached --name-only
git add -- "path/one" "path/two"
git diff --cached --name-only
git diff --cached --check
git diff --cached
git commit -m '<subject>' -m '<body>'
```
Verify the current branch, the empty index, and that the staged list exactly matches the plan. Review the cached diff against the planned commit. If a hook fails or changes files outside the plan, stop. Never bypass hooks. If a message contains a single quote, use `git commit -F -` with a here-string or temp file.

### 4. Push and PR
```bash
git push -u origin <branch>
```
If rejected, stop the push. `git fetch origin`; show `git log --oneline --left-right HEAD...origin/<branch>`; ask. If approved, `git merge --no-edit origin/<branch>`; stop on conflicts; push again only after approval, then re-run checks. Never force.

Create the PR with the approved English title and body:
```bash
gh pr create --base main --head <branch> --title '<title>' --body-file -
```
Pipe the body via a single-quoted here-string or a temp file. If the title or body contains a single quote or `$`, use a temp file. If an open PR exists, verify it before reuse:
```bash
gh pr view <branch> --json state,isDraft,baseRefName,headRefName,headRefOid,isCrossRepository
```
Reuse only if `state` is `OPEN`, `isDraft` is false, base is `main`, head matches the pushed branch, and `headRefOid` matches the pushed commit. Stop on a closed/merged PR. Capture `gh pr view <branch> --json number,url`.

### 5. Checks and Gate 2
```bash
gh pr checks <n> --watch --fail-fast
gh pr checks <n> --json name,state,bucket,link
gh pr view <n> --json state,isDraft,baseRefName,headRefName,headRefOid,mergeable,mergeStateStatus,autoMergeRequest,isCrossRepository
git status --short
git rev-parse HEAD
```
Stop unless: all checks `pass`; `isDraft` false; state `OPEN`; base `main`; head is the branch; `headRefOid` equals local HEAD; `mergeable` `MERGEABLE`; `mergeStateStatus` `CLEAN`; `autoMergeRequest` null; working tree clean. Use a bounded wait (for example, 10 minutes); on timeout, stop. No checks, skipped/cancelled, pending, unknown, merge queue, or auto-merge: stop and ask. If you cannot determine whether a merge queue is in use, ask before using `--delete-branch`. If `isCrossRepository` is true, or another open PR uses this branch as head/base (`gh pr list --head <branch> --state open`, `gh pr list --base <branch> --state open`), ask before `--delete-branch`.

Approved SHA = `headRefOid` presented at Gate 2. Present Gate 2 with PR URL, approved SHA, checks result, merge method, exact command, and whether `--delete-branch` will delete the local and remote branch. Do not merge without explicit approval. Re-run checks immediately before merge; if any check fails or the head changes, stop and re-approve.

### 6. Merge and cleanup
```bash
git status --short
git branch --show-current
git rev-parse HEAD
git worktree list
git rev-list --count origin/<branch>..HEAD
gh pr merge <n> --squash --delete-branch --match-head-commit <sha>
```
Before merging, verify: current branch is the PR head; tree clean; branch tip equals the approved SHA; branch not in another worktree; no local-only commits (`git rev-list --count ...` is `0`). If not, stop; do not let `--delete-branch` discard local work. If merge queue/auto-merge appears, stop and ask; do not use `--delete-branch`.

If the merge exits non-zero, verify:
```bash
gh pr view <n> --json state,mergedAt,headRefOid
```
Continue only if `MERGED` and `headRefOid` equals the approved SHA. Never retry. Verify `MERGED` even on exit 0 with `gh pr view <n> --json state,headRefOid` before cleanup.

Cleanup:
```bash
git fetch --prune
git switch main
git fetch origin main
git merge --ff-only origin/main
git branch --list <branch>
git ls-remote --heads origin <branch>
```
If the local branch remains, `git branch -d <branch>`. After squash, `-d` may fail; that is expected. Use `-D` only under Hard Rule 6. If the remote branch remains, ask before `git push origin --delete <branch>`. A non-zero `git ls-remote` exit is a failure, not proof that the branch is gone.

Git has no `--purge`; the cleanup is `--delete-branch` + `git fetch --prune` + local verification.

### 7. Report
Report PR URL, merge method, branches deleted, pending changes, and one line: what was skipped/not checked and any risk.
