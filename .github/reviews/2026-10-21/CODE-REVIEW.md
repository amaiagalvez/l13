# Code Review

Date: 2026-10-21 · Mode: READ-ONLY full audit (`/full-review`) · Orchestrator: `.github/agents/review-orchestrator.agent.md`

Raw specialist reports are in this folder: `laravel.md`, `security.md`, `database.md`, `performance.md`, `business.md`, `architecture.md`, `testing.md`, `api.md`, `frontend.md`, `maintainability.md`, `devops.md` and `devil-advocate.md`.

> This file replaces the earlier spec review of `add-roles-permissions-system` that was saved at this path. That review is still available in git: `git show 44883e0:.github/reviews/2026-10-21/CODE-REVIEW.md`.

## Executive Summary

The core CRUD, trash and archive domain is solid:
- The unique-name invariants are enforced in both validation and the database.
- Delete guards lock the parent row and count trashed children.
- `active` is never mass assignable.
- Inactive users are blocked.
- The package boundaries hold at the import level.
- All 1,549 package tests and the 10 app tests pass.

Almost every important problem comes from the **advanced search / filters** work. It is committed and pushed on the `develop` branch of the packages (`base - search`), and has further uncommitted changes.

- **CRITICAL: remote code execution.** The Livewire `AdvancedSearchModal` calls functions that come from client-controlled public state. The orchestrator reproduced it: the client-chosen function ran on the server with a client-chosen argument (SEC-001).
- **HIGH: account takeover.** Self-registration is on by default, and any user may set any other user's password (SEC-002).
- **HIGH: production deploys break.** The bridge puts closures into config, so `php artisan config:cache` / `optimize` fails (reproduced, OPS-001).
- **MEDIUM:**
  - Advanced filters have several functional bugs: LIKE escaping, tab counts, the URL state, the date operator and the project labels.
  - A comment can be added to an epic that is being trashed at the same moment.
  - Package CI is red (PHPStan), and the uncommitted entities13 composer path would break that package's CI if committed.
  - The MariaDB-only invariant tests never run in CI.
  - Package CI does not run on `develop` pushes at all, which is why none of this was caught.

Totals of accepted findings: 1 CRITICAL, 2 HIGH, 15 MEDIUM, 21 LOW. 19 findings were rejected.

## Fix Status

| ID | Status |
|---|---|
| SEC-001 | ✅ FIXED 2026-10-21 — #[Locked] fields/scope, no state callables invoked; tamper tests |
| BUG-001 | ✅ FIXED 2026-10-21 (by another session) — JSON label keys; Crud tests assert no raw keys |
| BUG-002 | ✅ FIXED 2026-10-21 — search/filters start from current URL; hidden inputs + clear link keep filters; Dusk test |
| BUG-003 | ✅ FIXED 2026-10-21 — operator wire:model.live + date value reset; unit + Dusk test |
| BUG-004 | ✅ FIXED 2026-10-21 — escape '!' declared; wildcard tests |
| BUG-005 | ✅ FIXED 2026-10-21 — shortcut only without search and filters (12 controllers); feature test |
| BUG-007 | ✅ FIXED 2026-10-21 — malformed values dropped in request; undeclared operators / wrong shapes skipped |
| BUG-008 | ✅ FIXED 2026-10-21 — condition kept (spec MUST) but evaluated once in mount; failing fields dropped |
| OPS-001 | ✅ FIXED 2026-10-22 — bridge field via new ProjectQueryIntegration::searchableFields() (default []); config key removed; config:cache verified; serialization test |
| OPS-002 | ✅ RESOLVED 2026-10-22 (reverted by another session) — composer.json/lock back to ../packages/basics13, no diff vs HEAD |
| OPS-003 | ✅ FIXED 2026-10-22 — entities13 baseline restored (another session), basics13 test errors gone; env baseline count updated; PHPStan OK in all 4 packages |
| TEST-003 | ✅ FIXED 2026-10-22 — user options tests assert exact active ids and order; 38 dead @requires docblocks removed |
| DB-001 | ✅ FIXED 2026-10-22 — lock, trashed check and insert in one transaction; transaction-level test |
| SEC-003 | ✅ FIXED 2026-10-22 — comments read authorizes viewAny; 403 test |
| DB-002 | ✅ FIXED 2026-10-22 — no-op booted() delete hooks removed from Epic and Project; provider guards remain the single source |
| DB-003 | ✅ FIXED 2026-10-22 — epic guard counts trashed comments; test |
| BUG-006 | ✅ FIXED 2026-10-22 — start_date also bounded by the project end; test |
| PERF-001 | ✅ FIXED 2026-10-22 — timeline epic rows use the loaded project; bridge eager-loads project.entity for epic lists; query-count tests (epic list 14→8 verified). Remaining: planning/timeline entity per project row, bounded by page size |
| ARCH-001 | ✅ FIXED 2026-10-22 — neutral blocks_delete key (documented on EntityIntegration::row) and entity_related alias; projects_exists/entity_projects added to PackageIsolationTest FORBIDDEN |
| ARCH-002 | 🟡 PARTIAL 2026-10-22 — join aliases neutral (project/epic/planning/timeline_related) and hasEntities→hasRelated; data-test names kept (views rule) and contract method hasActiveRelatedEntities kept (contract rename not approved) |
| ARCH-003 | ✅ FIXED 2026-10-22 — basics13 findTrashedByName() domain-neutral (+ trashedQuery()); EpicListQuery overrides it with the project scope; tests moved to projects13 |
| BUG-010 | ✅ FIXED 2026-10-22 — trashed type/project after validation → ValidationException on entity_type_ids / project_id (redundant nested transaction removed); tests |
| BUG-011 | ⏸️ NOT CHANGED 2026-10-22 — the change spec (specs/basics-crud-base, entities-domain, projects-domain) MUSTs an active boolean field on every scope; needs a spec change first |
| BUG-009 | ✅ FIXED 2026-10-22 — envelope narrowed flag (search or filters): no-match message and no create shortcut; tests |
| MAINT-001 | ✅ FIXED 2026-10-22 — label props/attributes, unused grouping methods, $filtersB64 and dead ?? fallbacks removed |
| FE-001 | ✅ FIXED 2026-10-22 — value inputs and selects carry the field label (from/to on ranges); render test |
| FE-002 | ✅ FIXED 2026-10-22 — advanced toggle has aria-expanded; render test |
| FE-003 | ✅ FIXED 2026-10-22 — bridge eu.json uses Bezeroa and valid Basque |
| MAINT-002 | ✅ FIXED 2026-10-22 — ARCHITECTURE (user active flag, account holder rule, comment FKs, transactions, PHP 8.4, contract decision), models.instructions layer 7 and ArchivedController docblocks corrected |
| OPS-004 | ✅ RECORDED 2026-10-22 — accepted as known debt in ARCHITECTURE.md |
| OPS-005 | 🟡 PARTIAL 2026-10-22 — laravel/boost moved out of basics13 require into demo13 require-dev (locks updated, boost v2.10.3 kept, boost:* commands OK); laravel/chisel kept in require: InstallFeaturesCommand uses it at runtime |
| TEST-001 | ✅ FIXED 2026-10-22 — package CI runs on develop; basics13/entities13/bridge workflows get a MariaDB service and run #[Group('mariadb')] (15 tests, all passing locally on MariaDB); entities13 TestCase honours DB_CONNECTION. Remaining: sibling checkouts use the default branch |
| TEST-002 | ✅ FIXED 2026-10-22 — project and epic filter feature tests: date range, is_empty, project name, archived and trash scopes, search plus filters, pagination keeps filters |
| BUG-012 | ⏸️ PENDING DECISION — the epics-comments spec does not say whether moving a project's dates must respect its epics; clarify the spec before coding |
| PERF-002 | ⏸️ ACCEPTED 2026-10-22 — acceptable at current data sizes (options now locked server-side by SEC-001); revisit with the capped entities.options endpoint when entities grow |
| SEC-002 | 🟡 PARTIAL 2026-10-22 — only the account holder changes password/email (request + drawer, tests). Registration stays basics13.auth.features.registration = true (owner decision: a host disables it in its own config, not via .env); demo13 still allows open sign-up |
| OPS-006 | ⏸️ NOT CHANGED 2026-10-22 — owner decision: docker-compose.yml stays as is; docker-commands skill now documents that composer/npm-all delete and regenerate the lock files |

