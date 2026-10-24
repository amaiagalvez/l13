# Code Review

**Date:** 2026-10-24 (audit started 2026-10-23; finished after midnight)
**Scope:** Whole workspace (`projects/demo13/` host + `packages/{basics13,entities13,projects13,entity-projects13}`), with emphasis on the 2026-10-23/24 wave: sessions + login audit + device types, user types + delegated account management, settings, impersonation, Excel export, project members + visibility, dashboard, list toolbar/backToList.
**Mode:** READ-ONLY. No code, tests, dependencies, configuration or data were modified.

> **Note on delegation:** the 10 specialist subagent reviews were launched, but 9 failed with an upstream error ("Insufficient account funds") before producing output; only the DevOps specialist completed (report saved at `devops.md`). The remaining specialist passes (Laravel, Security, Database, Performance, Business, Architecture, Testing, Frontend, Maintainability) were performed directly by the orchestrator against the same agent checklists, focused on the new wave. Devil's-advocate challenges were applied inline (see Rejected Findings).

> **Scope cutoff:** while this audit ran, the repos moved: new commits `Keep the list page, search and filters after every list action` (all three domain packages) and `add-database-menus` (basics13 `b8e5339`, entities13 `d1a232c`, projects13 `f1720c9`, host `894b7aa`), plus uncommitted working-tree changes in basics13 (lang JSONs, permissions/users views, `composer.lock`). The committed waves up to and including `user-types + dashboard` were reviewed in depth; `add-database-menus` is an in-flight OpenSpec change (`openspec/changes/add-database-menus/`) with a dirty working tree and is **excluded** — review it once it lands.

## Executive Summary

The 2026-10-23/24 feature wave (login audit/session register with lockout and device types, user types with delegated type-admin account management, application settings, super-admin impersonation, Excel export, project membership with visibility narrowing, dashboards) is well engineered and heavily tested (4,635 package tests + 21 host tests, all passing; PHPStan clean at host and all packages). Authorization boundaries are consistently enforced: every project/epic access goes through `ChecksProjectMembership` with `denyAsNotFound()` for non-members, every list/planning/timeline/dashboard/options query is narrowed by `ProjectVisibility`, and delegated user management draws one line (`DelegatesUserManagement`) shared by the policy, lists, selector and permission summary. No security vulnerabilities, no correctness bugs, and no package-boundary violations were found.

Actionable findings are few and minor: one Pint style failure in basics13 that **breaks its lint/CI job** (auto-fixable), one realistic memory risk in the Excel export path (50,000 rows materialized per request), an unbounded account dropdown in the session-log search modal, and a handful of low DevOps hygiene items (no `composer audit` in CI, unused `laravel/sail`, non-strict local MariaDB).

## Detected Stack

- PHP 8.5 (8.5.12 in containers), Laravel 13 (Livewire starter kit), Livewire 4.1 + Blaze, Flux (free), Tailwind 4, Blade + Vite
- MariaDB (Docker), PHPUnit 12.5 (not Pest), paratest, Dusk 8.7, Pint 1.27, Larastan/PHPStan 3.9, maatwebsite/excel, Fortify (login/2FA/passkeys/verification)
- Four path packages symlinked from `packages/` (each its own git repo): basics13, entities13, projects13, entity-projects13
- Docker Compose (laravel13, laravel13-npm, MariaDB, phpMyAdmin, MailHog); GitHub Actions CI in all 5 repos; no API routes (Blade/Livewire only → API reviewer skipped)
- Queues: database driver configured, no application jobs. Redis/Horizon: absent.

## Checks Executed

| Check | Scope | Result |
|---|---|---|
| `pint --test --parallel` | host demo13 | ✅ PASS (38 files) |
| `pint --test --parallel` | 4 packages | ❌ **basics13 FAIL: 1 style issue** (`src/Auth/LoginAudit.php`, `unary_operator_spaces`/`not_operator_with_successor_space`); entities13/projects13/entity-projects13 PASS |
| `phpstan analyse` (Larastan) | host + 4 packages | ✅ PASS (0 errors) |
| `composer audit --no-interaction` | host | ✅ No advisories |
| PHPUnit full suites (`testing13/phpunit.sh`, SQLite) | basics13 / entities13 / projects13 / entity-projects13 | ✅ 846+226+562+432 tests, 20,323 assertions, exit 0 (22 skipped: MariaDB integration group needs `BASICS13_MARIADB_DATABASE`; feature-flag/architecture skips) |
| `php artisan test --compact` | host (isolated `laravel_phpunit` MySQL) | ✅ 21 tests, 96 assertions, exit 0 |
| Dusk | — | **NOT RUN** per project rules (needs an isolated `laravel_dusk` browser run; user command from `projects/demo13/`: `../../testing13/dusktest.sh`) |

