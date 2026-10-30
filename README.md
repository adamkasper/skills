# Adam Kasper's skills

[Agent Skills](https://agentskills.io) I use every day. They follow the open spec, so they work in Claude Code, Codex, Cursor, Grok Build, Gemini CLI and any other agent that reads `SKILL.md`.

## Install

```bash
pnpx skills add adamkasper/skills
```

The [skills](https://github.com/vercel-labs/skills) CLI asks which skills to install. Pick one with `--skill <name>`, all of them with `--skill='*'`, and add `-g` to install globally.

## Skills

| Skill | Description |
|-------|-------------|
| [composition-over-configuration](skills/composition-over-configuration) | Builds Vue UI from small composable parts instead of one component with `showX` flags and config props |
| [dependency-changelog-review](skills/dependency-changelog-review) | Reviews JS/TS dependency bumps: one read-only subagent per package reads every release in the range, checks your usage and lockfile, and returns a verdict |
| [nuxt-fsd](skills/nuxt-fsd) | [Feature-Sliced Design](https://feature-sliced.design/) architecture for [Nuxt 4+](https://nuxt.com/) projects |
| [split-pr](skills/split-pr) | Splits a large change, branch or PR/MR into small reviewable GitHub PRs or GitLab MRs along CODEOWNERS lines, without touching your checkout |

### composition-over-configuration

Keeps new Vue components from turning into one component with a dozen `showX` props. Based on the talk [Composition Is All You Need](https://www.youtube.com/watch?v=4KvbVq3Eg5w).

- **Structure vs. value props.** A prop that only decides whether a child renders gets deleted, and the parent renders the child or leaves it out. `variant`, `size`, `disabled` and data props stay props.
- **Compound parts.** Dialogs, inputs and composers are built from a root and parts, the way Reka UI and shadcn-vue do it. A slot is for free content in one spot, and an exported part is for pieces with their own props or shared state.
- **Shared state in the root.** Parts get state from `createInjectionState` with an `OrThrow` consumer instead of prop drilling, so a variant swaps what the root provides and reuses every part.
- **Pages and layouts too.** Optional chrome is a wrapper or sibling component. Responsive variants of one tree hide with CSS instead of `v-if` copies that break hydration.
- **Presets, not flags.** An assembly can become a named preset whenever that reads better, and always does once it repeats three or more times. A new variant gets its own preset.
- **New UI only.** It leaves existing kitchen-sink components alone unless you ask.

### dependency-changelog-review

Answers "does this upgrade need a code change, and is it safe to merge?" for every bumped package on your branch, a Renovate or Dependabot PR, or a lockfile update. Works with npm, pnpm, Yarn and Bun, including workspaces and catalogs.

- **One reviewer per package.** Each reads every release note in the version range, scans the commits for unannounced fixes and unreleased regressions, checks security advisories and greps your code for every way you use the package.
- **Lockfile checks.** Each reviewer follows new or changed transitive dependencies, peer ranges, duplicate copies and new install scripts.
- **Model and effort per bump.** Patch bumps get a light review. Majors, pinned packages and your core stack get a deep one. The reviewer stays in the family you run on: Sonnet or Opus on Claude, Sol on GPT, the flagship on Grok, Flash or Pro on Gemini.
- **Pinned packages.** A "do not bump" comment or a Renovate or Dependabot ignore rule puts the package at the top of the report.
- **Read-only.** Reviewers change nothing. You get a verdict per package (Hold, Code change required, Change recommended, No change) and decide what to apply.

### nuxt-fsd

Teaches AI coding agents how to structure Nuxt 4+ projects using Feature-Sliced Design methodology. It covers:

- **Core principle**: FSD lives exclusively in `src/`, while Nuxt `app/` is the runtime shell that consumes it
- **Layer hierarchy**: `pages → widgets → features → entities → shared` (all in `src/`)
- **Thin page pattern**: `app/pages/` contains routing shells that delegate to FSD page slices in `src/pages/`
- **Import rules**: Strict downward-only imports with public API enforcement via `index.ts`, using `~~/src/` prefix
- **Pages-first workflow**: Start in `src/pages/`, extract only when reuse is proven (3+ usages)
- **Widget vs Feature decisions**: Coupled logic+UI (widget) vs standalone logic (feature)
- **TypeScript configuration**: Including `src/` in Nuxt's generated tsconfigs
- **Migration guide**: Adopting FSD in existing Nuxt projects

It activates on architecture decisions in a Nuxt project, questions about where to place new code, creating pages, features, entities or widgets, mentions of FSD, layers or slices, cross-module imports, and code extraction.

### split-pr

Turns one large change into a few small pull requests (GitHub, via `gh`) or merge requests (GitLab, via `glab`) that reviewers can take one at a time. It starts from uncommitted work (even on the default branch), the current branch, or an open PR/MR.

- **Backup first.** Everything, untracked files included, goes into a `refs/backup/pre-split-*` ref before anything moves.
- **Reviewer-aligned slices.** It reads CODEOWNERS with each platform's rules, GitLab sections included, so every slice pulls in as few owner groups as possible. Tests stay with their code, and compatibility shims go into the earlier slice so each slice merges on its own.
- **Independent unless it must stack.** Slices branch off the base, and stack only on a real code dependency. A file that belongs to two slices is split by hunk.
- **Your checkout stays put.** Each slice is built in a temporary worktree. Your branch, index and working tree stay exactly as they were.
- **Verified before pushing.** The slices must add up to the original, with no hunk dropped or duplicated.
- **Asks first.** Nothing is committed, pushed or opened until you approve the plan. PRs/MRs open as drafts, each body links the rest of the split, and the original author is credited when you split someone else's PR.

## License

[MIT](LICENSE)
