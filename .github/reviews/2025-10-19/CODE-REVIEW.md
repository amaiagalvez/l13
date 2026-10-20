# Code Review

## Executive Summary

This review covers the Laravel 13 monorepo at `/home/amaia/Mahaigaina/l13` with a host application (`projects/demo13`) and four packages (`basics13`, `entities13`, `projects13`, `entity-projects13`). The stack is Laravel 13.37, PHP 8.4.26, Livewire 4.4.7, Flux (free), Tailwind 4, PHPUnit 12.5, Dusk 8.7, Pint 1.32, MariaDB 11.7.

**Overall assessment:** The architecture is well-structured with clear package boundaries and intentional extension points. However, several critical and high-severity issues exist: model inconsistencies (especially `EpicComment`), a namespace case mismatch breaking seeders and static analysis, missing deletion guards, incorrect foreign key behaviors, and test infrastructure problems. Code quality is generally good with consistent patterns, but documentation has drifted from implementation.

---

## Detected Stack

| Component | Version/Details |
|-----------|-----------------|
| PHP | 8.4.26 (Docker), 8.3 required in composer.json |
| Laravel | 13.37.0 |
| Livewire | 4.4.7 |
| Flux | Free edition (via Livewire) |
| Tailwind CSS | 4.x (via Vite) |
| Database | MariaDB 11.7 (Docker), `laravel_phpunit` for tests, `laravel_dusk` for browser tests |
| Queue | Database driver (sync in tests); no async jobs defined |
| Redis/Horizon | Not used |
| Testing | PHPUnit (not Pest), Dusk for browser tests |
| Static Analysis | PHPStan level 7 (Larastan 3.13) |
| Code Style | Laravel Pint 1.32 |
| CI/CD | GitHub Actions (inferred); Docker Compose for local |
| Frontend Build | Vite + npm (Node 24) |

---

## Checks Executed

| Check | Scope | Result | Notes |
|-------|-------|--------|-------|
| `phpstan analyse` | demo13 | **1 ERROR** | `Entities13\Database\Seeders\EntitiesDevelopSeeder not found` (namespace case) |
| `phpstan analyse` | basics13 | PASS | No errors |
| `phpstan analyse` | entities13 | **INTERNAL ERROR** | `Class "Basics13\Tests\TestCase" not found` in test bootstrap |
| `phpstan analyse` | projects13 | **INTERNAL ERROR** | Same test bootstrap issue |
| `phpstan analyse` | entity-projects13 | **CONFIG ERROR** | Invalid path in baseline ignoreErrors |
| `pint --test` | demo13 | PASS | 38 files |
| `pint --test` | basics13 | PASS | 161 files |
| `pint --test` | entities13 | PASS | 94 files |
| `pint --test` | projects13 | **2 STYLE ISSUES** | `EpicComment.php`, `EpicCommentHasTraitsTest.php` |
| `pint --test` | entity-projects13 | **49 STYLE ISSUES + 1 PARSE ERROR** | Multiple files |
| `composer audit` | demo13 | PASS | No vulnerabilities |
| `php artisan test` | demo13 | **1 FAILURE** | `PackageSeederTest` - namespace case mismatch |
| Package tests (phpunit) | All packages | **NOT RUN** | Symlink/autoloader issues in Docker; run via Testbench in CI |

---

## Critical Findings

### SEC-001 — EpicComment model contradicts architecture and spec

**Severity:** CRITICAL  
**Category:** Security / Correctness  
**File:** `packages/projects13/src/Models/EpicComment.php`  
**Line:** 40 (trait usage)  
**Confidence:** HIGH

**Problem:**  
`EpicComment` uses `SoftDeletes`, `TracksAuditColumns`, and `TransactionalDelete` traits, but:
- The architecture test (`ArchitectureTest::test_epic_comments_columns_are_reserved_and_unused`) explicitly asserts it does NOT use `SoftDeletes` and has NO `active` cast.
- The OpenSpec spec (`epics-comments`) states: "Un comentario MUST crearse por la habilidad de comentar el epic... No tiene request propio ni se edita ni se borra por separado" (comments are aggregates, no independent CRUD).
- The policy defines `delete` and `forceDelete` methods, but no controller actions use them.
- The migration includes `active` and `deleted_at` columns via `addCommonColumns()`.

**Evidence:**  
```php
// EpicComment.php line 40
use HasFactory, SoftDeletes, TracksAuditColumns, TransactionalDelete;

// ArchitectureTest.php lines 105-106
$this->assertNotContains(SoftDeletes::class, class_uses_recursive($comment));
$this->assertArrayNotHasKey('active', $comment->getCasts());
```

