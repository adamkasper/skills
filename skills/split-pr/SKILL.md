---
name: split-pr
description: Splits one large body of work (uncommitted changes, a branch, or an open pull request or merge request) into small reviewable pull requests on GitHub (gh) or merge requests on GitLab (glab). Saves a backup ref first, plans reviewer-aligned slices from CODEOWNERS, builds each slice in a temporary worktree without touching the user's checkout, and checks that the slices add up to the original before opening draft PRs/MRs. Use when the user asks to split a PR, MR, branch or set of changes, break up a large PR, cut the number of reviewers or CODEOWNERS groups a change needs, or create stacked PRs or MRs.
---

# Split PR

Turn one large change into a few small PRs (GitHub) or MRs (GitLab) that reviewers can take one at a time. You plan, the user approves, then you build, verify and open them.

## Safety rules

- Create no branch, commit, push or PR/MR until the user has approved the plan.
- The user's branch, index and working tree stay exactly as they are. Build everything in temporary worktrees.
- No destructive git without approval: `reset --hard`, `clean`, branch deletion, force-push, rebasing or amending existing branches. Never close, retarget, force-push or narrow the original PR/MR without asking.
- Stage named paths only, never `git add -A` or `git add .` in a slice. The one exception is the snapshot in step 2, which runs on a throwaway index.
- Keep the backup ref and the original branch. Delete nothing at the end unless the user asks.

Run commands from the repo root. If shell variables do not survive between your commands, write down `$sha`, `$mb` and each worktree path and paste them in literally.

## 1. Platform and base branch

```sh
git remote get-url origin
git ls-remote --symref origin HEAD
git fetch origin
```

- `github.com` means `gh`, `gitlab.com` means `glab`. For another host, use whichever of `gh auth status --hostname <host>` and `glab auth status --hostname <host>` is logged in. If neither or both are, ask the user.
- The default branch is in the first line of `ls-remote`: `ref: refs/heads/<default>`. Never assume `main` or `master`.
- `<target>` is the branch the slices will merge into: the default branch, or the base of the PR/MR being split.

## 2. Snapshot the work

Pick exactly one input.

**An existing PR/MR**, by number or URL. Read its metadata (`gh pr view <n> --json number,url,author,baseRefName,headRefName` or `glab mr view <iid> -F json`, where the base is `target_branch`). Fetch its head without switching branches, since `gh pr checkout` and `glab mr checkout` would move the user's checkout. Local changes play no part.

```sh
git fetch origin pull/<n>/head               # GitHub
git fetch origin merge-requests/<iid>/head   # GitLab
sha=$(git rev-parse FETCH_HEAD)
```

**The current branch with a clean tree:** `sha=$(git rev-parse HEAD)`.

**The user's checkout with staged, unstaged or untracked changes**, even on the default branch. Snapshot all of it on a throwaway index, because `git stash create` would drop the untracked files.

```sh
idx=$(mktemp)
cp "$(git rev-parse --git-path index)" "$idx"
GIT_INDEX_FILE="$idx" git add -A
sha=$(git commit-tree "$(GIT_INDEX_FILE="$idx" git write-tree)" -p HEAD -m "pre-split snapshot")
rm -f "$idx"
```

Then, for every input, save the backup ref and find the merge base:

```sh
git update-ref "refs/backup/pre-split-$(date +%s)" "$sha"
mb=$(git merge-base "origin/<target>" "$sha")
git diff --stat -M "$mb" "$sha"
```

`$sha` is now the single source of truth. The snapshot leaves out gitignored files. Build every slice from `$mb`, not from the tip of `<target>`: copying a whole file from `$sha` onto a newer tip would silently revert whatever landed upstream since. The platform diffs against the merge base anyway.

## 3. Map the owners

Read CODEOWNERS from `origin/<target>`, because the platform uses the target branch's copy. Check the official docs if anything below looks out of date.

