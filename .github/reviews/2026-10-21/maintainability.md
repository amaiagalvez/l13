# Maintainability — raw specialist report (2026-10-21)

### MAINT-001: Tab counts show the filtered total when filters are applied
- Duplicate of LAR-003.

### MAINT-002: Two LIKE-escaping paths have drifted, and one lacks ESCAPE
- Duplicate of LAR-002.

### MAINT-003: The staged entities13 phpstan-baseline.neon (793 lines) is environmental noise
- Severity: MEDIUM · Confidence: HIGH
- Problem: its entries are TestCase methods that PHPStan could not resolve at generation time. `scanFiles` now resolves them, so the 132 patterns are unmatched and PHPStan fails.
- Recommendation: unstage the file and regenerate the baseline, which should be empty.

### MAINT-004: Dead and ineffective code in the uncommitted `AdvancedSearchModal` and its view
- Severity: LOW · Confidence: MEDIUM
- File: packages/basics13/src/Livewire/AdvancedSearchModal.php:34-40, 96-98, 129-139, 165-168, 198-200; packages/basics13/resources/views/components/list/advanced-search-modal.blade.php:39-40, 52
- Problems:
  - Label properties are protected, so they are lost after hydration.
  - The grouping methods are unused.
  - Some branches cannot be reached.
  - `x-show` is always true.

### MAINT-005: The epic `project_name` condition depends on a `{project}` route parameter that never exists
- Severity: LOW · File: packages/projects13/src/Queries/Epics/EpicListQuery.php:187

### MAINT-006: Searchable field definitions have drifted between resources
- Severity: LOW
- Problems: the label conventions differ, the config hook exists only for projects, and an `active` boolean filter appears on the active and archived lists, where it is pointless.

### MAINT-007: "scope" means both list state and resource name in the same call chain
- Severity: LOW · File: packages/basics13/src/Queries/ListQueryBase.php:434; packages/projects13/src/Contracts/ProjectQueryIntegration.php:18

### MAINT-008: `ArchivedController` docblocks describe the redirect targets inverted
- Severity: LOW · File: packages/basics13/src/Http/Controllers/ArchivedController.php:24, 34, 37-45

### MAINT-009: Epic-specific `project_id` appears in domain-neutral basics13
- Severity: LOW · File: packages/basics13/src/Queries/ListQueryBase.php:398-417 (`findTrashedByName(..., ?int $projectId)`)

### MAINT-010: Entity `update` has no JSON branch, and `syncTypes` throws `RuntimeException`
- Severity: LOW · Accept as debt.

### MAINT-011: Docs contradict the code
- Severity: LOW
- Problems:
  - models.instructions.md "layer 7" says TracksAuditColumns manages `active`, which it does not.
  - The 2026-10-19 report contradicts itself on CRITICAL-001.
  - ARCHITECTURE.md:103 says "no multi-step transaction", but entity create/update syncs the pivot inside one transaction.
