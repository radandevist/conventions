# @radandevist/conventions — design

Date: 2026-11-21 · Owner: Radan · Status: approved by the owner on 2026-11-24

## Intent and why

Radan keeps a growing set of rules: code quality limits, dependency hygiene, lint and format
settings, and how AI agents must work (draft PRs, annotated screenshots, one worktree per agent…).
Today each rule is copied by hand into every project: the owner's private memory, each repo's
`AGENTS.md`, each repo's lint config and CI. A copy is eventually forgotten or goes stale, and the
owner has to repeat himself. Owner, 2026-11-20: « j'ai un tas de règles et préférences et je
voudrais qu'elles soient centralisées et j'en ai marre de répéter tout encore et encore à chaque
projet ».

The model already works for UI rules: they are written once in `@radandevist/radan-ui`
(`docs/rules/`) and PublyApp only links to them. This package applies the same model to every
other kind of rule.

## Goals

1. A rule is written **once**, here, and every consuming project receives it by updating the
   package.
2. A rule ships **with the tool that enforces it** when one exists (CI gate, lint config), so the
   text and the check never drift apart.
3. Each project keeps only what is truly its own: its numbers (baseline, zones), its paths, its
   exceptions.

## Non-goals

- UI rules: they stay in radan-ui (`docs/rules/`). This package links to them, never copies them.
- Project-specific architecture rules (PublyApp's C# standards, route design, module structure):
  they stay in each repo.
  Cross-project architecture rules the owner fixed for every project (TypeScript and .NET) live here,
  under `rules/architecture/` (first one: ports and adapters, 2026-11-24).
- DigitalPrevention and any client project: **never** a consumer (owner decision 2026-11-20). They
  keep their own copies.

## Anti-goals (failure even if it technically works)

- A new rule landing silently and turning every project's CI red at once.
- A rule existing in two places again (here and copied in a repo).
- A package so tied to PublyApp that ShopKartOS cannot adopt it without PublyApp's shape (C# API,
  zones…).

## Owner decisions (2026-11-20)

| # | Decision |
|---|---|
| 1 | One **public** npm package, `@radandevist/conventions`, public GitHub repo `radandevist/conventions`. |
| 2 | It holds three families: (a) code-quality rules **with** their tools, (b) the shared lint/format config, (c) the agents' working rules as text (cross-project architecture rules, such as ports and adapters, ship as text with this family). |
| 3 | UI rules stay in radan-ui. |
| 4 | Consumers: PublyApp, ShopKartOS, radan-ui, future personal projects. Never DigitalPrevention. |
| 5 | Delivery: each project pins an exact version; a new rule = a new version + one bump PR per project. Same as radan-ui. |
| 6 | The lint/format config **moves** from radan-ui to this package; radan-ui keeps the old entry for one minor with a deprecation warning, then removes it. |
| 7 | Each project keeps its own numbers and zones; the package provides the rule and the tool. |

## Contents

### (a) Code quality — rules + tools

Generalised from PublyApp's palier 1 (`AGENTS.md` « Code quality — palier 1 »,
`packages/scripts-ts/src/check-quality-ratchet.ts`, `docs/guides/quality-ratchet.md`):

- **Quality ratchet CLI** (`conventions quality`): runs the bundled `@coderot-ai/cli` (local mode
  only, never `ci` upload mode), aggregates complexity and duplication per 1,000
  SLOC **per zone**, compares with the project's committed baseline, fails if a zone rises beyond
  tolerance, enforces the per-function cognitive-complexity cap on added/modified functions, and
  the PR size cap. Fail-closed on an empty or invalid scan and on a file judged missing; a partial
  scan that still returns valid clones is a known limit. Reads the baseline, thresholds and zones
  at the PR's base SHA.
- **Project config** (in the consuming repo, e.g. `quality.config.json`): zones (globs and
  precedence), baseline values, tolerance, function cap, PR size cap, and `ownerApproved` raise
  entries. The numbers live here, never in the package.
- **Palier text**: the rule as prose (`rules/quality.md`), versioned. A change in a rule's meaning
  is a **major** version; a change of tier or number (e.g. cap 15 → 12) is a config PR in the
  project.
