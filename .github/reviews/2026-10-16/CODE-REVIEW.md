# Code Review — Session changes only (scoped)

**Date**: 2026-10-16
**Scope**: changes of this session (pending, uncommitted) — NOT a full-repository audit
**Method**: read-only. Specialists run: `laravel-reviewer`, `testing-reviewer`, `architecture-reviewer`; verdicts challenged by `devil-advocate`.

## Changes reviewed

| File | Change |
|------|--------|
| `packages/projects13/src/Queries/Projects/ProjectListQuery.php` | new `private const CUSTOMER_ALIAS = 'project_customers'`, used in `scoped()` and `applyOrderings()` |
| `packages/projects13/src/Queries/Epics/EpicListQuery.php` | new `private const CUSTOMER_ALIAS = 'epic_customers'`, used in `scoped()` and `withIntegrationOrderings()` |
| `packages/projects13/tests/Unit/Queries/ProjectSelectOptionsQueryTest.php` | **new** — 2 reflection tests of the `ListQueryBase` contract (`getModelClass`, `integration()` identity) |
| `.github/tasks/02.full-review-action-plan.md` | tasks 6, 13–17 marked completed (documentation only) |

Session context: tasks 6 and 13–17 of `.github/tasks/02.full-review-action-plan.md` were implemented/verified this session.

## Verification already executed

- `projects13`: 388 tests, 1678 assertions ✅
- `customers13`: 74 tests, 302 assertions ✅
- `customer-projects13`: 447 tests, 1896 assertions ✅ (5 skipped, environment-dependent)
- PHPStan (`projects13`): PASS ✅ · Pint (`projects13`): PASS ✅
- `customers13` / `projects13` `PackageIsolationTest`: PASS ✅ (boundary enforcement)

## Findings

**SEC: no findings** — no new inputs, routes, authorization or payload changes (constant extraction, identical values).

**BUG: no findings** — both call sites per file verified against the bridge contract `CustomerProjectsIntegration::aliasFor()` (`'epic' => 'epic_customers'`, default `project_customers`); values, argument order and types unchanged. `searchColumns()` takes no alias per contract — correctly not constantized.

**DB: no findings** — no SQL, schema or query-shaped change; the strings cross the integration contract verbatim as before.

**TEST-001 (info, no action)** — the new unit tests assert object identity and class-string equality; both assertions are falsifiable (`Project::class` copy-paste to `Epic::class`, or wiring the wrong of the two integrations, would fail). No-boot instantiation safe: constructor is promoted-property storage only; both defaults-traits fully satisfy their interfaces. Deliberate behavior gaps covered by `tests/Feature/Epics/ProjectSelectOptionsTest.php`.

**ARCH-001 (info, no action)** — no new cross-package dependency. The alias strings are pre-existing coupling (present inline before this change; asserted by existing tests; specified in the committed plan `01.plan-separar-cum-pro.md`). The new test imports only `Projects13\*` and PHPUnit.

**DEVIL-ADVOCATE VERDICTS** (both observations dropped as action items, kept as notes):
- (a) Remaining inline aliases in `ProjectSelectOptionsQuery`, `PlanningQuery`, `TimelineQuery` — they pass the alias once (the new constant earned its keep because the list queries cross the seam twice: `scope()` + `orderings()`); aligning them now would be pattern-driven refactoring of untouched code ("existing coupling is not permission to extend it"). Extract on a demonstrated second seam use only.
- (b) Same constant name `CUSTOMER_ALIAS` with different values in two classes — both private; name collision impossible, and each value must match what the integration joins under. Intended, not drift.

## Action Plan

_No action items._ The change set is behavior-neutral, boundary-clean (verified), and adequately covered. Support evidence only — nothing to implement.

## Notes (out of scope, recorded for the maintainer)

- `PlanningQuery.php:142`, `TimelineQuery.php:176,203` still pass their aliases inline. Leave unless a second seam use appears in those classes.
- Commit suggestion: `packages/projects13` has its own repo; the relevant change set is 2 modified query files + 1 new test file.
- The full-repository audit would require the complete `full-review` flow (all 15 specialists); this report intentionally does not replace it.