## Detected Stack

| Area | Detected |
|---|---|
| PHP | 8.4.26 in Docker, CI on 8.4. `composer.json` declares `^8.3`, which is never tested |
| Framework | Laravel 13, Livewire 4 (+ `livewire/blaze`), Flux free, Fortify (2FA, passkeys) |
| Database | MariaDB 11.7. App PHPUnit uses `laravel_phpunit`, Dusk uses `laravel_dusk`. Package Testbench suites use SQLite, with MariaDB opt-in |
| Frontend | Blade, Alpine (via Livewire), jQuery 4 + select2, Tailwind 4, Vite Plus 1.0.0 |
| Queues / Redis / Horizon | Database queue driver, no jobs. No Redis, no Horizon |
| Tests | PHPUnit 12, Dusk 8 (in the demo13 CI). No Pest |
| Static analysis / style | Larastan (level 7 in the host), Pint |
| Docker | `webdevops/php-apache-dev:8.4`, MariaDB, phpMyAdmin, MailHog, Dusk image |
| CI/CD | GitHub Actions per repo, SHA-pinned. Package workflows trigger on `master` pushes, PRs and manual dispatch only |
| Layout | Host `projects/demo13` plus 4 path packages (`basics13`, `entities13`, `projects13`, `entity-projects13`), each its own git repo. The packages are on branch `develop` with uncommitted advanced-search changes |

## Checks Executed

| Check | Result |
|---|---|
| `pint --test` (app) | PASS (40 files) |
| `phpstan analyse` (app) | OK, no errors |
| `composer audit` (app) | No advisories |
| `php artisan test` (app, `laravel_phpunit`) | 10 passed (38 assertions) |
| `testing13/phpunit.sh basics13` | OK, 445 tests, **9 skipped** |
| `testing13/phpunit.sh entities13` | OK, 185 tests, **1 skipped** |
| `testing13/phpunit.sh projects13` | OK, 476 tests |
| `testing13/phpunit.sh entity-projects13` | OK, 433 tests, **5 skipped** |
| Pint per package | PASS in all 4 |
| PHPStan per package (`testing13/review.sh`) | projects13 and entity-projects13 OK. **basics13: 3 errors** (tests). **entities13: 132 errors** (`ignore.unmatched` from the staged baseline) |
| `config:cache` (throwaway container, scratch `bootstrap/cache`) | **FAIL**: "Your configuration files are not serializable." |
| SEC-001 proof (throwaway container, `Livewire::test`, harmless `print_r`) | **Reproduced** |
| `npm run build` | NOT RUN: it writes `public/build` |
| Dusk | NOT RUN: slow browser run; the review is static for UI flows |

All 15 skipped tests are MariaDB-only: `MariaDbUniqueActiveNameIndexTest` (9 skips), `EntityTypeTrashTest:143`, `ActiveNameCollationTest:21` and `ProjectCrudTest:384`.

## Critical Findings

### SEC-001 — Livewire public state is called as a PHP function (remote code execution)

Status: ✅ FIXED 2026-10-21 — #[Locked] fields/scope, no state callables invoked; tamper tests
Severity: CRITICAL
Category: Security (code injection)
File: packages/basics13/src/Livewire/AdvancedSearchModal.php
Line: 96-97, 198-199 (properties at 22, 27; rendered from resources/views/components/list/advanced-search-modal.blade.php:37, 48)
Confidence: HIGH

