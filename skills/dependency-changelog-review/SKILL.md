---
name: dependency-changelog-review
description: Reviews JavaScript and TypeScript dependency bumps (npm, pnpm, Yarn, Bun) by spawning one read-only reviewer subagent per bumped package; each reads every release in the range, checks how the repo uses the package and what the lockfile pulls in, and returns a verdict. Use when the user asks whether upgraded dependencies, a Renovate or Dependabot PR, or a lockfile update need code changes or are safe to merge.
---

# Dependency changelog review

For every bumped package, answer two questions: does our code have to change, and is the bump safe to merge? Reviewer subagents gather the evidence in parallel; you verify what they claim and write the report.

## 1. Collect the bumps

**Base.** The branch, commit or PR/MR the user names; otherwise the merge base with the default branch:

```sh
git fetch origin
git merge-base HEAD origin/HEAD   # origin/HEAD unset: take the default branch from `git remote show origin`
git diff <base> --name-only
```

Diff the working tree against `<base>`: uncommitted changes count, and whatever the default branch gained since the branch was cut stays out. Pass `<base>` to every reviewer.

**What counts as a bump.** Diff each changed file of these kinds and list every change as `name old → new`:

- `package.json` files: all dependency fields, `overrides` / `resolutions` / `pnpm.overrides`, `packageManager`, `engines`, `devEngines`.
- Catalogs and workspace config: `pnpm-workspace.yaml`, `.yarnrc.yml`, `catalog` / `catalogs` in the root `package.json`.
- Runtime pins: `.nvmrc`, `.node-version`, `.tool-versions`, `FROM node:` in Dockerfiles, `node-version` in CI.
- Versioned CLI calls in scripts and CI: `npx tool@1.2.3`, `pnpm dlx`, `yarn dlx`, `bunx`.
- Only the lockfile changed (lockfile maintenance, in-range updates): read the bumps from its direct-dependency entries (pnpm `importers`, npm `packages[""]` and workspace entries, the top level of `yarn.lock` / `bun.lock`).

`packageManager`, `engines` and `devEngines` must agree after the bump; report a mismatch.

**Group the bumps into reviewers.**

- **Pinned**: a comment next to the version says not to bump it, or a Renovate rule (`allowedVersions`, `ignoreDeps`, `enabled: false`) or a Dependabot `ignore` entry covers it. It goes to the top of the report, whatever its reviewer concludes.
- **Peer partners** (`eslint-plugin-oxlint` ↔ `oxlint`): one reviewer each, told about the other's bump.
- **Release trains** (one repo, one version line: `@tiptap/*`, `@vueuse/*`): one reviewer for the train, a verdict per package.
- **Added**: peer dependencies and lockfile impact only. **Removed**: confirm nothing references it any more. **Downgrade**: read the notes of the versions being dropped. **Node and the package manager**: release notes only, no commit scan or usage search.

**Pick a depth per reviewer.**

| Depth | For |
|---|---|
| light | Patch bumps, `@types/*`, dev tooling whose changes miss our config. Up to five light bumps may share one reviewer. |
| standard | Everything else. |
| deep | Majors, `0.x` minors, pinned packages, the core stack, security-sensitive packages (auth, crypto, sanitising, HTTP clients), ranges of more than ten releases, packages without usable release notes. |

The core stack is what the apps are built on: framework, router, UI library, build tool, TypeScript. When `AGENTS.md` or `CLAUDE.md` lists it, use that list.

## 2. Spawn the reviewers

Pick model and effort from the family you are running on:

| You run on | light | standard | deep |
|---|---|---|---|
| Claude | Sonnet · `medium` | Sonnet · `high` | Opus · `xhigh` |
| GPT | Sol · `low` | Sol · `medium` | Sol · `xhigh` |
| Grok | flagship · `medium` | flagship · `high` | flagship · `xhigh` |
| Gemini | Flash · `low` | Flash · `medium` | Pro · `high` |
| anything else | mid tier · one step below default | mid tier · default | strongest tier · highest below `max` |

- Take the newest version of that tier your subagent tool offers, under the exact identifier it accepts: an alias (`sonnet`), a model ID, or a slug with the effort built in (`…-sonnet-…-high`). Stay in your own family even when the tool offers others; a model the user names overrides the table.
- When the model lacks that effort level, take the nearest one it supports. When the tool has no model or effort field, or rejects the value, spawn without it and let the reviewer inherit your settings.
- Use a general-purpose subagent type: reviewers need a shell and network for `git`, `npm` and `gh`.
- Spawn all reviewers in one message so they run in parallel. If your harness caps concurrent subagents, send the deep ones first, then the rest in waves.
- Without subagents, review the packages yourself one at a time, with the prompt below as your checklist, and keep only each answer block.

