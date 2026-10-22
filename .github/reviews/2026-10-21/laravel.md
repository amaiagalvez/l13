# Laravel — raw specialist report (2026-10-21)

Read-only. Orchestrator verification notes are marked [verified].

### LAR-001: Closures stored in runtime config make `config:cache` / `optimize` fail
- Severity: HIGH · Confidence: HIGH [verified]
- File: packages/entity-projects13/src/ServiceProvider.php:100-123 (`options` and `condition` closures under `projects13.searchable_fields`); read at packages/projects13/src/Queries/Projects/ProjectListQuery.php:162
- Verified: running `php artisan config:cache` in a throwaway container, with a scratch `bootstrap/cache`, throws `LogicException: Your configuration files are not serializable.` The code is committed (entity-projects13 d1fda1f).
- Impact: `config:cache` and `optimize`, standard production deploy steps, fail once the bridge is installed.
- Recommendation: keep closures out of config. Provide the extra searchable fields through the existing `ProjectQueryIntegration` contract, and add a regression test that serializes `config()->all()`.

### LAR-002: Advanced text filter escapes with `!` but never declares `ESCAPE '!'`
- Severity: MEDIUM · Confidence: HIGH [verified]
- File: packages/basics13/src/Queries/ListQueryBase.php:199-206 (helper at 362-368; the correct pattern is at 105)
- Recommendation: use `whereRaw($column.' like ? escape \'!\'', ...)` as `whereMatches()` does, and add a test.

### LAR-003: State-tab badges show the filtered total when filters are active
- Severity: MEDIUM · Confidence: HIGH [verified at 10 call sites]
- File: `stateCounts(...Total: $search === '' ? $x->total() : null)` in the Entity, EntityType, Project and Epic index, archived and trash controllers (e.g. packages/entities13/src/Http/Controllers/EntityController.php:40)
- Recommendation: take the shortcut only when `$search === '' && $filters === []`.

### LAR-004: A filter value sent as an array causes a 500
- Severity: LOW · Confidence: HIGH
- File: packages/basics13/src/Http/Requests/SearchableListRequest.php:17-20, 35-49; packages/basics13/src/Queries/ListQueryBase.php:199, 219
- Recommendation: type-guard values per field type, and constrain operators with `Rule::in`.

### LAR-005: `syncTypes()` throws `RuntimeException` (500) on a trashed-type race, and its nested transaction is redundant
- Severity: LOW · Confidence: MEDIUM
- File: packages/entities13/src/Http/Controllers/EntityController.php:113-127
- Recommendation: throw `ValidationException::withMessages(['entity_type_ids' => ...])` instead.

### LAR-006 (INFO): Archived and trash scopes offer only archived entities/types as filter options
- File: packages/entities13/src/Queries/Entities/EntityListQuery.php:130-134; packages/entity-projects13/src/ServiceProvider.php:113-117
- Note: this is intentional per openspec/changes/add-advanced-search-modal-to-lists/design.md:72,92. Product decision.

Checked and fine:
- Routes and middleware: `auth` and `verified`, plus `whereNumber`.
- Form Requests authorize through policies, and unique rules ignore trashed records.
- `active` is not mass assignable.
- Trash restore and force delete are correct.
- Host bootstrap: trusted hosts are set, and `EnsureUserIsActive` has correct priority.
- Filter column names come from the server only, and values are bound.
- `AdvancedSearchModal` resolves closures in `mount()`.