Problem:

`$fields` and `$scope` are public and not `#[Locked]`, so the browser can change them. `getVisibleFields()` executes `$field['condition']($this->scope)`, and `getSelectOptions()` executes `$field['options']($this->scope)`. `mount()` already resolves the real closures (`normalizeField()`, lines ~290-306), so on later requests these values are whatever the client sent.

Evidence:

```php
public array $fields = [];        // 22
public string $scope = 'active';  // 27
$visibleFlag = $visibleFlag && $field['condition']($this->scope);   // 97
return $field['options']($this->scope);                              // 199
```

In a throwaway container, `Livewire::test(AdvancedSearchModal::class, …)->set('fields.name.condition', 'print_r')->set('scope', 'LW-INJECTION-MARKER')->set('show', true)` rendered HTML containing `LW-INJECTION-MARKER`. Using `system` instead of `print_r` would run a shell command. The code is committed and pushed in basics13 `develop` (500fbeb). `master` does not contain it.

Impact:

Any authenticated, verified user can execute arbitrary code on the server. That gives access to `.env` (APP_KEY, DB credentials) and the whole database. With open registration (SEC-002), this means anyone who can register.

Recommendation:

1. Delete the runtime calls at lines 96-98 and 198-200, and read only the booleans and arrays that `normalizeField()` already produced.
2. Add `#[Locked]` to `$fields` and `$scope`.
3. Add a regression test that tampers `fields.*.condition` and `fields.*.options` and asserts that nothing is invoked.
4. Remove `condition` entirely (see BUG-008); it hides nothing today.
5. Do not merge `develop` into `master` before this is fixed.

## High Findings

### SEC-002 — Open registration plus peer password change allows account takeover

Status: 🟡 PARTIAL 2026-10-22 — only the account holder changes password/email (request + drawer, tests). Registration stays basics13.auth.features.registration = true (owner decision: a host disables it in its own config, not via .env); demo13 still allows open sign-up
Severity: HIGH
Category: Security (authentication / authorization)
File: packages/basics13/config/basics13.php
Line: 7 (also packages/basics13/src/Policies/UserPolicy.php:37-40; packages/basics13/src/Http/Requests/UserRequest.php:42-48, 66-74; packages/basics13/routes/users.php:33)
Confidence: HIGH

Problem:

- `'registration' => true` is the package default, and the host has no `config/basics13.php` override.
- `UserPolicy::update()` returns `true`.
- `UserRequest` accepts an optional new `password` (and `email`) for any `{user}`, with no current-password check.

Evidence:

```php
public function update(User $user, Model $record): bool { return true; }
// UserRequest::passwordRule(): on edit, rules[0] = 'nullable' — any editor may set it
```

Impact:

A stranger can register, verify their email, send `PUT /users/{id}` with a new password, and log in as the victim. Only the victim's own 2FA stops this. Changing the email also grants log-viewer access, which is gated by email (`packages/basics13/src/ServiceProvider.php:109-116`). The documented "every user can do everything" decision covers resources; it does not cover public sign-up combined with resetting other users' passwords.

Recommendation:

- Now: set `registration` to false in the host (or make false the package default).
- Next: do not let one user change another user's password or email through `UserRequest` (accept those fields only for the user's own account, or drop them from the admin form).
- Do not introduce roles for this.

### OPS-001 — Closures in runtime config break `config:cache` / `optimize`

Status: ✅ FIXED 2026-10-22 — bridge field via new ProjectQueryIntegration::searchableFields() (default []); config key removed; config:cache verified; serialization test
Severity: HIGH
Category: Production / Laravel configuration
File: packages/entity-projects13/src/ServiceProvider.php
Line: 100-123 (closures at 113, 118; read at packages/projects13/src/Queries/Projects/ProjectListQuery.php:162)
Confidence: HIGH

Problem:

`declareProjectSearchFields()` writes `options` and `condition` closures into `config('projects13.searchable_fields')`.

Evidence:

`php artisan config:cache`, run in a throwaway container with a scratch `bootstrap/cache`, throws `LogicException: Your configuration files are not serializable.` The code is committed on entity-projects13 `develop` (d1fda1f).

Impact:

Standard production deploy steps fail whenever the bridge is installed.

Recommendation:

Keep config serializable. Let the bridge contribute the field through the existing `ProjectQueryIntegration` seam: add one method that returns `[]` by default. This is a contract change, so ask first, as `packages.instructions.md` requires. Drop `condition` (BUG-008). Do not replace the closures with callable strings or class names: they would keep the SEC-001 pattern. Add a test that serializes `config()->all()`.

## Medium Findings

### OPS-002 — entities13 Composer path repository changed to a container-only path (uncommitted)

Status: ✅ RESOLVED 2026-10-22 (reverted by another session) — composer.json/lock back to ../packages/basics13, no diff vs HEAD
Severity: MEDIUM
Category: Production / dependencies
File: packages/entities13/composer.json
Line: 51 (lock: packages/entities13/composer.lock:15)
Confidence: HIGH

Problem: `"url": "../packages/basics13"` was changed to `"/packages/basics13"`, and the lock was regenerated to match. These changes are unstaged.

Evidence: the `git diff` shows the change. `/packages` exists only in the Docker mount (`projects/demo13/docker-compose.yml:7`). The sibling packages keep `../packages/basics13`.

Impact: if committed, `composer install` in the entities13 CI and in any checkout outside Docker cannot resolve basics13, so no checks run.

Recommendation: revert both files to `../packages/basics13` before committing.

### OPS-003 — Package PHPStan gates are red

Status: ✅ FIXED 2026-10-22 — entities13 baseline restored (another session), basics13 test errors gone; env baseline count updated; PHPStan OK in all 4 packages
Severity: MEDIUM
Category: Production (CI) / maintainability
File: packages/entities13/phpstan-baseline.neon
Line: 1-793 (staged); packages/basics13/tests/Unit/Livewire/AdvancedSearchModalTest.php:140; packages/basics13/tests/Unit/Queries/Users/UserSelectOptionsQueryTest.php:72
Confidence: HIGH

