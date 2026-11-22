---
name: dependency-changelog-review
description: Reviews JavaScript and TypeScript dependency bumps (npm, pnpm, Yarn, Bun) by spawning one read-only reviewer subagent per bumped package. Each reviewer reads every release in the range, checks how the repo uses the package and what the lockfile pulls in, and returns a verdict. Use when the user asks whether upgraded dependencies, a Renovate or Dependabot PR, or a lockfile update need code changes or are safe to merge.
---

# Dependency changelog review

For every bumped package, answer two questions: does our code have to change, and is the bump safe to merge? Reviewer subagents gather the evidence in parallel. You verify what they claim and write the report.

## 1. Collect the bumps

**Base.** Use the branch, commit or PR/MR the user names. Otherwise use the merge base with the default branch:

```sh
git fetch origin
git merge-base HEAD origin/HEAD   # if origin/HEAD is unset, read the default branch from `git remote show origin`
git diff <base> --name-only
```

Diff the working tree against `<base>`, so uncommitted changes count and commits that landed on the default branch after you branched off stay out. Pass `<base>` to every reviewer.

**What counts as a bump.** Diff each changed file of these kinds and list every change as `name old → new`:

- `package.json` files: all dependency fields, `overrides` / `resolutions` / `pnpm.overrides`, `packageManager`, `engines`, `devEngines`.
- Catalogs and workspace config: `pnpm-workspace.yaml`, `.yarnrc.yml`, `catalog` / `catalogs` in the root `package.json`.
- Runtime pins: `.nvmrc`, `.node-version`, `.tool-versions`, `FROM node:` in Dockerfiles, `node-version` in CI.
- Versioned CLI calls in scripts and CI: `npx tool@1.2.3`, `pnpm dlx`, `yarn dlx`, `bunx`.

If only the lockfile changed (lockfile maintenance, in-range updates), read the bumps from its direct-dependency entries: pnpm `importers`, npm `packages[""]` and workspace entries, or the top level of `yarn.lock` and `bun.lock`.

`packageManager`, `engines` and `devEngines` must agree after the bump. Report a mismatch.

**Group the bumps into reviewers.**

- **Pinned.** A comment next to the version says not to bump it, or a Renovate rule (`allowedVersions`, `ignoreDeps`, `enabled: false`) or a Dependabot `ignore` entry covers it. Put it at the top of the report, whatever its reviewer concludes.
- **Peer partners.** Packages linked by a peer range, such as `eslint-plugin-oxlint` and `oxlint`, get one reviewer each. Tell each reviewer about the other bump.
- **Release trains.** Packages from one repo on one version line, such as `@tiptap/*` or `@vueuse/*`, share one reviewer that gives a verdict per package.
- **Narrower reviews.** For an added package, check only its peer dependencies and lockfile impact. For a removed one, confirm nothing references it any more. For a downgrade, read the notes of the versions you give up. For Node and the package manager, read the release notes only, with no commit scan or usage search.

**Pick a depth per reviewer.**

| Depth | For |
|---|---|
| light | Patch bumps outside the core stack and security-sensitive packages, `@types/*` bumps, and dev tooling whose changes do not touch our config. Up to five light bumps may share one reviewer. |
| standard | Patch bumps of the core stack or security-sensitive packages, and everything else not listed as light or deep. |
| deep | Majors, `0.x` minors, non-patch bumps of the core stack or security-sensitive packages, pinned packages, ranges of more than ten releases, packages with an advisory or security fix in the range, and packages without usable release notes. |

When a bump matches more than one row, pick the deepest one. The core stack is the framework, router, UI library, build tool and TypeScript that the apps run on; use the list in `AGENTS.md` or `CLAUDE.md` when one exists. Security-sensitive packages handle auth, crypto, sanitising or HTTP.

## 2. Spawn the reviewers

Pick the reviewer's model and effort (or thinking level, such as `thought_level` or `reasoning_effort`) from the family you run on:

| You run on | light | standard | deep |
|---|---|---|---|
| Claude | Sonnet, `low` | Sonnet, `high` | Opus, `xhigh` |
| GPT | Sol, `low` | Sol, `medium` | Sol, `xhigh` |
| Grok | flagship, `low` | flagship, `high` | flagship, `xhigh` |
| Gemini | Flash, `low` | Flash, `medium` | Pro, `high` |
| anything else | mid tier, lowest level above off | mid tier, default | strongest tier, highest level below `max` |

