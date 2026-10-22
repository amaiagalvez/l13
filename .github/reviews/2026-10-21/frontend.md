# Frontend — raw specialist report (2026-10-21)

### FE-001: Changing the date operator does not re-render the value inputs
- Severity: HIGH (proposed) · Confidence: HIGH
- File: packages/basics13/resources/views/components/list/advanced-search-modal.blade.php:44, 65, 89-102
- Problem: the operator select uses deferred `wire:model`. The default date operator is `between`, which shows two inputs. If the user switches to `before`, an array value is sent; switching back to `between` sends a scalar, which the query ignores.
- Recommendation: use `wire:model.live` on the operator, or Alpine `x-show`, and reset the value when the shape changes.

### FE-002: Simple search and advanced filters overwrite each other in the URL
- Severity: HIGH (proposed) · Confidence: HIGH
- File: packages/basics13/resources/js/app.js:499-510, 512-542
- Problem: both functions build the URL from `form.action`, which has no query string. Applying filters drops `search`, and searching drops `filters`, while the badges outside the fragment still show the old filters. The delete loop also mutates `searchParams` while iterating it.
- Recommendation: start from `new URL(location.href)` and copy the keys before deleting.

### FE-003: Fields hidden by `condition` come back under "show advanced"
- Severity: MEDIUM
- File: packages/basics13/src/Livewire/AdvancedSearchModal.php:92-100, 299-302

### FE-004: A malformed `filters` value causes a 500
- Duplicate of API-001. Also packages/basics13/resources/views/components/list/page-header.blade.php:67.

### FE-005: Project filter labels use `projects13::messages.*` keys, but projects13 has no messages.php
- Severity: MEDIUM · Confidence: HIGH [verified]
- File: packages/projects13/src/Queries/Projects/ProjectListQuery.php:110, 118, 127, 136, 145, 153
- Verified: packages/projects13/resources/lang/{en,es}/ contain only validation.php, and eu/fr have no PHP files.
- Impact: raw keys appear in the modal and badges in every locale.
- Recommendation: use the existing JSON keys, as EntityListQuery does.

### FE-006: Filter value inputs have no accessible name
- Severity: MEDIUM
- File: packages/basics13/resources/views/components/list/advanced-search-modal.blade.php:54-56, 79-101

### FE-007: Entity-type filter options are scoped by the type's own `active` flag
- Severity: MEDIUM
- Same root cause as BUG-005.
- File: packages/entities13/src/Queries/Entities/EntityListQuery.php:130-134

### FE-008: Operator labels receive the whole messages array, so `is_empty` shows "(empty)"
- Severity: LOW
- The labels are protected and lost after hydration.
- File: packages/entities13/resources/views/entities/list.blade.php:165-167; packages/basics13/src/Livewire/AdvancedSearchModal.php:33-40, 155

### FE-009: `$fields` and `$scope` are not `#[Locked]`
- Severity: LOW (as proposed)
- Superseded by SEC-001, which turned out to be exploitable.

### FE-010: Unknown operators show a raw translation key in the badge, and `$filtersB64` is unused
- Severity: LOW
- File: packages/basics13/resources/views/components/list/page-header.blade.php:57-58, 84

### FE-011: The advanced toggle round-trips to the server and has no `aria-expanded`
- Severity: LOW
- File: packages/basics13/resources/views/components/list/advanced-search-modal.blade.php:28-32, 52

### FE-012: Wrong and inconsistent Basque in the bridge translations
- Severity: LOW · Confidence: HIGH [verified]
- File: packages/entity-projects13/resources/lang/eu.json:2-3, 6-9
- Problem: "atzilea" is used instead of "Bezeroa", and "Bezeroak ez direkitut" is invalid Basque. These lines are committed (not part of the current diff). `eu` is the default locale.

### FE-013: A list emptied by filters shows the "no records yet" empty state and the create CTA
- Severity: LOW
- File: packages/basics13/src/Transformers/ListTransformer.php:119; packages/entities13/resources/views/entities/list.blade.php:127

Checked and fine:
- No `{!! !!}` in the components.
- `@js` is safe.
- `messages.php` has 81 keys in all 4 locales.
- The new JSON keys match across locales.
- The modal has `aria-labelledby`.
- `wire:key` is set on options.