Problem: the staged entities13 baseline lists TestCase-resolution errors that `scanFiles` already fixes, which produces 132 `ignore.unmatched` errors. basics13 has a vacuous `assertTrue(true)` and an `isset` that is always true.

Evidence: the `testing13/review.sh` output (run by the orchestrator).

Impact: both package workflows fail at PHPStan, before their tests run.

Recommendation: unstage and regenerate the entities13 baseline (it should be empty). Replace the `assertTrue(true)` with `assertDispatched(...)`, and assert exact ids instead of `isset`.

### DB-001 — Epic comment lock is released before the insert

Status: ✅ FIXED 2026-10-22 — lock, trashed check and insert in one transaction; transaction-level test
Severity: MEDIUM
Category: Database (concurrency)
File: packages/projects13/src/Http/Controllers/EpicCommentController.php
Line: 42-54
Confidence: HIGH

Problem: `lockForUpdate()` runs in its own `DB::transaction()`, which commits at line 45. The `trashed()` check and `save()` then run with no lock.

Evidence: compare `EpicController::store`, where the lock and the write share one transaction. The comment at line 41 claims a protection the code does not give.

Impact: a concurrent epic `destroy` can pass the "no comments" guard, and the comment is then saved on a trashed epic. That breaks `openspec/specs/epics-comments` ("El borrado se bloquea en cuanto hay comentarios"), and the epic can then never be force-deleted.

Recommendation: put the lock, the check and the save in a single `DB::transaction()`.

### BUG-001 — Project filter labels render raw translation keys

Status: ✅ FIXED 2026-10-21 (by another session) — JSON label keys; Crud tests assert no raw keys
Severity: MEDIUM
Category: Bugs / i18n
File: packages/projects13/src/Queries/Projects/ProjectListQuery.php
Line: 110, 118, 127, 136, 145, 153
Confidence: HIGH

Problem: the labels are `projects13::messages.name` and similar.

Evidence: `packages/projects13/resources/lang/{en,es}/` contain only `validation.php`, and there is no `eu` or `fr` PHP file. The package uses JSON translations.

Impact: the modal and badges show `projects13::messages.name` in every locale, including the default `eu`.

Recommendation: use the existing JSON keys (`'Name'`, `'Notes'`, …), as `EntityListQuery` does.

### BUG-002 — Search and advanced filters overwrite each other in the URL

Status: ✅ FIXED 2026-10-21 — search/filters start from current URL; hidden inputs + clear link keep filters; Dusk test
Severity: MEDIUM
Category: Bugs (frontend state)
File: packages/basics13/resources/js/app.js
Line: 499-542
Confidence: HIGH

Problem: `search()` and `applyFilters()` both build the URL from `form.action`, which is `route(...)` with no query string (`ListTransformer`).

Evidence: applying filters drops `search`, and typing a search drops `filters[...]`. Because only the fragment refreshes, the filter badges stay visible but no longer apply.

Impact: users see filters that are not in effect, and they lose their criteria without notice.

Recommendation: start from `new URL(location.href)`, delete `page`, and set or delete only the key being changed.

### BUG-003 — Changing a date operator does not re-render the value inputs

Status: ✅ FIXED 2026-10-21 — operator wire:model.live + date value reset; unit + Dusk test
Severity: MEDIUM
Category: Bugs (Livewire state)
File: packages/basics13/resources/views/components/list/advanced-search-modal.blade.php
Line: 44, 65, 89-102
Confidence: HIGH

Problem: the operator select uses deferred `wire:model`, so the one-input or two-input layout is decided by the operator the modal last rendered with.

Evidence: the default date operator is `between` (`ListQueryBase.php` ~475). Switching to `before` still posts `[d1, d2]`. Switching from a single operator to `between` posts a scalar, which `applyDateFilter` ignores (around line 222).

Impact: date filters silently return wrong results after the operator is changed.

Recommendation: use `wire:model.live` on the operator select (or `x-show` on the client), and reset the value when its shape changes.

### BUG-004 — Advanced text filters escape LIKE wildcards with `!` but never declare `ESCAPE '!'`

Status: ✅ FIXED 2026-10-21 — escape '!' declared; wildcard tests
Severity: MEDIUM
Category: Bugs / database query
File: packages/basics13/src/Queries/ListQueryBase.php
Line: 199-206 (helper 362-368; correct pattern at 105)
Confidence: HIGH

Problem: `escapeLikeValue()` prefixes `%`, `_` and `!` with `!`, but `where($column, 'LIKE', …)` sends no escape clause. MariaDB's default escape character is `\`.

Evidence: `whereMatches()` uses `' like ? escape \'!\''`. `applyTextFilter()` does not. Seven specialists reported this independently.

Impact: `contains`, `starts_with` and `ends_with` return wrong rows (usually none) for values containing `%`, `_` or `!`, and they disagree with the simple search.

Recommendation: reuse the `whereMatches` clause, and add one test with `%`, `_` and `!`.

### BUG-005 — State tab counts show the filtered total when filters are active

Status: ✅ FIXED 2026-10-21 — shortcut only without search and filters (12 controllers); feature test
Severity: MEDIUM
Category: Bugs
File: packages/entities13/src/Http/Controllers/EntityController.php
Line: 40 (and 9 more call sites: Entity{Archived,Trash}, EntityType{,Archived}, Project{Archived,Trash}, Epic{,Archived,Trash} controllers)
Confidence: HIGH

Problem: `stateCounts(activeTotal: $search === '' ? $entities->total() : null)`. The paginator is now also narrowed by `$filters`.

Evidence: `ListQueryBase::paginate()` applies the filters before paginating. No test asserts the counts while a filter is active.

Impact: the Active, Archived and Trash badges show wrong totals.