- **GitHub** looks in `.github/CODEOWNERS`, then `CODEOWNERS`, then `docs/CODEOWNERS`, and uses the first it finds. It does not support `!` negation or `[ ]` ranges. Code owners are not requested on draft PRs until they are marked ready.
- **GitLab** looks in `CODEOWNERS`, then `docs/CODEOWNERS`, then `.gitlab/CODEOWNERS`, and uses the first it finds. `[Name]` starts a section. Every section applies on its own, so one file can need owners from several sections. `^[Name]` is optional (owners listed, approval not required), `[Name][2]` needs two approvals, `[Name] @owner` sets the section's default owners, and `!path` excludes a path.
- On both, patterns follow most `.gitignore` rules, the last matching pattern wins (within each section on GitLab), and a pattern with no owners leaves the path unowned. GitLab also matches any path without a leading `/` at any depth.

Give every changed path its set of owners. Files with the same owners belong together. Without a CODEOWNERS file, use directories, packages and concerns (refactor, feature, config, dependencies) as the boundaries.

## 4. Plan the slices

- **Independent by default.** Every slice starts at `$mb` and targets `<target>`. Stack a slice on another only when it really needs that code (a new function, type or migration). Foundations go first, consumers after.
- **Each slice stands alone.** It builds, passes CI and can merge by itself. No broken imports or missing symbols in between.
- **Fewest owner groups per slice.** A slice that touches one group needs one review.
- **Tests travel with the code they test.** So do generated files, snapshots and lockfile changes.
- **Shims go first.** If a later slice depends on a rename, put the compatibility alias or re-export in the earlier slice, so every slice can merge alone.
- **Split mixed files by hunk.** Ask the user about a hunk or file that fits no slice, and about untracked files that look accidental (scratch files, local config).
- **Follow the repo's conventions.** Read CONTRIBUTING, AGENTS.md or CLAUDE.md for commit and title format, and the PR/MR template (`.github/pull_request_template.md`, `.github/PULL_REQUEST_TEMPLATE/`, `.gitlab/merge_request_templates/`).
- Aim for two to five slices, each reviewable in one sitting.

Show the plan as a table. Below it, list the branch names, the commit subjects and every untracked file the snapshot picked up.

| # | Title | Files / hunks | Owners | Depends on |
|---|-------|---------------|--------|------------|
| 1 | Add retry helper | `lib/retry.ts`, `lib/retry.test.ts`, `lib/index.ts` (hunk 1 of 2) | @org/platform | none |
| 2 | Retry failed uploads | `app/upload.ts`, `lib/index.ts` (hunk 2 of 2) | @org/web | 1 |

With three or more slices or any stack, add a Mermaid `graph LR` from `<target>` to each slice. Then **stop and ask the user to approve, adjust or cancel.** Do nothing else until you get a clear yes.

## 5. Build each slice

For each slice in plan order, `<slice-base>` is `$mb` for an independent slice, or the parent slice's branch in a stack.

```sh
wt=$(mktemp -d)
git worktree add "$wt" -b <slice-branch> <slice-base>
git -C "$wt" checkout "$sha" -- <added and modified paths, new path of each rename>
git -C "$wt" rm -q -- <deleted paths, old path of each rename>
```

For a file whose hunks go to different slices, take its diff from the slice's own base, delete the hunks of other slices, and apply what is left:

```sh
p=$(mktemp)
git -C "$wt" diff <slice-base> "$sha" -- <path> > "$p"
# edit "$p": delete every hunk that belongs to another slice, from its @@ line up to the next @@
git -C "$wt" apply --index --recount "$p"
rm -f "$p"
```

`--recount` fixes the hunk line counts after editing, so you can also drop single `+` lines or turn `-` lines into context lines. `git add -p` is interactive and does not work for agents. `checkout`, `rm` and `apply --index` all stage, so there is no `git add` step.

```sh
git -C "$wt" status --short     # only the planned paths, all staged
git -C "$wt" commit -m "<subject in the repo's format>"
git worktree remove "$wt"
```

One clean commit per slice. When the repo's fast checks (lint, typecheck, unit tests of the touched package) are cheap, run them in `$wt` before removing it. If a commit hook fails because the temporary worktree has no installed dependencies, tell the user rather than skipping hooks.

