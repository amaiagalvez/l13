# Testing — raw specialist report (2026-10-21)

Skip causes, verified by the orchestrator with grep:
- basics13: the 9 skips are `MariaDbUniqueActiveNameIndexTest`, which needs `BASICS13_MARIADB_DATABASE`.
- entities13: the 1 skip is `EntityTypeTrashTest.php:143`, which needs MySQL row locks.
- entity-projects13: the 5 skips are `ActiveNameCollationTest.php:21` (MySQL collation) and `Projects/ProjectCrudTest.php:384` (MySQL row locks).
- All 15 skips are MariaDB-only and run on SQLite in the package suites.

### TEST-001: MariaDB-only invariants (unique-name index, collation, row-lock guards) never run in CI
- Severity: HIGH · Confidence: HIGH
- File: packages/basics13/tests/Feature/MariaDbUniqueActiveNameIndexTest.php:30-34; packages/basics13/.github/workflows/tests.yml:67-73; packages/entities13/tests/Feature/EntityTypes/EntityTypeTrashTest.php:143; packages/entity-projects13/tests/Feature/ActiveNameCollationTest.php:21; packages/entity-projects13/tests/Feature/Projects/ProjectCrudTest.php:384
- Problem: the package workflows run on SQLite. demo13 CI has MariaDB but runs only the 10 app tests.
- Impact: the DB-enforced uniqueness, the accent/case collation and the delete/restore lock guards (openspec/specs/basics-trash-archived) are only checked by simulated exceptions in CI.
- Recommendation: add a `mariadb:11.7` service to the package workflows that own these tests, with `BASICS13_MARIADB_DATABASE` set. demo13's workflow already shows the pattern.

### TEST-002: No behavioural test covers advanced text filters with `%` or `_`, which hides the missing-ESCAPE bug
- Severity: MEDIUM · File: packages/basics13/tests/Unit/Queries/ListQueryBaseTest.php:1170-1203

### TEST-003: No test checks state counts under active filters, which hides the badge bug
- Severity: MEDIUM · File: packages/entities13/tests/Feature/Entities/EntityListFilterTest.php:34

### TEST-004: Filter feature tests exist for entities only
- Severity: MEDIUM · Confidence: MEDIUM
- Problem: there are no filter tests for projects (including the bridge's `entity_id` `is_empty` and multi-select filter), epics or entity types. Also untested: archived and trash scopes, search plus filters, pagination keeping `filters`, and malformed input. openspec/changes/add-advanced-search-modal-to-lists/tasks.md:76-92 is still unchecked.

### TEST-005: A field hidden by `condition` reappears with "show advanced", and nothing tests it
- Severity: MEDIUM · File: packages/basics13/src/Livewire/AdvancedSearchModal.php:100, 299-302; test at packages/basics13/tests/Unit/Livewire/AdvancedSearchModalTest.php:84-104

### TEST-006: Vacuous assertions
- Severity: LOW
- AdvancedSearchModalTest.php:137-140 contains only `assertTrue(true)`. PHPStan reports it.
- ListQueryBaseTest.php:211-309 asserts literal fixture arrays.
- ListQueryBaseTest.php:1598-1646 has unused fixtures.
- Dead `@requires` docblocks: PHPUnit 12 ignores doc-comment metadata.

### TEST-007: UserSelectOptionsQueryTest proves neither the active filter nor the ordering
- Severity: LOW · File: packages/basics13/tests/Unit/Queries/Users/UserSelectOptionsQueryTest.php:63-72, 88-101
- Problem: 30 active users already fill the 25-row cap, the `isset` on line 72 is always true, and the users in the ordering test are unused.

### TEST-008: The entities13 PHPStan baseline makes CI fail before the tests run
- Severity: MEDIUM · Duplicate of OPS-003 / MAINT-003.

Checked and fine:
- ResourceUniformityTest asserts the traits on all resources, including EpicComment.
- MariaDB index tests are solid when enabled.
- ListQueryBase operator coverage is good.
- Dusk runs in demo13 CI.
- Note: the 2026-10-19 report cites non-existent test files (PackageSeederTest, entities13 tests/Feature/EntityUniqueIndexTest.php).