Recommendation: reuse the total only when `$search === '' && $filters === []`, and add one feature test.

### BUG-006 — An epic without an end date may start after its project ends

Status: ✅ FIXED 2026-10-22 — start_date also bounded by the project end; test
Severity: MEDIUM
Category: Bugs (business rule)
File: packages/projects13/src/Http/Requests/EpicRequest.php
Line: 64-73
Confidence: HIGH

Problem: `start_date` has a lower bound only (`after_or_equal:<project start>`). The `before_or_equal:<project end>` rule is added only to `end_date`, and only when that date is present.

Evidence: `openspec/specs/epics-comments` says "Sus fechas MUST quedar dentro de las del proyecto". When `end_date` is set, the chain `start < end <= project.end` holds, so the gap is limited to epics with no end date.

Impact: an epic can be placed outside its project window, and it shows incorrectly on planning and timeline.

Recommendation: when the project has an `end_date`, add `before_or_equal:<project end>` to `start_date`.

### PERF-001 — Timeline lazy-loads the project for every epic

Status: ✅ FIXED 2026-10-22 — timeline epic rows use the loaded project; bridge eager-loads project.entity for epic lists; query-count tests (epic list 14→8 verified). Remaining: planning/timeline entity per project row, bounded by page size
Severity: MEDIUM
Category: Performance (N+1)
File: packages/projects13/src/Queries/Timeline/TimelineQuery.php
Line: 365 (eager load at 230-238)
Confidence: HIGH

Problem: `epicRow()` reads `$epic->project->name`, but only `epics` are eager-loaded and there is no inverse relation, so each epic runs one query. Planning (`PlanningQuery.php:247`) and the epic lists (`EntityProjectsIntegration.php:111, 245`) also lazy-load the entity per project row, bounded by page size.

Evidence: no `preventLazyLoading()` is enabled (`projects/demo13/app/Providers/AppServiceProvider.php`).

Impact: on the most loaded screen, query count grows with the number of epics on the page, with no upper bound.

Recommendation: pass `$project` (or its name) into `epicRow()`. For the per-page entity loads, add `project.entity:id,name,active` to the bridge's eager load (one line). No new scopes are needed.

### ARCH-001 — entities13 hard-codes bridge names that the isolation test cannot detect

Status: ✅ FIXED 2026-10-22 — neutral blocks_delete key (documented on EntityIntegration::row) and entity_related alias; projects_exists/entity_projects added to PackageIsolationTest FORBIDDEN
Severity: MEDIUM
Category: Architecture (package boundaries)
File: packages/entities13/src/Transformers/EntityListTransformer.php
Line: 19 (also packages/entities13/src/Queries/Entities/EntityListQuery.php:162)
Confidence: HIGH

Problem: `BLOCKS_DELETE = 'projects_exists'` and the alias `'entity_projects'` name the projects domain inside entities13. The bridge ignores the alias it receives (`EntityCountsIntegration.php:23, 37, 59`).

Evidence: `PackageIsolationTest` FORBIDDEN (`packages/entities13/tests/Unit/PackageIsolationTest.php:30-47`) matches namespaces, `PRO_*` tables and `projects.*` routes only. AGENTS.md says coupling includes payload keys.

Impact: hidden coupling. If the bridge renames its key, archive-instead-of-delete silently stops working.

Recommendation: use a neutral key (e.g. `blocks_delete`), returned by the bridge's `row()`. Use or drop the alias parameter, and add both strings to FORBIDDEN.

### TEST-001 — MariaDB-only invariants never run in CI, and package CI ignores `develop`

Status: ✅ FIXED 2026-10-22 — package CI runs on develop; basics13/entities13/bridge workflows get a MariaDB service and run #[Group('mariadb')] (15 tests, all passing locally on MariaDB); entities13 TestCase honours DB_CONNECTION. Remaining: sibling checkouts use the default branch
Severity: MEDIUM
Category: Testing
File: packages/basics13/.github/workflows/tests.yml
Line: 67-73 (triggers at the top of each package workflow)
Confidence: HIGH

Problem: the 15 skipped tests are the only real-engine checks of the unique-name index, the accent and case collation, and the row-lock guards. The package workflows use SQLite. The demo13 CI has MariaDB but runs only the 10 app tests. Package workflows also trigger only on `master` pushes, PRs and manual dispatch, while the work happens on `develop`.

Evidence: the skip messages (`markTestSkipped`) and `on: push: branches: [master]` in all four package workflows.

Impact: core spec invariants in `basics-trash-archived` are verified only against simulated exceptions, and pushes to `develop` (which carry SEC-001) are never tested.

Recommendation: add a `mariadb:11.7` service with `BASICS13_MARIADB_DATABASE` to the package workflows that own these tests (copy the demo13 pattern), and add `develop` to the push branches.

### TEST-002 — Filter behaviour is tested only for entities

Status: ✅ FIXED 2026-10-22 — project and epic filter feature tests: date range, is_empty, project name, archived and trash scopes, search plus filters, pagination keeps filters
Severity: MEDIUM
Category: Testing
File: packages/entities13/tests/Feature/Entities/EntityListFilterTest.php
Line: 1 (sole sibling)
Confidence: MEDIUM

Problem: there are no filter feature tests for projects (including the bridge `entity_id` `is_empty` and multi-select filter), epics or entity types. Also untested: the archived and trash scopes, search combined with filters, and pagination keeping `filters`.

Evidence: `openspec/changes/add-advanced-search-modal-to-lists/tasks.md:76-92` is still unchecked.

Impact: BUG-001 to BUG-005 were not caught.

Recommendation: add one feature test per list while finishing the change, and extend `ResourceUniformityTest` list-request tests with a filters case.

## Low Findings

Each line gives: ID, problem, file:line, then the recommendation. Severity is LOW for all.