## 6. Verify before pushing

Apply each slice's own diff, in plan order, to a detached worktree at `$mb`, then compare it with the snapshot. This works for independent, stacked and mixed plans:

```sh
chk=$(mktemp -d)
git worktree add --detach "$chk" "$mb"
git diff --binary <slice-base> <slice-branch> | git -C "$chk" apply --index   # once per slice
git -C "$chk" diff --cached --stat "$sha"                                     # must print nothing
git worktree remove --force "$chk"
```

For a pure stack, this shorter check is enough:

```sh
git diff --stat <last-slice-branch> "$sha"    # must print nothing
```

Plain `apply` is strict on purpose. A hunk that went into two slices fails the second time, where a cherry-pick would merge it silently. A dropped hunk shows up in the final diff. If an apply fails, the slices overlap: stack them, merge them, or fix the split. `--force` is needed because the check worktree holds staged changes, and it removes only that throwaway. If any check is not empty, stop and report it.

## 7. Push and open drafts

Check that the user can push to origin: `gh repo view --json viewerPermission --jq .viewerPermission` must be `WRITE`, `MAINTAIN` or `ADMIN`. On GitLab, `glab api projects/:fullpath | jq .permissions` must show an `access_level` of 30 (Developer) or more in `project_access` or `group_access`. Otherwise push to the user's fork (find it in `git remote -v`, or ask) and add `--head <fork-owner>:<branch>` to gh, or `--head <fork-namespace>/<project>` to glab. A stack needs every branch in the target repo, so from a fork use independent slices only, or ask.

```sh
git push origin <slice-branch>
gh pr create --draft --base <slice-target> --head <slice-branch> --title "<title>" --body-file <body.md>
glab mr create --draft --target-branch <slice-target> --source-branch <slice-branch> --title "<title>" --description-file <body.md> --yes
```

`<slice-target>` is `<target>`, or the parent slice's branch in a stack. Use long flags: gh's `-b` is the body but glab's `-b` is the target branch, and glab's `-d` is the description, not draft. Flags change between versions, so check `gh pr create --help` and `glab mr create --help` first.

- **Body.** Fill in the repo's template. Say what the slice does, how to test it, and list every PR/MR of the split with its order and dependencies. Once all exist, put the real links into every body with `gh pr edit <n> --body-file <body.md>` or `glab mr update <iid> --description-file <body.md>`.
- **Credit.** If the current user (`gh api user --jq .login`, `glab api user | jq -r .username`) is not the author of the PR/MR being split, start each body with "Original changes by @<author> in #<n>" (`!<iid>` on GitLab).
- **The original PR/MR** stays open and unchanged. Ask whether to close it, mark it draft or link the new ones.

## 8. Stacks after a merge

- **GitHub** retargets open PRs in the same repository to the merged PR's base when the merged PR's head branch is deleted, by hand or by the "automatically delete head branches" setting. Without the deletion, nothing is retargeted.
- **GitLab** retargets up to four open MRs that target the merged MR's source branch, at the moment it merges. Deleting the branch later does not retarget.
- After a squash or rebase merge, the child branch still carries the parent's original commit. Rebase it onto the target in a temporary worktree and, with the user's approval, force-push it:

```sh
git fetch origin
wt=$(mktemp -d)
git worktree add "$wt" <child-branch>
git -C "$wt" rebase --onto "origin/<target>" <old-parent-slice-commit>
git push --force-with-lease origin <child-branch>
git worktree remove "$wt"
```

Then retarget it if the platform did not: `gh pr edit <n> --base <target>` or `glab mr update <iid> --target-branch <target>`. Stack tools exist (GitHub's `gh stack` extension, `github/gh-stack`, and the experimental `glab stack`), but use them only when the user already does.

## 9. Report

- Each PR/MR: title, URL, branch, target and slice commit SHA (needed for the rebase in step 8).
- The backup ref name and `$sha`.
- What is left: the original branch and working tree (unchanged), the original PR/MR, any path that went into no slice, and any check you skipped.
