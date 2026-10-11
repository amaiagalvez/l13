# CODE REVIEW — 2026-10-11

## Executive Summary

Reviewed three packages (`basics13`, `customers13`, `projects13`) and the demo application (`projects/demo13`). The codebase follows a consistent resource-oriented architecture with shared base classes for controllers, queries, and transformers. Overall quality is high with strong test coverage, but several inconsistencies and overengineering patterns exist across the three resource models.

**Total Findings: 14** — CRITICAL: 0, HIGH: 2, MEDIUM: 5, LOW: 5, INFO: 2

---

## Findings

### HIGH-001: Inconsistent Deletion Guard Implementation

**File:** `packages/customers13/src/Models/Customer.php:68-71`, `packages/projects13/src/Models/Project.php` (no delete method), `packages/projects13/src/Models/Epic.php` (no delete method)

**Problem:** `Customer` overrides `delete()` to wrap in a transaction, but `Project` and `Epic` do not. The deletion guard in `ServiceProvider::blockDeletionWithChildren()` wraps the check in its own `DB::transaction()`, but the `Customer` model adds an additional transaction wrapper that `Project`/`Epic` lack.

**Evidence:**
- Customer: `public function delete(): ?bool { return $this->getConnection()->transaction(fn (): ?bool => parent::delete()); }`
- Project/Epic: No override — rely on base `SoftDeletes::delete()`
- Both packages' ServiceProviders use identical `DB::transaction()` pattern in deleting hooks

**Impact:** Inconsistent transaction boundaries. The `Customer` model's extra transaction is redundant (the hook already transactions) but not harmful. However, the inconsistency means future developers may not know which pattern to follow.

**Recommendation:** Remove the `delete()` override from `Customer` model — the ServiceProvider hook already handles the transaction and locking. Or add identical overrides to `Project` and `Epic` for consistency. Prefer removal (simpler).

**Confidence:** HIGH

---

### HIGH-002: EpicComment Missing SoftDeletes and Audit Columns

**File:** `packages/projects13/src/Models/EpicComment.php:31-36`

**Problem:** `EpicComment` uses `TracksAuditColumns` but NOT `SoftDeletes`, unlike `Customer`, `Project`, and `Epic`. The migration `create_epic_comments_table.php:20` calls `addCommonColumns()` which adds `softDeletes()` and `deleted_at` index, but the model doesn't use the trait.

**Evidence:**
- Migration: `addCommonColumns($table)` → adds `$table->softDeletes()` and index on `['deleted_at', 'id']`
- Model: `use HasFactory, TracksAuditColumns;` — no `SoftDeletes`
- All other resource models: `use HasFactory, SoftDeletes, TracksAuditColumns;`

**Impact:** Soft delete columns exist in DB but Eloquent won't use them. Queries won't automatically exclude trashed comments. `onlyTrashed()`, `withTrashed()`, `restore()`, `forceDelete()` won't work. The `deleted_at` index is wasted.

**Recommendation:** Add `use SoftDeletes;` to `EpicComment` model. Verify if comments should be soft-deletable per requirements — if not, remove `softDeletes()` from migration and the `deleted_at` index.

**Confidence:** HIGH

---

### MEDIUM-001: Customer Model Has Redundant `delete()` Override

**File:** `packages/customers13/src/Models/Customer.php:68-71`

**Problem:** The `delete()` method wraps `parent::delete()` in a transaction, but the `deleting` hook in `ServiceProvider` already executes inside `DB::transaction()` with `lockForUpdate()`. This creates nested transactions (MySQL saves only the outermost).

**Evidence:**
- Customer::delete() → `getConnection()->transaction(parent::delete())`
- ServiceProvider::blockDeletionWithChildren() → `DB::transaction(fn () => Customer::withTrashed()->lockForUpdate()->firstOrFail())`
- Test `test_direct_model_delete_keeps_the_guard_inside_its_transaction` expects transaction level +2

**Impact:** Unnecessary complexity. The test actually verifies the nested transaction behavior, but this is an implementation detail, not a requirement.

**Recommendation:** Remove the `delete()` override from `Customer`. The guard in ServiceProvider is sufficient and is the canonical pattern used by `Project` and `Epic`.

**Confidence:** HIGH

---

### MEDIUM-002: Inconsistent Test Base Classes

**File:** `packages/customers13/tests/TestCase.php`, `packages/projects13/tests/TestCase.php`