## Critical Findings

None.

## High Findings

None.

## Medium Findings

### MED-001 — basics13 CI lint job fails on one Pint rule violation

Severity: MEDIUM
Category: Testing / CI
File: packages/basics13/src/Auth/LoginAudit.php
Line: 285 (`! (clone $failures)->skip(...)` — space between `!` and the parenthesized expression)
Confidence: HIGH

Problem:
`vendor/bin/pint --test` fails on `src/Auth/LoginAudit.php` (`unary_operator_spaces`, `not_operator_with_successor_space`). The basics13 workflow runs `vendor/bin/pint --test` in its lint job (`.github/workflows/tests.yml:80`), so the next push fails the package pipeline.

Evidence:
Local run: `FAIL … 1 style issue` for exactly this file (all 334 other basics13 files pass); the workflow's lint step verified by grep.

Impact:
Red CI on every basics13 push until fixed; the fix is auto-formatting (`vendor/bin/pint src/Auth/LoginAudit.php`).

Recommendation:
Run Pint on the file and commit. Also worth a package pre-commit/CI habit of running `pint --test` locally before pushing.

---

### MED-002 — Excel export materializes up to 50,000 hydrated Eloquent models per request

Severity: MEDIUM
Category: Performance
File: packages/basics13/src/Support/Excel/ExcelExportService.php
Line: 83–92 (`collection()`: `$query->get()` with `limit(50000)`, then `->map()` into a full `Collection`)
Confidence: HIGH (mechanism certain; impact depends on row count)

Problem:
`FromCollection` loads every exported row as a hydrated Eloquent model plus a mapped array copy in memory before the spreadsheet writer starts. All three export controllers (entities13 `EntityController::export`, projects13 `ProjectController::export`/`EpicController::export`) pass `maxRows: 50000`.

Evidence:
`ExcelExportService::collection()` does `$query->get()` (no chunking/streaming) and `Maatwebsite\Excel\Concerns\FromCollection` requires the full collection; `maxRows: 50000` in all three controllers.

Impact:
With tens of thousands of rows, a single export request can exhaust the memory limit or tie up a worker for minutes; exports are user-triggered GET routes, so this is reachable by any user with list access.

Recommendation:
Implement `FromQuery` instead of `FromCollection` (maatwebsite/excel then reads the query in chunks — the same filtered `Builder` is already available), or lower the cap to a realistic figure (e.g. 5,000–10,000 rows) and document it. Simplest safe fix: `FromQuery` + `WithHeadings`/mapping in `WithMapping`.

## Low Findings

### LOW-001 — Session-log search modal loads every account (trashed included) into a dropdown

Severity: LOW
Category: Performance / scalability
File: packages/basics13/src/Queries/Sessions/SessionLogListQuery.php
Line: 104–115 (`accountOptions()` → `pluck('name','id')` over all users with trashed)
Confidence: HIGH

Problem:
The `user_id` filter of the access register's advanced search renders every account as `<option>` (deliberately including archived/trashed/super-admin, per the comment). It is an unbounded `pluck` inlined into the page payload.

Evidence:
`accountOptions()` has no limit; the field renders as a plain multi-select (`'type' => 'select', 'multiple' => true`).

Impact:
Fine at the current account count; page size and memory grow linearly with accounts.

Recommendation:
When the account count grows, switch the field to the existing searchable select-options pattern (`UserSelectOptionsRequest` endpoint) that the user form already uses. No change needed now.

---

### LOW-002 — No dependency advisory check in CI (any repo)

Severity: LOW
Category: DevOps
File: projects/demo13/.github/workflows/tests.yml (and the 4 package workflows)
Confidence: HIGH

Problem/Evidence/Impact/Recommendation: see OPS-001 in `devops.md`. Today's manual `composer audit` on the host is clean, so the gap is detection lag (Dependabot weekly + cooldown ⇒ up to ~12 days), not a present vulnerability.

---

### LOW-003 — Unused dev dependency `laravel/sail`

Severity: LOW
Category: DevOps / dependencies
File: projects/demo13/composer.json:30
Confidence: HIGH