- [✅ FIXED 2026-10-21 — malformed values dropped in request; undeclared operators / wrong shapes skipped] **BUG-007 — Filter values that are not scalars cause a 500, and operators are not validated.** `packages/basics13/src/Http/Requests/SearchableListRequest.php:17-20`; `ListQueryBase.php:199`; `page-header.blade.php:67`. Validate `filters.*.value` as a string or a list of scalars, and skip any operator not in the field's `operators` list. That check also resolves the badge/query divergence. Confidence: HIGH.
- [✅ FIXED 2026-10-21 — condition kept (spec MUST) but evaluated once in mount; failing fields dropped] **BUG-008 — `condition` never hides anything and is a latent bug.** No list route has `{entity}` or `{project}`, yet `entity-projects13/src/ServiceProvider.php:118` and `projects13/src/Queries/Epics/EpicListQuery.php:187` define conditions on them. A field hidden by `condition` would reappear under "show advanced" (`AdvancedSearchModal.php:92-100, 299-302`). Remove `condition` entirely. This also shrinks SEC-001 and OPS-001. Confidence: HIGH.
- [✅ FIXED 2026-10-22 — envelope narrowed flag (search or filters): no-match message and no create shortcut; tests] **BUG-009 — A list emptied by filters shows the "no records yet" message and the create CTA.** `packages/basics13/src/Transformers/ListTransformer.php:119`; `packages/entities13/resources/views/entities/list.blade.php:127`. Also consider filters, not only `search`. Confidence: HIGH.
- [✅ FIXED 2026-10-22 — trashed type/project after validation → ValidationException on entity_type_ids / project_id (redundant nested transaction removed); tests] **BUG-010 — Concurrent-race paths return 500 or 404 instead of a field error.** `packages/entities13/src/Http/Controllers/EntityController.php:113-127` (`RuntimeException`, plus a redundant nested transaction); `packages/projects13/src/Http/Controllers/EpicController.php:91, 109`. Throw `ValidationException::withMessages(...)`. Confidence: MEDIUM.
- [⏸️ NOT CHANGED 2026-10-22 — the change spec (specs/basics-crud-base, entities-domain, projects-domain) MUSTs an active boolean field on every scope; needs a spec change first] **BUG-011 — The boolean `active` filter is offered on lists already split by `active`.** `EntityListQuery.php`, `ProjectListQuery.php`, `EpicListQuery.php` (the `active` field). Offer it only in the trash scope, or drop it. Confidence: HIGH.
- [⏸️ PENDING DECISION — the epics-comments spec does not say whether moving a project's dates must respect its epics; clarify the spec before coding] **BUG-012 — Moving a project's dates can leave its epics outside the range.** `packages/projects13/src/Http/Requests/ProjectRequest.php:73-74`. The spec intent is ambiguous, so clarify `epics-comments` before coding. Confidence: MEDIUM.
- [✅ FIXED 2026-10-22 — no-op booted() delete hooks removed from Epic and Project; provider guards remain the single source] **DB-002 — Model `deleting` hooks call `lockForUpdate()` without executing a query.** `packages/projects13/src/Models/Epic.php:85`; `Project.php:87`. The comments claim TOCTOU protection, but the real guards are in `projects13/src/ServiceProvider.php:47-66`. Delete both `booted()` hooks. Confidence: HIGH.
- [✅ FIXED 2026-10-22 — epic guard counts trashed comments; test] **DB-003 — The epic delete guard ignores trashed comments.** `packages/projects13/src/ServiceProvider.php:64`. This is latent: nothing soft-deletes comments yet. Use `comments()->withTrashed()->exists()`. Confidence: MEDIUM.
- [✅ FIXED 2026-10-22 — comments read authorizes viewAny; 403 test] **SEC-003 — `GET epics/{epic}/comments` has no authorization call.** `packages/projects13/src/Http/Controllers/EpicCommentController.php:16`. Add `authorize('viewAny', Epic::class)` before roles land. Confidence: HIGH.
- [🟡 PARTIAL 2026-10-22 — join aliases neutral (project/epic/planning/timeline_related) and hasEntities→hasRelated; data-test names kept (views rule) and contract method hasActiveRelatedEntities kept (contract rename not approved)] **ARCH-002 — projects13 names entities, and the bridge duplicates projects13's private aliases.** `ProjectListQuery.php:26`; `EpicListQuery.php:29, 234`; `ProjectController.php:36`; `EntityProjectsIntegration.php:86, 218-226`. Use neutral names, and pass the alias to `searchColumns()`. Confidence: HIGH.
- [✅ FIXED 2026-10-22 — basics13 findTrashedByName() domain-neutral (+ trashedQuery()); EpicListQuery overrides it with the project scope; tests moved to projects13] **ARCH-003 — Epic-specific `project_id` in domain-neutral basics13.** `packages/basics13/src/Queries/ListQueryBase.php:398-417`. Override using the existing `firstTrashedByName()` in `EpicListQuery`. Confidence: HIGH.
- [✅ FIXED 2026-10-22 — label props/attributes, unused grouping methods, $filtersB64 and dead ?? fallbacks removed] **MAINT-001 — Dead and wrong label plumbing in `AdvancedSearchModal`.** The whole `__('basics13::messages')` array is passed as `operatorLabels`, so `is_empty` shows "(empty)". The props are protected and lost after hydration, the grouping methods are unused, `$filtersB64` is unused, and `__() ?? …` fallbacks never apply. `AdvancedSearchModal.php:33-40, 129-139, 155, 165-168`; `entities/list.blade.php:165-167`; `page-header.blade.php:57-58, 84`. Delete the three props and the unused code. Confidence: HIGH.
- [✅ FIXED 2026-10-22 — ARCHITECTURE (user active flag, account holder rule, comment FKs, transactions, PHP 8.4, contract decision), models.instructions layer 7 and ArchivedController docblocks corrected] **MAINT-002 — Docs contradict the code.**
  - `ARCHITECTURE.md` lines 54-56 and 99-103 say users' `active` is never written (but `UserArchivedController` exists), that comment authors are nulled and comments cascade (both FKs are RESTRICT), and that there are no multi-step transactions.
  - Line 176 says CI runs PHP 8.3.
  - `ArchivedController.php:24, 34, 37-45` docblocks describe the redirects inverted.
  - The `models.instructions.md` "layer 7" claims something about TracksAuditColumns that is not true.

  Fix the docs only. Confidence: HIGH.
- [⏸️ ACCEPTED 2026-10-22 — acceptable at current data sizes (options now locked server-side by SEC-001); revisit with the capped entities.options endpoint when entities grow] **PERF-002 — The advanced search modal loads every entity into the public `$fields` property and the Livewire snapshot.** `AdvancedSearchModal.php:22, 304-306`; `entity-projects13/src/ServiceProvider.php:113-117`. Acceptable at current sizes. Revisit when entities grow, reusing the capped `entities.options` endpoint. Confidence: HIGH.
- [✅ FIXED 2026-10-22 — user options tests assert exact active ids and order; 38 dead @requires docblocks removed] **TEST-003 — Assertions that prove less than their names claim.** In `UserSelectOptionsQueryTest.php:63-72, 88-101`, 30 active users fill the cap of 25, so the active filter is unproven, and the ordering-test users are unused. Doc-comment `@requires` lines are ignored by PHPUnit 12. Use small fixtures and exact ids. Confidence: HIGH.
- [✅ FIXED 2026-10-22 — value inputs and selects carry the field label (from/to on ranges); render test] **FE-001 — Filter value inputs have no accessible name.** `advanced-search-modal.blade.php:54-56, 79-101`. Add `aria-label` (and "from/to" on range inputs). Not checked in a browser. Confidence: MEDIUM.
- [✅ FIXED 2026-10-22 — advanced toggle has aria-expanded; render test] **FE-002 — The advanced toggle has no `aria-expanded`.** `advanced-search-modal.blade.php:28-32`. Confidence: HIGH.
- [✅ FIXED 2026-10-22 — bridge eu.json uses Bezeroa and valid Basque] **FE-003 — Wrong Basque copy in the default locale.** `packages/entity-projects13/resources/lang/eu.json:2-3, 6-9` uses "atzilea" instead of "Bezeroa", and "Bezeroak ez direkitut" is not valid Basque. Use "Bezeroa"; for line 6, "Ez da bezerorik aurkitu." Confidence: HIGH.
- [✅ RECORDED 2026-10-22 — accepted as known debt in ARCHITECTURE.md] **OPS-004 — Host CI tests packages at a moving HEAD, and package pushes never run Dusk.** `projects/demo13/.github/workflows/tests.yml:89-115, 222-248`. Record it as known debt in ARCHITECTURE.md. Confidence: HIGH.
- [🟡 PARTIAL 2026-10-22 — laravel/boost moved out of basics13 require into demo13 require-dev (locks updated, boost v2.10.3 kept, boost:* commands OK); laravel/chisel kept in require: InstallFeaturesCommand uses it at runtime] **OPS-005 — Boost (MCP server) and Chisel 0.x are in basics13 `require`, so they ship with `--no-dev`.** `packages/basics13/composer.json:9, 12`. Moving them to `require-dev` is a dependency change, so it needs approval. Confidence: HIGH.
- [⏸️ NOT CHANGED 2026-10-22 — owner decision: docker-compose.yml stays as is; docker-commands skill now documents that composer/npm-all delete and regenerate the lock files] **OPS-006 — Compose helper profiles delete the lockfiles.** `projects/demo13/docker-compose.yml:78-82, 137-141`. These are dev helpers; rename them so they say they regenerate, or install from the lock. Confidence: HIGH.

## Informational Findings

- **Advanced-search option scoping by the referenced record's own `active` flag.** On the archived and trash lists, active entities and types cannot be chosen. This was intended by `openspec/changes/add-advanced-search-modal-to-lists/design.md:72, 92`, so it is a product question, not a defect.
- **`composer.json` declares `php: ^8.3`, but only 8.4 is tested.** Narrow the constraint or add a CI leg (see MAINT-002).
- **The previous review (2026-10-19) is partly stale.** CRITICAL-001 (EpicComment traits) is resolved. It cites test files that do not exist (`entity-projects13/tests/Feature/Seeders/PackageSeederTest.php`, `entities13/tests/Feature/EntityUniqueIndexTest.php`).

## Security

- **SEC-001 (CRITICAL)**: remote code execution through Livewire state.
- **SEC-002 (HIGH)**: account takeover through registration plus peer password change. It also covers log-viewer access by email.
- **SEC-003 (LOW)**: the comments read endpoint has no authorization call.
- **BUG-007**: hardens filter input.

Checked and fine:
- No SQL injection: filter columns come from the server and values are bound.
- No unescaped Blade output in the reviewed views, and `innerHTML` is used only on same-origin fragments.
- CSRF protection through the `web` middleware group.
- Login, 2FA and passkey throttling are in place.
- Inactive users are blocked at login and by middleware.
- `active` is not mass assignable.
- Comment author comes from auth.
- Trusted hosts are set.
- `composer audit` is clean.

## Bugs / Correctness

BUG-001 to BUG-012. Most of them are in the advanced-search change and are cheap to fix before it is finished.

## Database

DB-001 (MEDIUM), DB-002 (LOW), DB-003 (LOW).

Checked and fine:
- A generated `active_name` column with unique indexes, plus a SQLite/Postgres partial index.
- Restore and force-delete conflict handling.
- Parent guards lock the parent and count trashed children.
- Consistent parent-then-child lock order.
- All FKs are RESTRICT and indexed.
- No destructive migrations.
- The live schema matches the migrations (checked through Boost).

## Performance

PERF-001 (MEDIUM), PERF-002 (LOW). The entity, project and comment lists aggregate in SQL. Select-option endpoints cap at 25. `stateCounts` reuses the paginator total (but see BUG-005).

## Architecture

- **ARCH-001 (MEDIUM)**: entities13 hard-codes bridge names.
- **ARCH-002, ARCH-003 (LOW)**: domain names and project scoping in the wrong package.
- **OPS-001**: also an architecture concern; the extension point should be a contract method, not closures in config.

Checked and fine:
- Bridge dependency direction is correct.
- Relations are resolved at runtime in the bridge.
- No speculative layers.
- Controllers are thin.

## Testing

TEST-001 and TEST-002 (MEDIUM), TEST-003 (LOW), plus OPS-003, which is about the red PHPStan. The suites are large and pass. The gaps are the real database engine, CI coverage of the `develop` branch, and coverage of the new filter feature.

## Production

- **OPS-001 (HIGH)**: `config:cache` fails.
- **OPS-002 (MEDIUM)**: entities13 composer path change.
- **OPS-003 (MEDIUM)**: package PHPStan gates are red.
- **OPS-004, OPS-005, OPS-006 (LOW).**

Checked and fine:
- Actions are pinned by SHA with read-only permissions.
- `.env` and `dc-data` are gitignored.
- `APP_DEBUG=false` by default.
- Log viewer is gated in production.
- DB, phpMyAdmin and MailHog bind to 127.0.0.1.

## Maintainability

MAINT-001 and MAINT-002 (LOW). Also see BUG-008: `condition` is a feature that hides nothing today and should be removed rather than fixed.

## Rejected Findings

| Source | Reason |
|---|---|
| LAR-006, BUG-005, FE-007 (option scoping) | Intended by the change's design.md, so a product question (listed under Informational) |
| PERF-005 (entity list join + DISTINCT) | Not confirmed with EXPLAIN; speculative |
| PERF-006 (duplicate exists query) | Negligible impact |
| MAINT-007 ("scope" naming) | Style |
| MAINT-006, config-hook part | The intended bridge extension point |
| ARCH-004 (shared "is active filter" predicate) | Unneeded abstraction; operator validation (BUG-007) removes the divergence |
| ARCH-005, `'names'` group part | Generic word, not domain knowledge |
| TEST-006, fixture-array and unused-fixture parts | Style |
| API-007 (response shape differences) | The specialist itself recommended leaving it as is |
| API-008 (email enumeration in user options) | Speculative until roles restrict user listing |
| API-009 / FE-009 | Merged into PERF-002 / SEC-001 |
| OPS-008 (`.env.example` profile) | Local-dev file; preference |
| OPS-009 (`concurrently`, select2 with jQuery 4) | Trivial, and the incompatibility is unsupported |
| OPS-010 (starter-kit name, `"*"` constraints) | Cosmetic; `"*"` on path repositories is intended |
| SEC-006, OPS-007, OPS-011 (0.0.0.0 ports, dev passwords, Dusk image) | Local-dev only, documented in ARCHITECTURE "Production Assumptions" |
| FE-011, server round-trip part | Normal Livewire; preference |
| Cross-repo CI dispatch (OPS-002 original recommendation) | Adds complexity; recorded as debt instead (OPS-004) |

## Action Plan

Ordered by impact. Each step is the smallest change that removes the problem.

1. **Now, before anything reaches `master`:**
   - Fix SEC-001: stop invoking callables in `getVisibleFields()` and `getSelectOptions()`, add `#[Locked]` to `$fields` and `$scope`, and add a tamper regression test.
   - Remove `condition` everywhere (BUG-008).
2. **Now:** set `registration` to false in the host config (SEC-002). Then restrict password and email changes through `UserRequest` to the user's own account.
3. **Before the next deploy:** move the bridge's searchable field out of config through the existing `ProjectQueryIntegration` seam (OPS-001, contract change: ask first). Add a test that serializes `config()->all()`.
4. **Before committing package work:**
   - Revert the entities13 composer path and lock (OPS-002).
   - Unstage and regenerate the entities13 PHPStan baseline, and fix the 3 basics13 test errors (OPS-003).
5. **CI:** add `develop` to the package workflow push branches, and add a MariaDB service for the 15 MariaDB-only tests (TEST-001).
6. **Finish the advanced-search change:**
   - Fix BUG-001 (labels), BUG-002 (URL state), BUG-003 (date operator), BUG-004 (LIKE escape), BUG-005 (state counts) and BUG-007 (input validation).
   - Write the filter tests from TEST-002, and clean up MAINT-001.
7. **Fix DB-001:** put the comment-store lock, check and save in one transaction.
8. **Fix BUG-006:** add `before_or_equal:<project end>` to the epic `start_date`.
9. **Fix PERF-001:** pass `$project` into `TimelineQuery::epicRow()`, and add `project.entity` to the bridge's epic eager load.
10. **Fix ARCH-001:** use neutral key and alias names in entities13, and extend the FORBIDDEN list.
11. **Low-cost cleanups:** DB-002, DB-003, SEC-003, BUG-009 to BUG-011, ARCH-002, ARCH-003, TEST-003, FE-001 to FE-003, and the MAINT-002 doc fixes.
12. **Needs a decision:** BUG-012 (spec clarification), OPS-005 (dependency move), the option-scoping product question, and OPS-004/OPS-006 (record as debt or rename).

# Decisiones a corregir despues del Review
[x] 1. no quiero usar BASICS13_REGISTRATION, en config basics13 por defecto true, y si en la aplicacion quiero cambiarlo, usare el config, no el .env
[x] 2. no quiero cambiar el docker-compose del proyecto, deshaz los cambios y adecua el skill docker-commands a lo que tengo en el docker-compose.yml
[x] 3. cambia las versiones de php y en todas utiliza 8.5
4. para laravel 13, qué version de imagen de node me aconsejas? ahora uso 24-bullseye, pero hay alguna más nueva? con la 26 funcionaría bien la aplicación?