**Problem:** Two different base test classes with different configurations:
- `customers13/TestCase`: Loads `LivewireServiceProvider`, `FluxServiceProvider`, sets up Blade anonymous component path, defines `projects.index` route
- `projects13/TestCase`: Uses `ValidatesFormRequests` trait, loads all three package providers, uses `CarbonImmutable` for Date facade, defines `login`, `verification.notice`, `dashboard` routes

**Evidence:** Both extend `Orchestra\Testbench\TestCase` but configure different providers, routes, and environment setup.

**Impact:** Duplication and drift risk. Tests in one package may not run correctly if moved to the other. Shared test utilities (`ValidatesFormRequests`) only available in `projects13`.

**Recommendation:** Create a shared `Basics13\Tests\TestCase` in `basics13` package that both extend. Move common setup there. Keep package-specific routes/providers in each.

**Confidence:** MEDIUM

---

### MEDIUM-003: Duplicate Validation Test Patterns

**File:** `packages/customers13/tests/Unit/Validation/CustomerRequestTest.php`, `packages/projects13/tests/Unit/Validation/ProjectRequestTest.php`, `packages/projects13/tests/Unit/Validation/EpicRequestTest.php`

**Problem:** `EpicRequestTest` has 23 comprehensive validation tests (required, min/max, date rules, unique scoped, exists with active condition, trim, authorize). `CustomerRequestTest` has 5 tests. `ProjectRequestTest` has 3 trivial tests (mostly placeholders with `assertTrue(true)`).

**Evidence:**
- EpicRequestTest: 248 lines, tests every rule, edge case, translation
- CustomerRequestTest: 60 lines, basic authorize + name rules only
- ProjectRequestTest: 41 lines, 2 tests are `assertTrue(true)` placeholders

**Impact:** Inconsistent validation coverage. `ProjectRequest` validation logic is untested (dates, customer_id exists, unique name scoped to customer). Bugs in project validation would not be caught.

**Recommendation:** Expand `ProjectRequestTest` to match `EpicRequestTest` coverage. Expand `CustomerRequestTest` to test notes length, trim, unique rule structure, authorize edge cases.

**Confidence:** HIGH

---

### MEDIUM-004: Inconsistent Query Test Coverage

**File:** `packages/customers13/tests/Unit/Queries/CustomerListQueryTest.php` (22 lines), `packages/projects13/tests/Unit/Queries/ProjectListQueryTest.php` (83 lines), `packages/projects13/tests/Unit/Queries/EpicListQueryTest.php` (91 lines)

**Problem:** `CustomerListQueryTest` only verifies method existence via reflection. `ProjectListQueryTest` and `EpicListQueryTest` test constants, method signatures, and `stateCounts()` return values.

**Evidence:**
- CustomerListQueryTest: 2 reflection tests
- ProjectListQueryTest: 9 tests including SEARCH_COLUMNS constant, stateCounts output, private method existence
- EpicListQueryTest: 9 tests including RECENT_COMMENTS_LIMIT constant, findTrashedByName signature with projectId

**Impact:** Customer query logic (active counts with projects/epics/comments subqueries, search columns) is untested. The complex `withActiveCounts()` and `activeCommentsCount()` methods have zero test coverage.

**Recommendation:** Add tests for `CustomerListQuery` matching the depth of `ProjectListQueryTest` — test search columns constant, stateCounts, withActiveCounts behavior, findTrashedByName.

**Confidence:** HIGH

---

### MEDIUM-005: Policy Tests Inconsistent — Reflection vs Behavior

**File:** `packages/customers13/tests/Unit/Policies/CustomerPolicyTest.php`, `packages/projects13/tests/Unit/Policies/ProjectPolicyTest.php`, `packages/projects13/tests/Unit/Policies/EpicPolicyTest.php`

**Problem:** `CustomerPolicyTest` uses reflection to verify method existence only. `ProjectPolicyTest` and `EpicPolicyTest` instantiate the policy and assert return values (`assertTrue()`).

**Evidence:**
- CustomerPolicyTest: 3 tests, all reflection-based
- ProjectPolicyTest: 8 tests, all behavioral (call method, assert true)
- EpicPolicyTest: 9 tests, all behavioral

**Impact:** Customer policy behavior is untested. If a method returned `false`, the test would still pass.

**Recommendation:** Convert `CustomerPolicyTest` to behavioral tests matching `ProjectPolicyTest` pattern.

