# API review — raw specialist report (2026-10-21)

Scope: internal JSON/AJAX endpoints (no `routes/api.php`). Read-only. Current working tree, including the uncommitted advanced-search work.

Checked and fine: every route is under `web` (session + CSRF) and `auth`, and the domain routes also have `verified`. Select-option endpoints authorize `viewAny` through FormRequests, validate `q` and cap results at 25. Inline creates return 201 with `id` and `name`. A trashed-name conflict returns 409. Unique-index races map to 422. Comments expose only the author's name. `entity-projects13` has no routes.

### API-001: Advanced-search filter value that is an array causes a 500
- Severity: MEDIUM · Category: Validation · Confidence: HIGH
- File: packages/basics13/src/Queries/ListQueryBase.php:199 (also 222-223); packages/basics13/src/Http/Requests/SearchableListRequest.php:17-19
- Problem: `filters.*.value` is validated only as `nullable`, and `applyTextFilter` then does `(string) $value`. The date `between` branch reads `$value[0]` and `$value[1]` after only checking `count === 2`.
- Evidence: `?filters[name][operator]=contains&filters[name][value][]=x` throws "Array to string conversion".
- Impact: Any verified user can trigger a 500, which leaks a stack trace when debug is on.
- Recommendation: Validate `filters.*.operator` as `in:` the known operators. Validate `filters.*.value` as `string|array|max`, and `filters.*.value.*` as scalar. Add type guards in `apply*Filter` and tests for each bad shape.

### API-002: Advanced text LIKE filters escape with `!` but never declare `ESCAPE '!'`
- Severity: MEDIUM · Category: Correctness · Confidence: HIGH
- File: packages/basics13/src/Queries/ListQueryBase.php:202,205,206 (escaping at 362-368; `whereMatches` at 105-109 does declare it)
- Problem: The pattern `%50!%%` is matched with MariaDB's default escape character, `\`.
- Impact: Filters containing `%`, `_` or `!` return wrong rows, and basic search and advanced filter disagree on the same term.
- Recommendation: Reuse the `whereMatches`/`searchPattern` approach with `escape '!'`, and add tests.

### API-003: Declared field `operators` and `visible` are not enforced server-side
- Severity: LOW · Confidence: HIGH
- File: packages/basics13/src/Queries/ListQueryBase.php:121-153; packages/entities13/src/Queries/Entities/EntityListQuery.php:96-104
- Problem: Unknown operators fall to `default` branches, and the boolean branch (line 242) and single-select branch (line 324) still apply a condition there.
- Recommendation: Skip any operator that is not in the field's `operators` list.

### API-004: `GET epics/{epic}/comments` has no authorization check
- Severity: LOW · Confidence: HIGH
- File: packages/projects13/src/Http/Controllers/EpicCommentController.php:16; packages/projects13/routes/web.php:49-51
- Problem: This is the only JSON read without a FormRequest or `authorize()`. It has no impact today, because the policy allows everyone, but it will bypass roles once they exist.
- Recommendation: Add `$this->authorize('viewAny', Epic::class)` and a 403 test.

### API-005: Race paths return 500 or 404 instead of 422/409
- Severity: LOW · Confidence: MEDIUM
- File: packages/entities13/src/Http/Controllers/EntityController.php:123; packages/projects13/src/Http/Controllers/EpicController.php:91,109
- Problem: When a type is trashed after validation, a plain `RuntimeException` is thrown and the response is a 500. When a project is trashed after validation, `findOrFail` returns a 404.
- Recommendation: Throw a `ValidationException` on the field.

### API-006: Epic comment lock is released before the insert
- Severity: LOW · Confidence: HIGH
- File: packages/projects13/src/Http/Controllers/EpicCommentController.php:42-45, 52-54
- Problem: `lockForUpdate` runs in its own `DB::transaction`, which commits immediately, and the save happens outside it. The comment at line 41 claims a guarantee the code does not give.
- Recommendation: Put the lock, the check and the save in one transaction.

### API-007 (INFO): Update and destroy response shapes differ by resource
Entity and EntityType `update` return redirects only, while Project and Epic `update` return JSON when the client expects it. Leave as is until a JS caller needs it.

### API-008 (INFO): User options search by email can enumerate email addresses
packages/basics13/src/Queries/Users/UserSelectOptionsQuery.php:33. This matters only once `viewAny` on users is restricted.

### API-009 (INFO): Advanced-search option closures load every row
packages/entities13/src/Queries/Entities/EntityListQuery.php:130-134; packages/entity-projects13/src/ServiceProvider.php:113-117. Acceptable at current data sizes.

Not verified: whether the JS sends `X-CSRF-TOKEN` on fetch writes.