- **pnpm catalog rule**: the « shared dependency goes through a catalog » text and its check
  (generalised from PublyApp's `dependency-health-pin-location.test.ts`), with the override
  carve-out and named catalogs.
- **Languages rule** (optional per project): PublyApp's « C# and TypeScript only » as a text rule
  with a check that a project opts into.

### (b) Lint and format config

Moved from `radan-ui/packages/radan-ui/lint/`: `oxlint.json`, `oxfmt.json`, and the `anti-slop`
oxlint plugin. Exposed as package exports (e.g. `@radandevist/conventions/oxlint`), extended by
each project's own `.oxlintrc.json`.

### (c) Agents' working rules (text)

One markdown file per rule under `rules/agents/` (quality in `rules/quality.md`, CI in `rules/ci.md`,
cross-project architecture under `rules/architecture/`),
plus one index of all rules, written in English for agents, each with a short « why ». First set,
from the owner's existing rules:

- Draft PR from the first commit; PR title in English Conventional Commits; PR/issue numbers shown
  as clickable links.
- One git worktree per agent; never the shared clone; never `docker prune`; kill only exact PIDs.
- Every UI PR carries annotated screenshots (red frames/arrows/numbers, UI in the project's
  language), opened and checked before posting.
- Reviews by a fresh session that did not write the code; no merge with a CRITICAL/MAJOR/MEDIUM
  finding; the owner approves each merge.
- Secrets: never read `.env*`, never print keys/tokens.
- Talking to the owner: simple French, one decision per message with context, example, options and
  a recommendation; recaps as « Fait / En cours / Après ».

The index lists every rule file: agents, quality, catalog, CI and architecture. Projects link that single index
from their `AGENTS.md` through a versioned GitHub URL
(`https://github.com/radandevist/conventions/blob/<tag>/rules/index.md`), so agents read the pinned
version even before install and on GitHub. The `node_modules` path comes in addition, for local use
only.

## Consumption

- Each project: `"@radandevist/conventions": "<exact version>"` as a dev dependency (through the
  pnpm catalog where it applies), a `quality.config.json`, an `.oxlintrc.json` extending
  `@radandevist/conventions/oxlint-react` for React projects or `@radandevist/conventions/oxlint` for
  framework-free TypeScript, and a short block in `AGENTS.md` linking the rule index.
- CI: job steps `conventions quality` (replaces PublyApp's bespoke script) and `conventions catalog`
  (replaces the catalog test removed in migration step 2).
- A new rule: changeset → version PR → publish → one bump PR per project; the bump PR also updates
  the tag in the project's versioned GitHub rule link, shows the new failures, if any, and is merged
  only once green.

## Publishing

Same as radan-ui: changesets, `changesets/action@v2`, npm Trusted Publishing (OIDC, provenance), no
npm token, release on push to `main`. Node ≥ the version PublyApp uses. TypeScript sources run with
Node's type stripping (PublyApp's « TypeScript only » rule), compiled output published for
consumers, with `rules/` in the published `files`.

## Migration order

1. **Package v0.1**: ratchet CLI + catalog check + rule texts, extracted from PublyApp with tests.
2. **PublyApp adopts it**: replace `check-quality-ratchet.ts`, its baseline and the catalog test by
   the package + `quality.config.json` (same numbers); `conventions catalog` runs in CI in place of
   the removed catalog test; `AGENTS.md` sections become links. Must stay green with identical
   measurements.
3. **Lint config moves**: package minor adds `oxlint`/`oxlint-react`/`oxfmt`/`anti-slop`; radan-ui
   re-exports with a deprecation warning for one minor, then removes; PublyApp, a React project,
   adopts `oxlint-react` (absorbs PublyApp issue #2435).
4. **ShopKartOS adopts it** (its own zones and baseline).
5. **radan-ui adopts it** for its own quality and lint.

## Calls made by the author, approved with the design on 2026-11-24

Settled after the builder check of 2026-11-21:

1. **Owner-only controls are not configurable.** The label names (`size-exception`,
   `quality-baseline-exception`) and the rule « only the owner raises a baseline or allows a large
   PR » are constants in the package, never in a file a PR can edit. Each consuming repo protects
   its `quality.config.json` (numbers + `ownerApproved` entries) with CODEOWNERS, as PublyApp does
   today; the CLI reads thresholds and zones at the PR's base SHA, so a PR cannot loosen its own
   gate, and reads only the `ownerApproved` additions made by the PR at the head SHA, which
   CODEOWNERS protects.
2. **C# stays in scope.** The CLI measures every language coderot supports (C# and TypeScript at
   least), because PublyApp must reproduce its baseline exactly. Exclusions (generated files,
   migrations, worktrees, generated clients) and test-file detection are config entries with
   sensible defaults; PublyApp's current patterns become its config.
3. **Catalog check semantics** = PublyApp's AGENTS.md catalog rule (merged with PublyApp issue
   #2447): every dependency (dependencies, devDependencies, optionalDependencies, peerDependencies)
   declared in two or more workspace manifests goes through a pnpm catalog (default or named); an
   override-pinned dependency appears in no catalog and is exact-pinned in each manifest; a
   single-workspace dependency stays local. Exposed as a CLI subcommand (`conventions catalog`),
   not a lint rule.
4. **Lint config is split.** `oxlint` (base, framework-free TypeScript), `oxlint-react` (extends
   base + React settings/rules), `oxfmt`, and the `anti-slop` plugin move. radan-ui's `house`
   plugin: rules that are generic move to the package; rules about radan-ui's own components stay
   in radan-ui. radan-ui's old entry re-exports the new ones for one minor with a warning.
5. **The package owns the coderot pin.** `@coderot-ai/cli` and its whole dependency closure are
   bundled into the package, so a consumer's lockfile cannot resolve a different version of
   coderot or of anything it depends on. A bump anywhere in that closure is a package release,
   which the per-project bump PR then shows.
6. **GitHub-specific is fine.** All consumers are on GitHub. The CLI keeps PublyApp's event handling
   (`pull_request` with labels, `merge_group` without), which needs no switch: a repo without a
   merge queue simply never sends `merge_group`. Local overrides (`QUALITY_BASE`,
   `QUALITY_SIZE_EXCEPTION`) stay.

## Left to the builder

- Exact CLI shape and config schema (keep it minimal; YAGNI).
- How to generalise the zone model so ShopKartOS (TypeScript only) needs no PublyApp concepts.
- Whether the PublyApp-specific checks (CODEOWNERS contract) stay in PublyApp; the label names and
  the owner-only rule remain package constants (choice 1); every repo protects `quality.config.json`
  with CODEOWNERS.