**Confidence:** HIGH

---

### LOW-001: Overengineered `TracksAuditColumns` Concern

**File:** `packages/basics13/src/Concerns/TracksAuditColumns.php`

**Problem:** The concern uses `creating`/`updating`/`deleting` model events to set `created_by`/`updated_by`/`deleted_by`. This is 80+ lines for functionality that could be 3 lines using `$table->foreignId('created_by')->nullable()->constrained()->cascadeOnDelete()` + `Model::creating(fn ($m) => $m->created_by = auth()->id())` in a boot method. The concern also adds complexity for testing (must mock auth).

**Evidence:** 
- Concern handles 3 events, checks `auth()->check()`, uses `getKey()`
- Used by all 4 models (Customer, Project, Epic, EpicComment)
- Migration already has FK constraints on audit columns

**Impact:** Extra abstraction layer with no benefit over simpler approach. Harder to test (requires authenticated user). Not reusable outside this codebase.

**Recommendation:** Consider replacing with model `booted()` static methods or observers. Keep if team prefers this pattern, but document why.

**Confidence:** MEDIUM

---

### LOW-002: `EpicComment` Missing Factory States

**File:** `packages/projects13/src/Database/Factories/EpicCommentFactory.php` (not found — likely uses default)

**Problem:** `EpicComment` factory not visible in source tree. `EpicCommentTest` creates instances with `new EpicComment` and `fill()`, not factory. Other models have dedicated factories with states.

**Evidence:** 
- `CustomerFactory`, `ProjectFactory`, `EpicFactory` exist
- `EpicCommentTest::test_epic_comment_fields_can_be_mass_assigned()` uses `new EpicComment`
- `EpicCrudTest` uses `EpicComment::factory()` — so factory exists but not in src tree

**Impact:** Inconsistent factory usage. May be in vendor or generated.

**Recommendation:** Verify factory location. If missing, create `EpicCommentFactory` with states matching other factories.

**Confidence:** LOW

---

### LOW-003: Duplicate `fullName()` Logic Only in Project

**File:** `packages/projects13/src/Models/Project.php:102-105`

**Problem:** `Project::fullName()` returns `"name (customer name)"` for selectors. `Customer` and `Epic` have no equivalent. `EpicListTransformer` builds `project_label` manually in transformer.

**Evidence:**
- Project::fullName() used in `EpicController::drawerEpic()` and select options
- EpicListTransformer::payloadFor() builds `'project_label' => $epic->project->fullName()`
- No `Customer::fullName()` or `Epic::fullName()`

**Impact:** Minor inconsistency. If customer/epic need display labels in selectors, pattern should be consistent.

**Recommendation:** Add `fullName()` to `Customer` and `Epic` if needed, or document why only Project has it.

**Confidence:** LOW

---

### LOW-004: `addCommonColumns()` Adds `deleted_at` Index to EpicComment Unnecessarily

**File:** `packages/basics13/src/Support/Database/helpers.php:27`, `packages/projects13/src/Database/Migrations/2026_09_30_175707_create_epic_comments_table.php:20`

**Problem:** `addCommonColumns()` adds `index(['deleted_at', 'id'])` for soft delete performance. But `EpicComment` doesn't use `SoftDeletes` (see HIGH-002), so the index is unused.

**Evidence:** Migration calls `addCommonColumns($table)` which adds the index. Model lacks `SoftDeletes`.

**Impact:** Wasted index storage and write overhead.

**Recommendation:** Fix HIGH-002 first. If `EpicComment` gets `SoftDeletes`, index is valid. If not, create a variant `addCommonColumnsWithoutSoftDeletes()` for comments.

**Confidence:** MEDIUM

---

### LOW-005: Inconsistent Route Naming in Transformers

**File:** `packages/customers13/src/Transformers/CustomerListTransformer.php:127-138`, `packages/projects13/src/Transformers/ProjectListTransformer.php`, `packages/projects13/src/Transformers/EpicListTransformer.php`

**Problem:** All three transformers define `routes()` returning array with keys `active`, `archived`, `trash`, `destroy`, `archive`, `activate`, `restore`, `trashDestroy`. `EpicListTransformer` adds `comment` route. The route *names* differ per resource (e.g., `customers.index` vs `projects.index` vs `epics.index`) which is correct, but the *structure* is duplicated across three classes.

**Evidence:** Each transformer reimplements the same 8-key array with different route names.