See OPS-003 in `devops.md`. Recommend removing or documenting.

---

### LOW-004 — Local MariaDB runs non-strict (`--sql-mode=""`) while CI runs with defaults

Severity: LOW
Category: DevOps / environment parity
File: projects/demo13/docker-compose.yml:242
Confidence: MEDIUM

See OPS-007 in `devops.md`. Values that would error in CI/production truncate silently in local dev; drop the override.

---

### LOW-005 — Unmaintained MailHog image in the dev stack

Severity: LOW
Category: DevOps
File: projects/demo13/docker-compose.yml:273–278
Confidence: MEDIUM

See OPS-004 in `devops.md`. Dev-only, loopback-bound; plan a switch to Mailpit when convenient.

## Informational Findings

- **INFO-001 (OPS-002):** Host CI checks sibling packages out at default-branch HEAD and only the host runs Dusk — already recorded in ARCHITECTURE.md "Known Technical Debt"; unchanged.
- **INFO-002 (OPS-009):** `amaia/*: "*"` constraints float freely if the path repositories are ever removed/versioned; `^1.0` would match the packages' own `version: 1.0.0`.
- **INFO-003 (OPS-008):** The `laravel13-composer` compose helper deletes the committed `composer.lock`; a casual `--profile composer` run silently rebuilds it from unconstrained resolutions. Back up instead, or gate behind confirmation.
- **INFO-004:** Login lockout is per-login without IP (`LoginAudit::lockoutKey`), so a known email can be deliberately locked out for `lockout_minutes` by anyone typing wrong passwords for it. This is a documented, deliberate tradeoff (the class docblock and the MariaDB `basics-login-audit` spec describe exactly this) and `lockout_minutes` is configurable in `BAS_settings`; recorded so it is a conscious decision, not an oversight.
- **INFO-005:** The 22 skipped package tests are environment-gated by design (MariaDB integration group requires `BASICS13_MARIADB_DATABASE`; Fortify-feature and architecture checks skip when not applicable), not silent coverage loss.
- **INFO-006:** `add-database-menus` (OpenSpec change in flight; new commits in all repos + dirty basics13 working tree) was excluded from this review — review it after it lands.

## Security

Verified, no findings:

- **Authorization / IDOR:** Project and epic policies answer `denyAsNotFound()` for non-members (`ChecksProjectMembership::decideOn`), so URLs do not reveal existence. Every list, planning, timeline, dashboard, project-options and epic-form project-options query is narrowed through `ProjectVisibility` (verified by grep: `ProjectListQuery`, `EpicListQuery`, `PlanningQuery`, `TimelineQuery`, `DashboardSummaryQuery`, `ProjectSelectOptionsQuery` (editable-only), `EpicRequest`). Member management (list/candidates/store/update/destroy) is gated by `manageMembers` (owner or `projects.manage`); member candidates expose only `{id, name}` of active, non-trashed, non-super-admin, non-member accounts.
- **Delegated user management:** type admins (`users.manage_own` without global `users.edit`) are narrowed consistently by `DelegatesUserManagement` in the policy (`allows()`), the lists (`UserListQuery::visibleTo`), the selector, and the permission summary; peers (admins of a shared type), the actor itself, super-admins and global managers are out of reach. Role assignment by a type admin is limited to roles whose every permission the actor holds (`grantableRoleIds`); the type admin can never set the `is_admin` pivot flag (`UserRequest::types()`).
- **Mass assignment:** `active` is never fillable from requests (dedicated archive/activate actions only; `UserRequest`/`UserTypeRequest` validated fields are name/notes/password/email only). Settings update reads only `values.{variable}` with per-type rules; type/name/description come from the seeding migration.
- **Impersonation:** `UserPolicy::impersonate` requires the super-admin, never the super-admin account itself and never one's own account; routes are POST + auth + verified; the top bar shows the red impersonation banner and leave button; leaving restores the original user and the login audit closes the impersonation row.
- **Login audit:** the password is never read; lockout uses a cache-based lock with `Cache::add` race protection; failure-spike alert is rate-limited to once per 5-minute window; `alert_email` is optional and validated.
- **Session table:** `BAS_sessions` (correctly prefixed, no clash with Laravel's `sessions`) stores ip (45, IPv6), user_agent (512), uuid session_key unique; `impersonator_id` FK `nullOnDelete`.
- **XSS/redirects:** Blade output is escaped (`{{ }}`) throughout the new views; `Controller::backToList()` only redirects when the previous URL equals the list route or extends it with `?…` — no open redirect.
- **Secrets:** none in CI (no secrets present in the workflows; `persist-credentials: false`); dev credentials documented as local-only.

## Bugs / Correctness

None found in the reviewed wave. Points specifically checked: lockout counter vs. lock interaction (`RateLimiter::hit` then `Cache::add`, only the request that sets the lock writes the row); `whereKeyNot([])` on an empty active-user collection selects all accounts (Laravel `0=1` negation — intended); impersonation open/close session-key juggling in `LoginAudit` (login of the impersonator's own id closes the row and restores the key); user-type name uniqueness across four language columns with blank-to-NULL normalization and both MySQL/SQLite index-name spellings on the race fallback; `PRO_project_members` FK `cascadeOnDelete` on force delete and members kept through soft delete (restore restores access).

## Database

- New migrations reviewed: `BAS_settings`, `BAS_sessions` (+`device_type`, +`impersonator_id`), `BAS_user_types`, `BAS_user_user_type` (unique `project|user`-style composite keys where applicable), role-table `BAS_` prefix renames. Indexes present where queried: `BAS_sessions(created_at)`, `(user_id, created_at)`, `(event, created_at)`, unique `session_key`; `impersonator_id` FK.
- Delete guards follow the established `lockForUpdate()` transaction pattern (`UserTypeController::destroy` comment confirms the deleting hook holds the row lock and refuses while accounts hold the type).
- All `BAS_`/`PRO_`/`ENT_` table references updated consistently (verified via the `HasRoles`/`pluck('BAS_roles.id')` diffs; PHPStan clean with no ignored-error regressions of note).
- No destructive migrations; `down()` steps drop only what `up()` created.

## Performance

- MED-002 (Excel export memory) and LOW-001 (unbounded account dropdown) above.
- `SessionDashboardQuery::summary()` runs ~10 small indexed queries on an admin-gated page; `activeUserIds()` is computed twice. Rejected as immaterial (see Rejected Findings).
- `LoginAudit` adds one `Setting::get()` + one indexed user lookup per failed login and one active-check per login — deliberate, documented "no cache so a changed value applies on the next request" design.

## Architecture

- **Package boundaries respected.** The bridge owns all cross-domain logic: `EntityCountsIntegration` (counts, delete guard, entity-selector narrowing to visible projects) lives in entity-projects13 and uses the new `Entities13\Contracts\EntityIntegration` extension points; entities13/projects13 remain independent of each other and of the bridge. `ProjectVisibility` is a projects13 concern the bridge consumes, not the reverse.
- `basics13` additions (login audit, settings, Excel service, delegation) are domain-neutral infrastructure; no entity/project business logic leaked in.
- The delegation/visibility model keeps one source of truth per boundary (`DelegatesUserManagement`, `ProjectVisibility`, `ChecksProjectMembership`) — no duplication of the rules across entry points.
- No speculative abstractions introduced; the rejected-patterns list (no repositories/DTOs/services) is still honored.

## Testing

- Full suites green (see Checks Executed); the wave is covered by dedicated suites (sessions/login-audit, user types/delegation, settings, impersonation, excel export, members/visibility, dashboard) reflected in the 846 basics13 + 562 projects13 test counts with 13,880 + 2,845 assertions.
- Skips are environment-gated, not silent (INFO-005).
- Dusk not run in this review (project rule); per ARCHITECTURE.md, projects/epics trash/archived/drawer flows are feature-tested only — unchanged known coverage posture.
- Placement follows the project rules (Unit without DB, Feature with real rows; MariaDB group tagged).

## Production

- `.env.example`: `APP_DEBUG=false`, empty key; debugbar/pail dev-only; log viewer deny-by-default. Docker dev credentials and exposed admin ports are documented local-only assumptions (ARCHITECTURE.md "Production Assumptions").
- CI green except the basics13 lint job (MED-001). No `composer audit` gate in CI (LOW-002).

## Maintainability

- New code follows sibling structure exactly (controller + trash/archived controllers + requests + policies + queries + transformers + views + 4-language JSON per resource).
- Documentation quality is high: non-obvious decisions are explained in docblocks (lockout key without IP, day-bounds instead of `whereDate`, left-join rationale, blank-to-NULL name normalization, deny-as-not-found).
- Minor: `UserRequest` has grown to ~300 lines handling global-editor and type-admin paths for roles and types; still cohesive and well-commented — watch it as a future split candidate only if it keeps growing (no action now).

## Rejected Findings

Challenged and dropped (devil's advocate):

- **"`DeviceType::label()` uses bare `__()` without namespace → missing translations"** — verified: `basics13` registers `loadJsonTranslationsFrom` and `Desktop/Mobile/Tablet` exist in all four lang JSONs. False positive.
- **"`whereKeyNot([])` on empty active-user ids builds broken SQL"** — Laravel renders `0=1` negated; semantics are exactly "nothing excluded". False positive.
- **"Setting::get() is uncached → per-login query"** — deliberate documented design ("no cache, so a changed value applies on the next request"); admin page and lockout only. Stylistic/known tradeoff.
- **"handleLogin re-checks `active` although the active-users listener already rejects"** — explicitly documented as order-independent hardening. Rejected.
- **"SessionDashboardQuery::summary calls activeUserIds twice / runs many counts"** — admin-only page, indexed queries, no evidence of impact. Speculative optimization.
- **"Lockout enables account-DoS"** — deliberate, documented design choice with a configurable window (kept as INFO-004, not a defect).
- **"leaveImpersonation 404s if the original account was deleted"** — requires a super-admin to be deleted mid-impersonation; recoverable by logging out. Edge case not worth code.
- **OPS-005 (dev image as prod base) and OPS-006 (duplicated dev creds)** — documented local-only assumptions in ARCHITECTURE.md; kept as notes in devops.md, not findings with action items.
- **"Member candidates include soft-deleted users"** — soft-delete global scope excludes them; `active=true` + `is_super_admin=false` + not-a-member already narrow it. False positive.

## Action Plan

1. **MED-001** — Run `vendor/bin/pint` on `packages/basics13/src/Auth/LoginAudit.php` and commit; the basics13 lint/CI job fails on it today. (1 minute, unblocks CI.)
2. **MED-002** — Switch `ExcelExportService` to `FromQuery` (chunked reads) or lower the 50,000-row cap in the three export controllers; verify with a large seeded export. (Small, contained change in basics13 + optional controller cap edits.)
3. **LOW-002** — Add `composer audit --no-interaction --abort-on-unfixable` to the `ci` job of the workflows (host first, packages after).
4. **LOW-004** — Drop `--sql-mode=""` from the local MariaDB so dev matches CI/production behavior.
5. **LOW-003 / LOW-005** — Remove unused `laravel/sail`; plan the MailHog → Mailpit swap when convenient.
6. **LOW-001** — No action now; revisit the session-log account filter when the user count grows.
7. After `add-database-menus` lands (in-flight OpenSpec change), request a focused follow-up review of it.

## Resolution (closed 2026-10-24)

| ID | Status | Commit |
|----|--------|--------|
| MED-001 | Not reproducible: `pint --test` passes on all basics13 files, `LoginAudit.php` unchanged | — |
| MED-002 | Fixed: `ExcelExportService` uses `FromQuery` + `WithMapping` (chunked read, `maxRows` kept); test for chunking under the cap | basics13 `65ed10b` |
| LOW-001 | Deferred, as recommended | — |
| LOW-002 / OPS-001 | Fixed: `composer audit --no-interaction --abort-on-unfixable` in all five CI workflows | host `53d779a`, basics13 `65ed10b`, entities13 `84ad4f9`, projects13 `5788925`, entity-projects13 `0e2857b` |
| LOW-003 / OPS-003 | Fixed: `laravel/sail` removed | host `53d779a` |
| LOW-004 / OPS-007 | Fixed: `--sql-mode=""` dropped; local MariaDB runs strict | host `53d779a` |
| LOW-005 / OPS-004 | Fixed: MailHog replaced by Mailpit (`axllent/mailpit:v1.27`) | host `53d779a`, l13 `ea9c09e` |
| OPS-005 | Fixed: port 80 bound to `127.0.0.1` | host `d44a844` |
| OPS-008 | Fixed: clean-install helpers keep `composer.lock` / `package-lock.json` | host `d44a844`, l13 `dd285fc` |
| OPS-009 | Fixed: `amaia/*` constrained to `^1.0` | host `d44a844` |
| OPS-002, OPS-006, INFO-* | No action (documented debt / deliberate tradeoffs) | — |

Validation: host `composer ci:check` (21 tests, 100 % coverage); package suites basics13 (859), entities13 (230), projects13 (565), entity-projects13 (432) green; PHPStan basics13 clean. Dusk not run.