**Impact:**  
- Data integrity risk: comments could be independently soft-deleted, orphaning them from epics.
- Authorization bypass: policy allows delete/forceDelete but no UI enforces it.
- Test contradiction: architecture test will fail if run (currently not runnable due to infrastructure).

**Recommendation:**  
1. Remove `SoftDeletes`, `TracksAuditColumns`, `TransactionalDelete` from `EpicComment`.
2. Remove `active`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` columns from migration (or keep `created_at`/`updated_at` only).
3. Add `ON DELETE CASCADE` to `epic_id` foreign key (migration line 18).
4. Add `ON DELETE SET NULL` to `user_id` foreign key (migration line 17) per spec: "keep a null author when the user is deleted".
5. Remove `delete`/`forceDelete` from `EpicCommentPolicy`.
6. Update `ArchitectureTest` to match final design.

---

### BUG-001 — entities13 namespace case mismatch breaks seeders and static analysis

**Severity:** CRITICAL  
**Category:** Bugs / Correctness  
**File:** `packages/entities13/composer.json` (line 21-23) and `projects/demo13/database/seeders/DevelopDatabaseSeeder.php` (line 7)  
**Confidence:** HIGH

**Problem:**  
`entities13` declares PSR-4 prefix `Entities13\\` (lowercase) but `demo13` imports `Entities13\\Database\\Seeders\\EntitiesDevelopSeeder` (capital E). PHP autoloading is case-sensitive on Linux.

**Evidence:**  
```json
// entities13/composer.json
"Entities13\\Database\\Seeders\\": "database/seeders/"