Fill in this prompt for each reviewer:

```text
You review one dependency bump in <repo root>. Other agents review the other bumps; stay on yours.

Project: <one line from step 1: package manager, workspace layout, main frameworks>
Base: <base>   Lockfile: <path>
PACKAGE: `<name>` <old> → <new>   Depth: <light | standard | deep>
<Notes from step 1: pin and its reason, peer partners and their bumps, release-train siblings. Re-verify them.>

Read-only: edit nothing, install nothing, run no build, lint or typecheck. git diff/show/log, npm view, npm diff, gh/glab api, curl and `<package manager> why` are fine.

1. Release notes. Read every version in (<old>, <new>]. List them with `npm view <name> versions --json`, skipping prereleases unless <old> or <new> is one. Find the source with `npm view <name> repository --json` (`directory` points into a monorepo). Fetch notes in bulk: `gh api repos/<owner>/<repo>/releases --paginate --jq '.[] | "## \(.tag_name)\n\(.body)"'`, the CHANGELOG.md at the <new> tag (the package's own in a monorepo), or the published one at https://cdn.jsdelivr.net/npm/<name>@<new>/CHANGELOG.md. For a major, read the migration guide too. For every breaking change, deprecation, changed default and fix, decide whether it touches our usage.
2. Commits. Scan the subjects between the two tags: `gh api repos/<owner>/<repo>/compare/<old-tag>...<new-tag> --jq '.commits[].commit.message | split("\n")[0]'`; in a monorepo, `gh api 'repos/<owner>/<repo>/commits?sha=<new-tag>&path=<package dir>' --paginate` back to <old-tag>. This catches fixes the notes leave out, including security fixes merged as "Merge commit from fork". Open a diff only when a subject calls for it.
3. Published code, for depth deep or when the notes are thin: `npm diff --diff=<name>@<old> --diff=<name>@<new> --diff-name-only`, then rerun without `--diff-name-only`, passing the paths of the entry points and type declarations we use.
4. Registry and advisories: `gh api 'advisories?ecosystem=npm&affects=<name>@<old>'` for what the bump fixes, the same with <new> for what stays open (`[]` with exit code 0 means none); `npm view <name>@<new> deprecated`; the publish date of <new> in `npm view <name> time --json`.
5. Our usage. Grep the repo, skipping lockfiles, node_modules and build output, for every way we reach the package: static, dynamic, type-only and subpath imports; registrations in framework, build, lint and test configs; CSS imports; CLI calls in scripts, CI and Dockerfiles; auto-imported APIs, which appear by export name rather than package name.
6. Lockfile. Compare this package's entries at <base> (`git show <base>:<lockfile>`) and in the working tree, then follow its dependencies: new or changed transitive packages, peer ranges we must satisfy, duplicate copies, new install scripts. Grep for the names you find; leave the full lockfile diff unread.

Verdicts:
- Hold: do not merge the bump as is (a regression or advisory that hits us, a peer range we cannot meet, a pin whose reason still holds).
- Code change required: the bump is fine once our code or config changes.
- Change recommended: works as is; a deprecation, new default or new option is worth adopting.
- No change.

Answer briefly, in this shape:
- Verdict
- Evidence: version, quoted line, link, for each item behind the verdict
- Our code: path:line and the concrete change
- Lockfile: new or changed transitive packages, duplicates, peer conflicts
- Security: advisories fixed or still open, deprecation, install scripts, <new> published less than 3 days ago
- Manual tests worth doing
- Not verified: what you skipped or could not reach
```

## 3. Synthesize

Wait for every reviewer. Then:

1. Read the code behind every Hold, Code change required and Change recommended yourself. Fetch the cited changelog entry only when the quote looks doubtful.
2. Cross-check: peer partners and release trains moved together and their ranges still hold; settle conflicting lockfile claims with a grep of the lockfile diff.
3. Report the overall verdict first (the worst verdict wins), then a table (package, `old → new`, depth, verdict), then only what needs action: code changes, pinned packages, packages that must move together, duplicates, security notes, manual tests, open uncertainties.

Change nothing without asking. Once the user agrees, follow the repo's rules in `AGENTS.md` / `CLAUDE.md` (feature flags, type-check baselines and the like). If a bump changes user-visible behaviour you cannot preserve, raise it instead of shipping it.
