# Architecture — raw specialist report (2026-10-21)

### ARCH-001: The bridge puts closures into runtime config, which breaks `config:cache`
- Duplicate of LAR-001 (verified).
- Suggested seam: add `searchableFields()` to the existing `ProjectQueryIntegration`. This changes a contract, so ask first.

### ARCH-002: entities13 hard-codes bridge names
- Severity: MEDIUM · Confidence: HIGH
- File: packages/entities13/src/Transformers/EntityListTransformer.php:19 (`'projects_exists'`); packages/entities13/src/Queries/Entities/EntityListQuery.php:162 (`'entity_projects'`)
- Problem: the bridge ignores the alias it receives (packages/entity-projects13/src/Integrations/EntityCountsIntegration.php:23, 37, 59). PackageIsolationTest's FORBIDDEN list (packages/entities13/tests/Unit/PackageIsolationTest.php:30-47) cannot detect either string.
- Recommendation: use neutral key and alias names, and add both strings to FORBIDDEN.

### ARCH-003: projects13 names entities, and the bridge duplicates projects13's alias literals
- Severity: LOW
- File: packages/projects13/src/Queries/Projects/ProjectListQuery.php:26; packages/projects13/src/Queries/Epics/EpicListQuery.php:29, 234; packages/projects13/src/Http/Controllers/ProjectController.php:36; packages/entity-projects13/src/Integrations/EntityProjectsIntegration.php:86, 218-226

### ARCH-004: Advanced-filter "is active" rules are written three times
- Severity: MEDIUM
- File: packages/basics13/src/Queries/ListQueryBase.php:132-137, 148; packages/basics13/src/Livewire/AdvancedSearchModal.php:96-98, 187, 222-234, 299-302; packages/basics13/resources/views/components/list/page-header.blade.php:13-24, 46-55
- Recommendation: add one static predicate on ListQueryBase.

### ARCH-005: basics13 carries domain knowledge
- Severity: LOW
- Problems: `findTrashedByName(..., ?int $projectId)` and the `'names'` group order in AdvancedSearchModal:109.

### ARCH-006: The `condition` callables never hide anything, because no list route has `{entity}` or `{project}`
- Severity: LOW
- File: packages/entity-projects13/src/ServiceProvider.php:118; packages/projects13/src/Queries/Epics/EpicListQuery.php:187

### ARCH-007: The label props on the Livewire modal produce wrong labels, and the unused methods are dead code
- Severity: LOW
- Duplicate of FE-008/MAINT-004.

Checked and fine:
- Bridge dependency direction is correct.
- Runtime relations live in the bridge.
- The standalone default for `searchable_fields` is `[]`.
- basics13 JS and lang have no domain knowledge.
- No speculative layers.