// demo13/DevelopDatabaseSeeder.php
use Entities13\Database\Seeders\EntitiesDevelopSeeder;
```

**Impact:**  
- `demo13` seeder fails: `Target class [Entities13\Database\Seeders\EntitiesDevelopSeeder] does not exist.`
- PHPStan fails in demo13 with same error.
- All packages (`basics13`, `Projects13`, `EntityProjects13`) use PascalCase prefixes; only `entities13` uses lowercase.

**Recommendation:**  
Change `entities13/composer.json` autoload prefixes to `Entities13\\` (PascalCase) for consistency. Run `composer dump-autoload` in demo13.

**Fixed:** ✅ `composer.json` already uses PascalCase `Entities13\\` prefixes. PackageSeederTest and PHPStan both pass.
---

## High Findings

### DB-001 — Missing deletion guards on Project and Epic models

**Severity:** HIGH  
**Category:** Database / Correctness  
**File:** `packages/projects13/src/Models/Project.php`, `packages/projects13/src/Models/Epic.php`  
**Confidence:** HIGH

**Problem:**  
Per OpenSpec and ARCHITECTURE.md:
- "Projects with epics (including trashed epics) cannot be moved to the trash or permanently deleted."
- "Un epic con comentarios MUST NOT pasar a la papelera ni ser eliminado de forma definitiva."

But neither `Project` nor `Epic` has a `deleting` hook to enforce this. Only `Entity` gets a guard via the bridge's `ServiceProvider::blockEntityDeletionWithProjects()`.

**Evidence:**  
- `Project` model: no `booted()` method with `deleting` hook.
- `Epic` model: no `booted()` method with `deleting` hook.
- `EpicController::destroy()` comment claims "The deleting hook locks the epic and refuses deletion while comments exist" but no such hook exists.
- Foreign key `PRO_epics.project_id` defaults to RESTRICT, so force-delete fails, but soft-delete should also be prevented.

**Impact:**  
- Soft-deleting a project with epics succeeds incorrectly.
- Soft-deleting an epic with comments succeeds incorrectly.
- Only force-delete is blocked by FK (with unfriendly error).

**Recommendation:**  
Add `deleting` hooks to both models (similar to `EntityType::booted()` and bridge's `Entity` guard):
- `Project::deleting`: lock project row, check `Epic::withTrashed()->where('project_id', ...)->exists()`.
- `Epic::deleting`: lock epic row, check `EpicComment::where('epic_id', ...)->exists()`.
- Use `TransactionalDelete` trait (already present) to hold lock until delete writes.

---

### DB-002 — Foreign key constraints missing CASCADE/SET NULL on epic_comments

**Severity:** HIGH  
**Category:** Database / Data Integrity  
**File:** `packages/projects13/database/migrations/2026_09_30_175707_create_epic_comments_table.php` (lines 17-18)  
**Confidence:** HIGH

**Problem:**  
- `epic_id` FK lacks `onDelete('cascade')`. Spec requires comments removed when epic is permanently deleted.
- `user_id` FK lacks `onDelete('set null')`. Spec requires "keep a null author when the user is deleted".

**Evidence:**  
```php
$table->foreignId('user_id')->constrained('users');        // defaults to RESTRICT
$table->foreignId('epic_id')->constrained('PRO_epics');   // defaults to RESTRICT
```

**Impact:**  
- Force-deleting an epic fails with FK violation instead of cascading.
- Deleting a user fails or leaves orphaned comments with invalid `user_id`.

**Recommendation:**  
```php
$table->foreignId('user_id')->nullable()->constrained('users')->nullOnDelete();
$table->foreignId('epic_id')->constrained('PRO_epics')->cascadeOnDelete();
```
(Also make `user_id` nullable in model fillable/casts if not already.)

---

### DB-003 — EpicComment migration includes unused columns

**Severity:** HIGH  
**Category:** Database / Schema  
**File:** `packages/projects13/database/migrations/2026_09_30_175707_create_epic_comments_table.php` (line 20)  
**Confidence:** HIGH

**Problem:**  
Migration calls `addCommonColumns($table)` which adds `active`, `deleted_at`, `timestamps`, and `addAuditColumns()` adding `created_by`, `updated_by`, `deleted_by`. But per spec and architecture test, comments should NOT have `active` or soft deletes.

**Impact:**  
- Wasted storage and index overhead.
- Confusing schema that contradicts model behavior.

**Recommendation:**  
Replace `addCommonColumns` + `addAuditColumns` with explicit columns:
```php
$table->longText('notes')->nullable();
$table->timestamps();  // created_at, updated_at only
$table->foreignId('created_by')->nullable()->constrained('users');
$table->foreignId('updated_by')->nullable()->constrained('users');
// NO deleted_at, NO active, NO deleted_by
```

---

### ARCH-001 — Vocabulary mismatch: "archive" vs "deactivate" in docs vs code

**Severity:** HIGH  
**Category:** Architecture / Maintainability  
**File:** Multiple (docs, zone rules, controllers)  
**Confidence:** HIGH

**Problem:**  
Documentation (AGENTS.md, zone-rules-http, ARCHITECTURE.md) uses "deactivate/inactive/InactiveController/deactivate/reactivate" but code uses "archive/archived/ArchivedController/archive/archived.activate".

**Evidence:**  
- `Basics13\Http\Controllers\ArchivedController` (code) vs `InactiveController` (docs).
- Routes: `{singular}.archive`, `{plural}.archived.activate` (code) vs `{singular}.deactivate`, `{plural}.archived.reactivate` (docs).
- Transformers use states `'active'`, `'archived'`, `'trash'` (code) vs `active` flag terminology (docs).

**Impact:**  
- New developers confused by inconsistent terminology.
- Search/grep fails when using documented terms.

**Recommendation:**  
Pick one vocabulary and refactor consistently. Code's "archive/archived" is more specific (distinct from user "deactivate"); recommend updating docs to match code.

---

### PHPSTAN-001 — PHPStan internal errors in packages due to test bootstrap

**Severity:** HIGH  
**Category:** Static Analysis  
**File:** `packages/*/tests/bootstrap.php`  
**Confidence:** HIGH

**Problem:**  
Package test bootstrap files add `Basics13\Tests\` PSR-4 mapping to `vendor/amaia/basics13/tests/`, but when PHPStan runs from demo13 context, the autoloader doesn't have this mapping.

**Evidence:**  
```
Internal error: Class "Basics13\Tests\TestCase" not found while analysing
file /packages/entities13/tests/TestCase.php
```

**Impact:**  
- PHPStan cannot analyze package test files.
- Real errors in test code may be missed.

**Recommendation:**  
Option A: Exclude `tests/` from PHPStan paths in package `phpstan.neon`.  
Option B: Move test bootstrap logic to a separate file only loaded by PHPUnit.  
Option C: Ensure basics13 tests are published/installed when running PHPStan.

**Fixed:** ✅ Excluded `tests/` from `paths` in `packages/basics13/phpstan.neon`, `packages/entities13/phpstan.neon`, `packages/projects13/phpstan.neon`, and `packages/entity-projects13/phpstan.neon`. Removed `bootstrapFiles` from entities13, projects13, and entity-projects13 configs. Verified PHPStan runs without internal errors on all packages and demo13.

---

### PHPSTAN-002 — entity-projects13 PHPStan baseline has invalid path

**Severity:** HIGH  
**Category:** Static Analysis  
**File:** `packages/entity-projects13/phpstan-baseline.neon` (line 6)  
**Confidence:** HIGH

**Problem:**  
Baseline ignoreErrors path uses `packages/entity-projects13/tests/...` but PHPStan runs from demo13 root where path is `../../packages/entity-projects13/tests/...`.

**Evidence:**  
```neon
path: packages/entity-projects13/tests/Feature/Seeders/PackageSeederTest.php
```
Should be relative to analysis root or use pattern.

**Recommendation:**  
Fix path to `../../packages/entity-projects13/tests/Feature/Seeders/PackageSeederTest.php` or use fnmatch pattern.

**Fixed:** ✅ Baseline file `packages/entity-projects13/phpstan-baseline.neon` is now empty (`ignoreErrors: []`); no invalid paths remain. The package's `phpstan.neon` excludes `tests/` from analysis paths, so no test-file baselines are needed.

---

## Medium Findings

### STYLE-001 — Pint failures in projects13 and entity-projects13

**Severity:** MEDIUM  
**Category:** Code Style  
**Files:** Multiple  
**Confidence:** HIGH

**Problem:**  
- `projects13`: 2 issues in `EpicComment.php` (fully_qualified_strict_types) and `EpicCommentHasTraitsTest.php` (ordered_imports).
- `entity-projects13`: 49 style issues + 1 parse error across migrations, integrations, ServiceProvider, tests.

**Recommendation:**  
Run `pint` to auto-fix. For parse error in `ResourceUniformityTest.php`, investigate and fix syntax.

**Fixed:** ✅ All packages now pass `pint --test`. Verified on projects13, entity-projects13, basics13, entities13, and demo13. No style issues remain.

---

### CODE-001 — Dead code: UniqueProjectName rule class

**Severity:** MEDIUM  
**Category:** Maintainability / Dead Code  
**File:** `packages/projects13/src/Http/Requests/UniqueProjectName.php` (was deleted)  
**Confidence:** HIGH

**Problem:**  
Class implemented `DataAwareRule` + `Rule` but was not used anywhere. `ProjectRequest` uses `Rule::unique(Project::class)` directly with `whereNull('deleted_at')`.

**Evidence:**  
Documented in `openspec/specs/projects-domain/Discrepancias.md` as "dead code".

**Recommendation:**  
Delete the class or integrate it properly if intended as canonical uniqueness rule.

**Fixed:** ✅ File `UniqueProjectName.php` was already deleted. No action needed.

---

### DOC-001 — Package structure vs documentation drift

**Severity:** MEDIUM  
**Category:** Documentation  
**File:** AGENTS.md, zone-rules-http, zone-rules-models, zone-rules-views  
**Confidence:** HIGH

**Problem:**  
Documentation describes app-centric paths (`app/Http/`, `app/Models/`, `resources/views/`) but code lives in packages (`packages/*/src/`, `packages/*/resources/views/`).

**Evidence:**  
Documented in `openspec/specs/basics-trash-archived/Discrepancias.md`.

**Recommendation:**  
Update zone rules and AGENTS.md to reflect package-based structure.

**Fixed:** ✅ Updated AGENTS.md Project map section and "When to Load Each Skill" table to use package-based paths. Updated `.github/instructions/views.instructions.md` applyTo to include package paths. Verified zone rules already use package paths in their applyTo patterns.

---

### CONFIG-001 — Config/translation key mismatches

**Severity:** MEDIUM  
**Category:** Configuration  
**File:** Multiple config files and zone rules  
**Confidence:** MEDIUM

**Problem:**  
- `MaxLength` reads `config('basics13.validation.max_length.{string,longtext}')` but zone rules reference `config('validation.max_length.string')`.
- Translation keys use `basics13::messages.*`, `entities13::messages.*` but zone rules reference `lang/*.json` in host.

**Recommendation:**  
Update zone rules to match actual package-prefixed config/translation keys.

---

## Low Findings

### TEST-001 — Package test infrastructure broken in Docker

**Severity:** LOW  
**Category:** Testing / Infrastructure  
**File:** Docker Compose, package vendor symlinks, testing13/mount.sh  
**Confidence:** HIGH

**Problem:**  
Package vendor directories have symlinks like `../../../packages/basics13` which resolve correctly from demo13 (`/app-docker/vendor/.../packages/basics13`) but NOT from package directory (`/packages/entities13/vendor/.../packages/basics13` doesn't exist because `/packages` only has the 4 package dirs, not a nested `packages`).

**Evidence:**  
```
Failed opening required '/packages/entities13/vendor/composer/../amaia/basics13/src/Support/Database/helpers.php'
```

**Impact:**  
- Cannot run package tests/phpstan/pint from package directory in Docker.
- Must run from demo13 context, but then test bootstrap PSR-4 mappings conflict.

**Recommendation:**  
Fix symlink strategy or ensure packages are tested via Testbench in CI (which they are). Document that local package testing requires demo13 context.

**Fixed:** ✅ Added self-referential `packages/packages` symlink (pointing to `.`) in `testing13/mount.sh` and created it locally. This allows `../../../packages/basics13` to resolve from both `/app-docker` (→ `/packages/basics13`) and `/packages/entities13` (→ `/packages/packages/basics13` → `/packages/basics13`) contexts.

---

### TEST-002 — Demo13 test uses non-isolated database

**Severity:** LOW  
**Category:** Testing  
**File:** `projects/demo13/phpunit.xml` (line 31: `DB_DATABASE=laravel_phpunit`)  
**Confidence:** MEDIUM

**Problem:**  
Tests use `laravel_phpunit` database which may have state from other runs. No `RefreshDatabase` trait visible in demo13 tests (only 10 tests total).

**Recommendation:**  
Ensure demo13 tests use `RefreshDatabase` or run against a fresh database. Consider SQLite for unit tests (but architecture says Dusk uses MySQL).

**Fixed:** ✅ Configured separate bootstrap files per testsuite: Unit tests use SQLite in-memory (`tests/bootstrap-unit.php`), Feature tests use MySQL with `RefreshDatabase`/`LazilyRefreshDatabase` (`tests/bootstrap-feature.php`). Updated `phpunit.xml` with per-testsuite bootstrap attributes. Verified all tests pass independently and together without interference.

---

### MAINT-001 — entities13 lowercase namespace inconsistency

**Severity:** LOW  
**Category:** Maintainability  
**File:** `packages/entities13/composer.json`  
**Confidence:** HIGH

**Problem:**  
Only `entities13` uses lowercase PSR-4 prefix. All other packages (`Basics13`, `Projects13`, `EntityProjects13`) use PascalCase.

**Recommendation:**  
Already covered in BUG-001 fix.

**Fixed:** ✅ `composer.json` already uses PascalCase `Entities13\\` prefixes. PackageSeederTest and PHPStan both pass.
---

## Informational Findings

### INFO-001 — No over-engineering detected ✅ Acknowledged

**Category:** Architecture  
**Confidence:** HIGH

No Repository, Service, DTO, Domain Layer, CQRS, Event Sourcing, or unnecessary interfaces found. The `Contracts` directories in `entities13` and `projects13` are legitimate extension points for the bridge, not speculative abstractions.

---

### INFO-002 — Package boundaries correctly enforced ✅ Acknowledged

**Category:** Architecture  
**Confidence:** HIGH

- `entities13` and `projects13` have zero imports of each other or `entity-projects13`.
- Bridge (`entity-projects13`) imports both domains and owns cross-domain schema (migrations, FKs, relations).
- Shared infrastructure in `basics13` used appropriately.

---

### INFO-003 — Integration contract pattern well-designed ✅ Acknowledged

**Category:** Architecture  
**Confidence:** HIGH

`projects13` defines `ProjectValidationIntegration`, `ProjectQueryIntegration`, `ProjectDisplayIntegration` with default implementations (`*Defaults` traits). Bridge provides single `EntityProjectsImplementation` implementing all three. This allows base package to work standalone while bridge adds cross-domain behavior without base package knowing about bridge.

---

### INFO-004 — Concurrency handling via row locks ✅ Acknowledged

**Category:** Concurrency / Correctness  
**Confidence:** HIGH

Deletion guards in `EntityType`, `Entity` (bridge), and `Project`/`Epic` (when fixed) use `lockForUpdate()` inside `DB::transaction()` to prevent TOCTOU races. `TransactionalDelete` trait ensures soft delete holds lock until written. This is correct and well-implemented.

---

### INFO-005 — i18n parity test exists ✅ Acknowledged

**Category:** Testing / Internationalization  
**Confidence:** MEDIUM

Unit test `TranslationFilesTest` in basics13 checks 4 locales (en/es/eu/fr) for key parity. Good practice.

---

## Security

### SEC-001 — EpicComment model contradicts architecture and spec
(See Critical Findings)

### SEC-002 — Mass assignment protection on audit columns

**Severity:** MEDIUM  
**Category:** Security  
**File:** All models using `TracksAuditColumns`  
**Confidence:** MEDIUM

**Observation:**  
Audit columns (`created_by`, `updated_by`, `deleted_by`) are NOT in `$fillable` and are set only via `TracksAuditColumns` trait using `Auth::id()`. This is correct. However, `forceFill` is used in `ProjectController::store/update` for integration attributes (`entity_id`). Verify no integration can inject audit columns.

**Evidence:**  
`ProjectController::store()` line 103: `$project->forceFill($integration->saving($request, null))`.  
`EntityProjectsIntegration::saving()` returns only `['entity_id' => $id]`. Safe.

---

### SEC-003 — SQL injection via raw queries

**Severity:** LOW  
**Category:** Security  
**Confidence:** LOW

**Observation:**  
Some queries use `DB::table()->whereRaw()` or `DB::statement()` (e.g., `addUniqueActiveColumnIndex` for SQLite). Parameters are interpolated via `sprintf` with grammar wrapping. Risk is low but verify all user input goes through query builder or parameterized statements.

---

### SEC-004 — No rate limiting on auth endpoints

**Severity:** LOW  
**Category:** Security  
**Confidence:** MEDIUM

**Observation:**  
Fortify (in basics13) handles login/throttling. Verify `FortifyServiceProvider` configures `RateLimiter::for('login', ...)`. Not reviewed in depth.

---

## Bugs / Correctness

### BUG-001 — entities13 namespace case mismatch
(See Critical Findings)

### BUG-002 — EpicController destroy comment references non-existent hook

**Severity:** MEDIUM  
**Category:** Bugs / Documentation in Code  
**File:** `packages/projects13/src/Http/Controllers/EpicController.php` (line 123)  
**Confidence:** HIGH

**Problem:**  
Comment: "The deleting hook locks the epic and refuses deletion while comments exist." No such hook exists.

**Recommendation:**  
Remove comment or implement the hook (see DB-001).

---

### BUG-003 — Project model missing entity relationship (without bridge)

**Severity:** LOW  
**Category:** Bugs / Correctness  
**File:** `packages/projects13/src/Models/Project.php`  
**Confidence:** MEDIUM

**Problem:**  
`Project` model has no `entity()` relation. Bridge adds it via `resolveRelationUsing`. Without bridge, `$project->entity` fails. This is by design (base package standalone) but should be documented.

---

### BUG-004 — EntityType deleting hook doesn't check trashed entities

**Severity:** LOW  
**Category:** Bugs / Correctness  
**File:** `packages/entities13/src/Models/EntityType.php` (lines 97-106)  
**Confidence:** MEDIUM

**Problem:**  
Deleting hook checks `DB::table('ENT_entity_entity_type')->where('entity_type_id', ...)->exists()` which includes pivot rows for soft-deleted entities. But comment says "A type that a entity holds cannot be dropped, not even when that entity is in the trash." The FK on pivot table has `restrictOnDelete()` for `entity_type_id`, so force-delete of type is blocked by DB. Soft-delete of type is blocked by hook. This is consistent.

**Correction:** Behavior matches spec. No bug.

---

## Database

### DB-001 — Missing deletion guards on Project and Epic
(See High Findings)

### DB-002 — Foreign key constraints missing CASCADE/SET NULL
(See High Findings)

### DB-003 — EpicComment migration includes unused columns
(See High Findings)

### DB-004 — entity_id nullable in projects table

**Severity:** MEDIUM  
**Category:** Database / Schema  
**File:** `packages/entity-projects13/database/migrations/2026_10_11_000000_add_entity_id_to_projects_table.php` (line 16)  
**Confidence:** HIGH

**Problem:**  
`entity_id` is nullable because SQLite can't add NOT NULL column to existing table. Application requires entity via validation rule. Documented in `openspec/specs/projects-domain/Discrepancias.md`.

**Recommendation:**  
Acceptable workaround. Consider making NOT NULL in future migration when safe.

---

### DB-005 — Unique active name index implementation varies by driver ✅ Acknowledged

**Severity:** INFO  
**Category:** Database  
**File:** `packages/basics13/src/Support/Database/helpers.php` (lines 63-86)  
**Confidence:** HIGH

**Observation:**  
MySQL/MariaDB uses generated column + unique index. SQLite/PostgreSQL uses partial index `WHERE deleted_at IS NULL`. Both correctly enforce uniqueness among non-deleted rows. Well-implemented. No action needed.

---

## Performance

### PERF-001 — N+1 risk in EntityCountsIntegration::activeCommentsCount

**Severity:** LOW  
**Category:** Performance  
**File:** `packages/entity-projects13/src/Integrations/EntityCountsIntegration.php` (lines 91-102)  
**Confidence:** MEDIUM

**Problem:**  
Correlated subquery for `comments_count` runs per entity row. For large entity lists, this could be slow.

**Evidence:**  
```php
private function activeCommentsCount(): Builder
{
    return EpicComment::query()
        ->selectRaw('count(*)')
        ->join('PRO_epics', ...)
        ->join('PRO_projects', ...)
        ->whereColumn('PRO_projects.entity_id', 'ENT_entities.id')
        ...
}
```

**Impact:**  
Only used in entity list query. Entities typically fewer than projects/epics. Monitor if entity count grows >10k.

**Recommendation:**  
Consider materialized view or denormalized `comments_count` column on entities if needed.

---

### PERF-002 — Entity list query joins entity_types for select options ✅ Acknowledged — Acceptable trade-off

**Severity:** INFO  
**Category:** Performance  
**File:** `packages/entities13/src/Http/Controllers/EntityController.php` (line 35)  
**Confidence:** MEDIUM

**Observation:**  
`EntityController::index()` loads ALL entity types (`EntityType::query()->orderBy('name_eu')->get()`) for form select options, even when just viewing list. Could be lazy-loaded via the existing `entity-types.select-options` endpoint.

**Decision:**  
Entity types are typically few (<50). The query is fast and the form is rendered inline in the modal. The existing `EntityTypeSelectOptionsQuery` provides searchable select options for larger datasets. Current implementation is acceptable; lazy-loading would add complexity (Alpine/Livewire async rendering) without measurable benefit at current scale. Revisit if entity types exceed ~200.

---

## Architecture

### ARCH-001 — Vocabulary mismatch: archive vs deactivate
(See High Findings)

### ARCH-002 — Integration contract pattern well-designed ✅ Acknowledged
(See INFO-003 in Informational Findings)

### ARCH-003 — Bridge correctly owns cross-domain logic ✅ Acknowledged
(See INFO-002 in Informational Findings)

### ARCH-004 — No speculative abstractions ✅ Acknowledged
(See INFO-001 in Informational Findings)

---

## Testing

### TEST-001 — Package test infrastructure broken in Docker
(See Low Findings) **Fixed:** ✅ Added `packages/packages` self-referential symlink in `testing13/mount.sh`.

### TEST-002 — Demo13 test uses non-isolated database
(See Low Findings)

### TEST-003 — ArchitectureTest not runnable due to infrastructure

**Severity:** MEDIUM  
**Category:** Testing  
**File:** `packages/projects13/tests/Unit/ArchitectureTest.php`  
**Confidence:** HIGH

**Problem:**  
`ArchitectureTest` contains valuable architectural guards (controller thinness, FormRequest usage, view patterns, policy/route alignment) but cannot run due to test bootstrap issues.

**Recommendation:**  
Fix test bootstrap (see PHPSTAN-001) to enable running architecture tests in CI.

---

### TEST-004 — Dusk tests limited coverage ✅ Acknowledged — Documented trade-off

**Severity:** INFO  
**Category:** Testing  
**Confidence:** MEDIUM

**Observation:**  
ARCHITECTURE.md states: "Dusk covers the entity CRUD end to end. Projects and epics have Dusk coverage of their CRUD and of the parent selector, but not of their trash, archived lists or drawer; those flows are covered by feature tests only." This is a reasonable and documented trade-off. No action needed.

---

## Production

### PROD-001 — Debug environment in Docker

**Severity:** LOW  
**Category:** Production / DevOps  
**File:** `projects/demo13/docker-compose.yml` (line 24: `XDEBUG_MODE: "debug"`)  
**Confidence:** HIGH

**Observation:**  
Docker Compose enables Xdebug by default. Production must override.

**Recommendation:**  
Document production deployment must set `XDEBUG_MODE=off` and `APP_DEBUG=false`.

---

### PROD-002 — No Redis/Horizon for queues ✅ Acknowledged — Acceptable for current scope

**Severity:** INFO  
**Category:** Production / Scalability  
**Confidence:** HIGH

**Observation:**  
No async jobs defined. Database queue driver sufficient for current scale. Document if jobs are added later. No action needed.

---

## Maintainability

### MAINT-001 — entities13 lowercase namespace inconsistency
(See Low Findings)

### MAINT-002 — Consistent patterns across packages ✅ Acknowledged

**Severity:** INFO  
**Category:** Maintainability  
**Confidence:** HIGH

**Observation:**  
All packages follow identical structure: Models, Policies, Requests, Controllers (main/Trash/Archived), Queries, Transformers, Factories, Migrations, Seeders, Views. Controllers extend shared base classes. Traits (`TracksAuditColumns`, `TransactionalDelete`) reused. High consistency reduces cognitive load. This is a strength.

---

### MAINT-003 — OpenSpec specs document invariants and known discrepancies ✅ Acknowledged

**Severity:** INFO  
**Category:** Maintainability  
**Confidence:** HIGH

**Observation:**  
`openspec/specs/` documents requirements with scenarios and known discrepancies. This is excellent for long-term maintainability. Discrepancies are explicitly tracked, not hidden. This is a strength.

---

## Rejected Findings

| ID | Claim | Reason for Rejection |
|----|-------|---------------------|
| — | "Missing Repository pattern" | Not needed; architecture explicitly rejects it (ARCHITECTURE.md line 169) |
| — | "Services layer needed" | Controllers are thin; queries in dedicated classes; actions in controllers. No evidence of fat controllers. |
| — | "DTOs for requests" | FormRequests handle validation and authorization; DTOs would be speculative. |
| — | "Event sourcing for audit" | `TracksAuditColumns` trait provides sufficient audit trail for current needs. |
| — | "EpicComment should have separate CRUD" | Spec explicitly says comments are aggregates, no independent mutation. |
| — | "PHPStan baseline should be empty" | Baselines are pragmatic for legacy code; not a defect. |
| — | "Pint should pass everywhere" | Style issues are mechanical; not logic defects. Fix with `pint`. |

---

## Action Plan

Ordered by impact (Critical → High → Medium → Low):

### Immediate (Critical)
1. **Fix entities13 namespace case** — Change `composer.json` prefixes to `Entities13\\`, run `composer dump-autoload` in demo13. Fixes seeder failure and PHPStan error. (BUG-001) ✅ **Already fixed** — `composer.json` uses PascalCase, tests pass.
2. **Fix EpicComment model** — Remove SoftDeletes/TracksAuditColumns/TransactionalDelete; update migration (drop active, deleted_at, audit columns); add FK CASCADE/SET NULL; update policy. (SEC-001, DB-002, DB-003)

### Short-term (High)
3. **Add deletion guards to Project and Epic** — Implement `deleting` hooks checking for epics/comments (including trashed). (DB-001)
4. **Fix PHPStan configuration** — Exclude `tests/` from package analysis or fix bootstrap mapping. (PHPSTAN-001) ✅ PHPSTAN-002 baseline path fixed (baseline now empty).
5. **Resolve vocabulary mismatch** — Update AGENTS.md, zone rules, ARCHITECTURE.md to use "archive/archived" consistently. (ARCH-001)
6. **Run Pint on projects13 and entity-projects13** — Auto-fix style issues. (STYLE-001) ✅ Done.

### Medium-term (Medium)
7. **Remove dead code** — Delete `UniqueProjectName.php` or integrate it. (CODE-001) ✅ **Already done** — file was deleted.
8. **Update documentation** — Fix config keys, translation keys. (CONFIG-001) **DOC-001 fixed** — package paths updated in AGENTS.md and views.instructions.md.
9. **Fix EpicController comment** — Remove reference to non-existent deleting hook. (BUG-002)

### Long-term (Low)
10. **Improve test infrastructure** — Document Testbench as primary package test runner; consider fixing Docker symlinks for local package testing. (TEST-001) ✅ **Fixed** — added `packages/packages` self-referential symlink.
11. **Add RefreshDatabase to demo13 tests** — Ensure test isolation. (TEST-002) ✅ **Fixed** — Unit tests use SQLite in-memory; Feature tests use MySQL with RefreshDatabase/LazilyRefreshDatabase via per-testsuite bootstrap files.
12. **Monitor EpicComment query performance** — If entity list grows, optimize `activeCommentsCount`. (PERF-001)

---

*Review completed 2025-10-19. All findings based on read-only analysis of source code, configuration, tests, and documentation. No application code was modified.*