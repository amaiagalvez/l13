---
name: test-rules
description: "Test location rules, PHPUnit conventions, test types, database traits. Load when writing or modifying tests."
---

# Test Rules

## Test Location Rules (apply on EVERY test creation/modification)

**Decide where the test belongs BEFORE writing it:**

| Test Type | Location | DB? | Trait |
|-----------|----------|-----|-------|
| Pure logic (rules, policies, helpers, middleware config, query structure, translations) | `tests/Unit/...` | ❌ No | none |
| Validation rules (FormRequest `rules()`, `authorize()`, `prepareForValidation()`) | `tests/Unit/Validation/*RequestTest.php` | ❌ No | none |
| Policy methods (`viewAny`, `create`, `update`, etc.) | `tests/Unit/Policies/*PolicyTest.php` | ❌ No | none |
| Query builders (`ListQueryBase`, search patterns, state counts) | `tests/Unit/Queries/*ListQueryTest.php` | ❌ No | none |
| HTTP integration (full request/response, view rendering) | `tests/Feature/...` | ✅ Yes | `RefreshDatabase` |
| Read-light Feature tests (auth pages, query tests with 1 create, middleware) | `tests/Feature/...` | ✅ Yes (lazy) | `LazilyRefreshDatabase` |

## Database Trait Guidance

**Project convention (used in this codebase):**
- `RefreshDatabase` — for tests that create multiple records, test CRUD/Trash flows, or need full isolation per test.
- `LazilyRefreshDatabase` — for tests that create 0–1 records and mostly read (auth pages, settings, dashboards).

**Performance note (from `testing-best-practices` skill):**
`LazilyRefreshDatabase` is preferred when applicable because it only runs migrations when the test actually touches the database. Consider migrating existing `RefreshDatabase` tests to `LazilyRefreshDatabase` when they create few records.

## Decision Checklist for New Tests
1. Can it run without touching the database? → `tests/Unit/`
2. Does it only validate rules/authorize/prepareForValidation? → `tests/Unit/Validation/`
3. Does it test a Policy method directly? → `tests/Unit/Policies/`
4. Does it test query structure (constants, method signatures, search patterns)? → `tests/Unit/Queries/`
5. Does it need HTTP + DB (CRUD, Trash, ListQuery with real data)? → `tests/Feature/` + `RefreshDatabase`
6. Does it only create 0-1 records and mostly reads? → `tests/Feature/` + `LazilyRefreshDatabase`

## Refactoring Existing Tests
- When touching a Feature test that only does pure logic, move it to Unit.
- When touching a Feature test with few creates, consider `LazilyRefreshDatabase`.

## General Testing Conventions
- Add/update tests for behavior changes (not for copy/styling). Cover the change and its key failures only.
- Create: `DX php artisan make:test --phpunit Name` (feature; `--unit` for unit; no suite dir in name).
- Run the narrowest: `DX php artisan test --compact <path|--filter=name>`; add `--parallel` for local speed (never in CI — shared DB) with flags before the path: `DX php artisan test --parallel --compact <path>`. Rerun after each fix.
- Dusk (`tests/Browser`) only for JS behavior.
- Duplicate-insert races tested per resource for store, update and restore by simulating a duplicate-key `QueryException`.
- Dusk runs against `laravel_test` with a non-English default locale, so assert on translated text.
- Run Dusk as: `docker compose exec -e XDEBUG_MODE=off laravel13-dusk php artisan dusk [tests/Browser/...php]`, never in the default service.