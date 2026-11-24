# CI rules (fast, honest CI)

Rules for every personal project (PublyApp, ShopKartOS, radan-ui, future ones). Never DigitalPrevention.
Written for agents; each rule says why. Source: the CI work of 2026-11-21 → 23 (radandevist/conventions#2,
radandevist/shopkart-os#331, radandevist/radan-ui#88) and PublyApp's CI lessons.

**What is shared and what is not.** Shared tools from this package (`conventions quality`,
`conventions catalog`, see the [design](../docs/design/2026-11-21-conventions-design.md)) replace a
project's own copy of the same check. Everything else stays project-owned: workflow files, deploy and
selftest scripts, the numbers (time budgets, PR size cap) and which checks are required.

Measured results:

| Project | Before | After | What did it |
|---|---|---|---|
| ShopKartOS `pnpm verify` (local gate, no GitHub CI) | 3.9 min | ~37 s (45 s under load) | bounded parallel pool of independent steps; the browser layout check (115 s → ~32 s, the longest step) in a bounded browser pool; faster deploy selftest |
| radan-ui PR CI | 118 s p50 / 310 s p95 | 60–71 s (target below not reached yet) | independent jobs instead of `needs:` chains; cancel superseded PR runs; slow native job split in a matrix; cached native install; heavy nav-stress only when its paths change |

## 1. Two gates, two budgets

- **PR gate**: the checks that run on every push of a pull request. **Target p50 under 1 minute;
  ceiling: p95 under 5 minutes** (a p50 alone hides slow runs). Why: a slow gate gets skipped, and a
  skipped gate is how main goes red.
- **Merge gate**: the checks that must be green on the **exact head SHA** before a merge. It is the PR
  gate plus every heavy check whose inputs the change touches (see §3, affected-only). It may take
  longer than the PR gate; its list is written down in each project.
- **Nightly**: heavy full runs (e.g. radan-ui's full nav-stress, ~10 min) also run on main every night.
  A nightly run is **not** a merge gate: anything a change can break must be in the merge gate through
  its path filter. A red nightly blocks the next merges until it is fixed.

## 2. Never buy speed with honesty

- **Measure before touching.** Record p50 and p95 over several real runs before any change; every CI PR
  states its before/after p50/p95 and links the runs or the timing log. Why: guesses about where time
  goes are usually wrong (ShopKartOS: the layout check was the longest step, not the unit tests).
- **No automatic retries, no widened timeouts, no `continue-on-error` on a real check.** They hide
  flakes instead of fixing them. A flaky check is a bug to fix, or a written, project-owned exception.
- **Never drop a check to go faster.** Make it faster, or run it only when its inputs change (§3).
  Same cases before and after.
- **The merge gate runs on the last commit.** Targeted runs are fine while working; a merge needs the
  full merge gate green on the exact head SHA. Any rebase or new commit invalidates an earlier run
  (ShopKartOS #344 broke main when this was skipped).

## 3. How to make it fast

- **Run independent steps in parallel, in a bounded pool** (`min(4, cores)` by default, overridable by
  env). Steps that share a resource (database file, build output, port, browser profile) declare it and
  take a read or write lock: **isolation groups**, never "hope they don't collide".
- **Stop at the first failure** as soon as a step exits non-zero; kill the rest; print which step failed
  and where its log is. Logs per step in a known folder; one header line per step (id, status, ms).
- **Independent CI jobs, not chains.** Drop `needs:` unless a job truly consumes another's output.
- **Cancel superseded runs on pull requests only.** Use one group per PR and a unique group per run
  elsewhere, so a queued run on main is never cancelled:
  ```yaml
  concurrency:
    group: ${{ github.workflow }}-${{ github.event_name == 'pull_request' && github.ref || github.run_id }}
    cancel-in-progress: ${{ github.event_name == 'pull_request' }}
  ```
- **Cache what costs, keyed exactly.** pnpm store; slow native installs keyed on lockfile + manifest +
  runtime version, no loose restore keys. Keep a cache only if the measured restore beats the install.
- **Affected-only, with fall-back to full.** Heavy suites (browser stress, native builds) run when files
  in their input paths change (path filters), and in full on main. When in doubt about what a change
  affects, run everything.
- **A path-filtered job is never a required check by itself.** GitHub leaves a required check that did
  not run as "Expected — waiting", and the PR cannot merge. Either keep it out of the required list
  (radan-ui: only `check` is required), or put it behind an always-running aggregator job with the
  required name that passes when no path matched and runs the suite otherwise.
- **Split a slow job** into a matrix when its parts are independent; shard tests only with a coverage
  guard that uses the test tool's own rule (never a hand-written list that can drop files).
- **Use the tool's own rules, never re-implement them** (lint rules, test selection, coverage).
- **Re-measure the runtime choice, don't assume it.** Example: Bun was trialled on ShopKartOS (2026-11-23,
  Bun 1.4.5) for test/format/lint while keeping pnpm as package manager: not adopted (156 failing tests
  on missing Vitest APIs, ~1 s total gain). Re-trial when its gaps close.

## 4. Where CI runs

- **No CI runner on the owner's PC**, and no paid runner without a quote the owner accepted.
- **A project without GitHub CI** (ShopKartOS today) uses its local full gate as the merge gate. The
  agent that ran it states the tested SHA and the result in the PR (comment) before asking to merge;
  the merger checks that SHA equals the PR head. This is an honor system until the project has CI.

## 5. Size and review

- CI changes ship as small PRs, one change each, with numbers, within the project's own PR size cap and
  its exclusions (e.g. PublyApp: 600 changed lines, excluding tests, generated files and lockfiles).
- Reviews follow the project's merge gate: no merge while a CRITICAL, MAJOR or MEDIUM finding stands.
