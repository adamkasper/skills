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
| [dependency-changelog-review](skills/dependency-changelog-review) | Reviews JS/TS dependency bumps: one read-only subagent per package reads every release in the range, checks your usage and lockfile, and returns a verdict |
| [nuxt-fsd](skills/nuxt-fsd) | [Feature-Sliced Design](https://feature-sliced.design/) architecture for [Nuxt 4+](https://nuxt.com/) projects |

### dependency-changelog-review

Answers "does this upgrade need a code change, and is it safe to merge?" for every bumped package on your branch, a Renovate or Dependabot PR, or a lockfile update. Works with npm, pnpm, Yarn and Bun, including workspaces and catalogs.

- **One reviewer per package**: each reads every release note in the version range, scans the commits for unannounced fixes, checks security advisories and greps your code for every way you use the package.
- **Lockfile aware**: follows new or changed transitive dependencies, peer ranges, duplicate copies and new install scripts.
- **Model and effort per bump**: patch bumps get a light pass, majors, pinned packages and your core stack a deep one. The reviewer stays in the family you run on: Sonnet or Opus on Claude, Sol on GPT, the flagship on Grok, Flash or Pro on Gemini.
- **Respects pins**: a "do not bump" comment or a Renovate/Dependabot ignore rule puts the package at the top of the report.
- **Read-only**: reviewers change nothing; you get a verdict per package (Hold, Code change required, Change recommended, No change) and decide what to apply.

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

## License

MIT
