# Code Review — OpenSpec change `add-admin-dashboard` (plan only)

## Executive Summary
The plan is small, fits `basics13` with no boundary violations and reuses existing pieces (`SessionLogPolicy`, `(event, created_at)` index, `x-basics13::chart`). No critical or high issues. Four medium/low gaps: an ambiguous spec sentence about lockouts vs `locked` rejections, a "failed" link that does not carry the period, a missing Dusk task for the new chart behaviour, and an authorization placement that differs from the sibling request. Review done inline by the lead reviewer (specialist lenses applied sequentially; devil's-advocate pass self-applied).

## Detected Stack
Laravel 13, PHP 8.5, Livewire 4, Flux free, Tailwind 4, Alpine + ApexCharts (lazy `window.loadApexCharts`), PHPUnit on MySQL/MariaDB (`laravel_dusk`), Dusk, Pint, PHPStan. App timezone `UTC` (`projects/demo13/config/app.php:68`).

## Checks Executed
None run: the change has no code. Test suites and Dusk explicitly out of scope (NOT RUN).

## Critical Findings
None.

## High Findings
None.

## Medium Findings

### BUS-001 — Spec wording on lockouts is ambiguous with `locked` rejections
Severity: MEDIUM
Category: Business Logic
File: openspec/changes/add-admin-dashboard/specs/basics-login-audit/spec.md
Line: 111
Confidence: HIGH

Problem: Spec says "Los bloqueos MUST NOT contarse como fallos aparte". Design (design.md:63-64) counts `event = failed` including `failure_reason = locked` rejections and excludes `lockout` rows. "bloqueos" can be read as the `locked` rejections, which the design does count.
Evidence: `SessionEvent` has `Lockout`; `locked` is a `failure_reason` of `failed` rows (add-session-log-and-settings design.md:43-44).
Impact: An implementer or tester following the spec alone could exclude `locked` rows; the scenario does not cover them.
Recommendation: Rewrite as "Las filas `lockout` MUST NOT contarse; los rechazos `locked` MUST contarse como fallos" and add one `locked` row to the "Recuentos del periodo" scenario (expected 4 fallos) or a separate scenario. Add it to task 1.2.

### FE-001 — "Ver sesiones" link drops the period
Severity: MEDIUM
Category: Frontend / Business Logic
File: openspec/changes/add-admin-dashboard/design.md
Line: 73-74 (also tasks.md:94)
Confidence: MEDIUM

Problem: The link goes to the list "filtrado por fallidos" only; the card shows failures of the selected period.
Evidence: `SessionLogListQuery` already supports `event` and a date-range filter (`SessionLogListQuery.php:64`, `:118-137`).
Impact: The list shows all-time failures, a different number from the card the user just clicked.
Recommendation: Specify that the link carries `event=failed` plus the period's date range (first day..today), and assert the URL in the Feature test of 2.3.

## Low Findings

### TEST-001 — No task writes the Dusk test of the two-series chart
Severity: LOW
Category: Testing
File: openspec/changes/add-admin-dashboard/tasks.md
Line: 93, 103
Confidence: HIGH

Problem: 2.2 changes JS (`colors` read from hidden spans) but no task creates a Dusk test; 4.1 only says to hand the user the command.
Evidence: `resources/js/app.js:589` uses `getComputedStyle(this.$el).color` for one colour; the new branch is JS-only behaviour.
Impact: The per-series colour path and the unchanged single-colour path (projects13 sparkline, `summary.blade.php:52`) are untested.
Recommendation: In 2.2 add "test Dusk: the dashboard chart renders two series with distinct colours and the projects13 sparkline still renders".

### LAR-001 — Authorization placement differs from the sibling request
Severity: LOW
Category: Laravel / Consistency
File: openspec/changes/add-admin-dashboard/design.md
Line: 56-59
Confidence: MEDIUM

Problem: Design authorizes in the controller (`authorize('viewAny', ...)`) while it also creates `SessionDashboardRequest`; the sibling `SessionLogListRequest::authorize()` does `can('viewAny', SessionLog::class)`.
Impact: Two patterns for the same resource; a request with default `authorize()` returning false/absent is a common slip.
Recommendation: Put the check in `SessionDashboardRequest::authorize()` like the sibling, and drop the controller call (or state explicitly why not).

### TEST-002 — Non-scalar `period` not covered
Severity: LOW
Category: Testing
File: openspec/changes/add-admin-dashboard/tasks.md
Line: 87, 92
Confidence: MEDIUM

Problem: Only `period=45` is tested; `period[]=7` or `period=abc` must also fall back to 30 without a 500/422.
Recommendation: Add those inputs to the Unit test of 1.1.

## Informational Findings

### INFO-001 — `DATE(created_at)` groups by the database's stored zone
Design says "días en la zona horaria de la app" (design.md:60). That holds only because `app.timezone` is `UTC`; if it changes, `DATE()` will bucket in UTC while the range bounds use the app zone. Add one sentence noting the assumption (or bucket in PHP, as `DashboardSummaryQuery` does for weeks). Also the "SQLite" mention is irrelevant: tests run on MySQL/MariaDB.

### INFO-002 — Index reference
Risks section cites "índice en `created_at`"; the queries filter by `event` first, so the used index is `(event, created_at)` (migration line 32). Wording only.

## Security
No new write routes; read-only under `web, auth, verified` and `sessions.view`. Counts only, no logins/IPs exposed. No issue.

## Bugs / Correctness
BUS-001, FE-001.

## Database
No migrations. Existing `(event, created_at)` index covers both the failed count and the daily series. Users count via `SoftDeletes` on `Basics13\Models\User` is correct (archived = `active=false`, still counted, as designed). INFO-001, INFO-002.

## Performance
One aggregate over at most 90 days, indexed; acceptable without cache, as stated.

## Architecture
Everything stays in `basics13` (sessions + users, both basics13 data) and the host sidebar; no cross-domain coupling. Query class in `src/Queries/Sessions` follows the `DashboardSummaryQuery` pattern. Sidebar `routeIs('sessions.*') && ! routeIs('sessions.dashboard')` mirrors the existing `users.*` exception (sidebar.blade.php:65, :87). No overengineering found.

## Testing
TEST-001, TEST-002; scenario gap in BUS-001.

## Production
Nothing relevant (no deps, no migrations, no queues).

## Maintainability
LAR-001.

## Rejected Findings
- "Hidden `<span>` colour classes may be purged by Tailwind": rejected; basics13 views are an `@source` of the host CSS (`projects/demo13/resources/css/app.css:8`), so literal classes in the view are compiled.
- "Sidebar group not shown with only `sessions.view`": rejected; already handled by add-session-log-and-settings task 7.1.
- "`COUNT(DISTINCT user_id)` loses rows": rejected; `login` rows always have `user_id`, as the design states.
- "90 x-axis labels too dense": style/UX preference without evidence; dropped.
- "Separate capability needed": rejected; data belongs to `basics-login-audit`.

## Action Plan
1. spec.md:111 + scenario + task 1.2: clarify `lockout` excluded / `locked` included and test it (BUS-001).
2. design.md:73-74 + task 2.3: link carries `event=failed` and the period date range; assert it (FE-001).
3. task 2.2: add a Dusk test for two series and for the unchanged single-colour sparkline (TEST-001).
4. design.md:56-59 + task 2.1: authorize in `SessionDashboardRequest::authorize()` like `SessionLogListRequest` (LAR-001).
5. task 1.1: add `period[]=7` and `period=abc` cases (TEST-002).
6. design.md:60-67, 78: note the UTC assumption of `DATE()` and the actual index used (INFO-001, INFO-002).