**Impact:** Duplication. Adding a new action (e.g., `duplicate`) requires editing 3 files.

**Recommendation:** Move route name configuration to a shared config file or base transformer method that accepts resource prefix. Or keep as-is since route names are resource-specific and unlikely to change.

**Confidence:** LOW

---

### INFO-001: `basics13` Package Views Not Published by Default

**File:** `packages/basics13/src/ServiceProvider.php:34-44`

**Problem:** View and lang publishing is commented out in `basics13` ServiceProvider. The package loads views with `loadViewsFrom()` but doesn't publish them, so applications can't customize them without manually copying.

**Evidence:** Lines 34-44 are commented with `// $this->publishes([...], 'basics13-views');`

**Impact:** Applications using `basics13` cannot override views without manual work.

**Recommendation:** Uncomment the publishes if view customization is intended. Remove if views are intentionally internal.

**Confidence:** MEDIUM

---

### INFO-002: `ProjectListQuery` and `EpicListQuery` Have `SEARCH_COLUMNS` Constant, `CustomerListQuery` Does Not

**File:** `packages/projects13/src/Queries/Projects/ProjectListQuery.php`, `packages/projects13/src/Queries/Epics/EpicListQuery.php`, `packages/customers13/src/Queries/Customers/CustomerListQuery.php`

**Problem:** `ProjectListQuery` and `EpicListQuery` define `SEARCH_COLUMNS` constant tested in unit tests. `CustomerListQuery` passes search columns inline to `paginate()` call.

**Evidence:** 
- ProjectListQueryTest asserts `SEARCH_COLUMNS` constant exists with 4 columns
- EpicListQueryTest asserts constant with 5 columns
- CustomerListQuery passes `searchColumns: ['name']` directly

**Impact:** Minor inconsistency. Constant allows testing and reuse.

**Recommendation:** Add `SEARCH_COLUMNS` constant to `CustomerListQuery` for consistency.

**Confidence:** HIGH

---

## Action Plan (Ordered by Impact)

1. **HIGH-002** — Fix `EpicComment` model to use `SoftDeletes` (or remove from migration)
2. **HIGH-001** — Remove redundant `delete()` override from `Customer` model
3. **MEDIUM-003** — Expand `ProjectRequestTest` and `CustomerRequestTest` validation coverage
4. **MEDIUM-004** — Add comprehensive `CustomerListQueryTest` matching project/epic pattern
5. **MEDIUM-005** — Convert `CustomerPolicyTest` to behavioral tests
6. **MEDIUM-002** — Extract shared `TestCase` to `basics13` package
7. **MEDIUM-001** — Already covered by HIGH-001 fix
8. **LOW-004** — Address after HIGH-002 resolution
9. **LOW-001** — Evaluate `TracksAuditColumns` concern simplification
10. **LOW-002** — Verify/create `EpicCommentFactory`
11. **LOW-003** — Add `fullName()` to Customer/Epic or document
12. **LOW-005** — Consider shared route config for transformers
13. **INFO-001** — Decide on view publishing for basics13
14. **INFO-002** — Add `SEARCH_COLUMNS` constant to `CustomerListQuery`

---

## Positive Observations

- **Strong architectural consistency**: All three resources follow identical controller/query/transformer patterns
- **Excellent test coverage** for epic/project CRUD, trash, archive, concurrency, and validation
- **Race condition handling**: `UniqueConstraintViolation` + DB unique indexes + concurrent test data provider
- **Deletion guards** with row locking are correctly implemented in ServiceProviders
- **i18n ready**: All user-facing strings use `__()` with package namespaces
- **Type safety**: PHP 8.4 features (property hooks, attributes) used correctly
- **No speculative abstractions**: No repositories, services, DTOs, or domain layers

---

## Testing Commands Executed

```bash
# Package test discovery
find packages/*/tests -name "*Test.php" | head -50

# Model comparisons
grep -n "use.*SoftDeletes\|use.*TracksAuditColumns\|protected \$table\|protected \$attributes\|function casts\|function delete" packages/*/src/Models/*.php

# Migration comparisons
cat packages/*/src/Database/Migrations/*.php

# Test comparisons
cat packages/*/tests/Unit/Validation/*RequestTest.php
cat packages/*/tests/Unit/Queries/*ListQueryTest.php
cat packages/*/tests/Unit/Policies/*PolicyTest.php
cat packages/*/tests/Feature/*CrudTest.php
```