- Stay in your own family and generation (your model's version number, such as Claude 5.5 or Gemini 4), even when the tool offers other families or older generations. Skip any tier alias that resolves to an older generation.
- When a tier from the table does not exist in your generation, pick the cheapest model of your generation that the tool offers for `light` and `standard` (or the strongest one for `deep`), or inherit your own model and pass the level from the table.
- Use the exact identifier your subagent tool accepts: an alias, a model ID, or a slug that encodes the level (such as `claude-sonnet-5-5-low`). A model the user names overrides the table.
- Map the level in the table to the parameter your tool uses (`effort`, `thought_level`, `reasoning_effort`, or a suffixed model slug). Give `light` reviewers the lowest sensible non-off level instead of leaving them at a higher parent default. When the model lacks that level, pick the nearest one it supports. When the tool has no model or level parameter, or rejects a value, omit only that parameter so the reviewer inherits it while keeping the other setting.
- Use a general-purpose subagent type, because reviewers need a shell and network access for `git`, `npm` and `gh`.
- Spawn all reviewers in one message so they run in parallel. If you can run only a few subagents at once, send the deep ones first and the rest in later batches.
- If you cannot spawn subagents, review the packages yourself one at a time. Use the prompt below as your checklist and keep only the answer for each package.

Fill in this prompt for each reviewer:

```text
You review the dependency bump or bumps listed below in <repo root>. Other agents review the rest, so review only yours.

Project: <one line from step 1: package manager, workspace layout, main frameworks>
Base: <base>   Lockfile: <path>
PACKAGE: `<name>` <old> → <new>   Depth: <light | standard | deep>
<Notes from step 1: pin and its reason, peer partners and their bumps, release-train siblings, or narrower-review scope. Re-verify them.>

Read-only: edit nothing, install nothing, run no build, lint or typecheck. git diff/show/log, npm view, npm diff, gh/glab api, curl and `<package manager> why` are fine. If npm refuses to run inside the repo because of an engine check, run `npm view` and `npm diff` from a temp directory.

1. Release notes. Read every version after <old> up to and including <new>. List them with `npm view <name> versions --json`, and skip prereleases unless <old> or <new> is one. Find the source with `npm view <name> repository --json`; in a monorepo, its `directory` field gives the package's folder. Fetch the notes in bulk with `gh api repos/<owner>/<repo>/releases --paginate --jq '.[] | "## \(.tag_name)\n\(.body)"'`, from the CHANGELOG.md at the <new> tag (the package's own in a monorepo), or from the published one at https://cdn.jsdelivr.net/npm/<name>@<new>/CHANGELOG.md. For a major, read the migration guide too. For the SDK of a hosted service (payments, auth, analytics, maps), also read the vendor's own changelog, because the package notes can leave out the API or script version the package now targets. For every breaking change, deprecation, changed default and fix, decide whether it affects how we use the package.
2. Commits. Scan the commit subjects between the two tags with `gh api repos/<owner>/<repo>/compare/<old-tag>...<new-tag> --jq '.commits[].commit.message | split("\n")[0]'`. In a monorepo, page through `gh api 'repos/<owner>/<repo>/commits?sha=<new-tag>&path=<package dir>' --paginate` back to <old-tag> instead. This catches fixes the notes leave out, including security fixes merged as "Merge commit from fork". Then check what came after <new>: commit subjects from <new-tag> to `HEAD` and issues that mention <new> (`gh search issues --repo <owner>/<repo> "<new>"`) show regressions whose fix is not released yet. Open a diff only when a subject needs a closer look.
3. Published code. For depth deep, when the notes are thin, or when the range contains a security fix or advisory, run `npm diff --diff=<name>@<old> --diff=<name>@<new> --diff-name-only`. Then run it again without `--diff-name-only` and pass the paths of the entry points and type declarations we use.
4. Registry and advisories. `gh api 'advisories?ecosystem=npm&affects=<name>@<old>'` lists what the bump fixes, and the same query with <new> lists what stays open. An empty `[]` with exit code 0 means there are none. Check `npm view <name>@<new> deprecated`, and read the publish date of <new> from `npm view <name> time --json`.
5. Our usage. Grep the repo for every way we use the package, skipping lockfiles, node_modules and build output. Look for static, dynamic, type-only and subpath imports, registrations in framework, build, lint and test configs, CSS imports, and CLI calls in scripts, CI and Dockerfiles. Auto-imported APIs appear by export name, not package name, so grep for those names too. For a hosted-service SDK, also find the same vendor's server-side SDK elsewhere in the repo, because its API version must match.
6. Lockfile. Compare this package's entries at <base> (`git show <base>:<lockfile>`) and in the working tree, then follow its dependencies. Look for new or changed transitive packages, peer ranges we must satisfy (including optional peers that other packages declare on this one), duplicate copies and new install scripts. Grep for the names you find instead of reading the whole lockfile diff.

Verdicts:
- Hold: do not merge the bump as is. Reasons include a regression or advisory that affects us, a peer range we cannot meet, or a pin whose reason still holds.
- Code change required: the bump is fine once our code or config changes.
- Change recommended: the bump works as is, but a deprecation, new default or new option is worth adopting.
- No change.

Answer briefly, in this shape:
- Verdict
- Evidence: version, quoted line and link for each item behind the verdict
- Our code: path:line and the concrete change
- Lockfile: new or changed transitive packages, duplicates, peer conflicts
- Security: advisories fixed or still open, deprecation, install scripts, and whether <new> is less than 3 days old
- Manual tests worth doing
- Not verified: what you skipped or could not reach
```

## 3. Synthesize

Wait for every reviewer. Then:

1. Read the code behind every Hold, Code change required and Change recommended yourself. Fetch the cited changelog entry only when the quote looks doubtful.
2. Check that peer partners and release trains moved together and that their ranges still hold. When reviewers disagree about the lockfile, grep the lockfile diff to settle it.
3. Report the overall verdict first. The worst verdict of any package wins. Then add a table with package, `old → new`, depth and verdict. After the table, list only what needs action: code changes, pinned packages, packages that must move together, duplicates, security notes, manual tests and open uncertainties.

Change nothing without asking. Once the user agrees, follow the repo's rules in `AGENTS.md` or `CLAUDE.md`, such as feature flags or type-check baselines. If a bump changes user-visible behaviour you cannot preserve, raise it instead of shipping it.
