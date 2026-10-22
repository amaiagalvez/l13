# Performance — raw specialist report (2026-10-21)

No `preventLazyLoading()` is enabled (projects/demo13/app/Providers/AppServiceProvider.php).

### PERF-001: Timeline lazy-loads the project for each epic and the entity for each project
- Severity: MEDIUM · Confidence: HIGH
- File: packages/projects13/src/Queries/Timeline/TimelineQuery.php:230-238, 345, 365; packages/entity-projects13/src/Integrations/EntityProjectsIntegration.php:245
- Recommendation: pass `$project` into `epicRow()`, and apply `queryIntegration->scope()` with `select('PRO_projects.*')` on the page query. Low cost.

### PERF-002: Planning lazy-loads the entity for each project row
- Severity: LOW · File: packages/projects13/src/Queries/Planning/PlanningQuery.php:175-179, 247
- Recommendation: same fix as PERF-001.

### PERF-003: Epic lists lazy-load the entity once per distinct project
- Severity: LOW · File: packages/entity-projects13/src/Integrations/EntityProjectsIntegration.php:108-112, 160-167, 231-256
- Recommendation: eager-load `project.entity:id,name,active`, or read the epic's joined `entity_name`.

### PERF-004: The advanced search modal loads every entity into the public `$fields` property, the Livewire snapshot and the markup
- Severity: MEDIUM · Confidence: HIGH
- File: packages/basics13/src/Livewire/AdvancedSearchModal.php:22, 64, 304-306; packages/entity-projects13/src/ServiceProvider.php:113-117; packages/basics13/resources/views/components/list/advanced-search-modal.blade.php:110-118
- Recommendation: keep `options` out of public state, or reuse the capped `entities.options` remote search.

### PERF-005: The entity list always joins projects and then DISTINCTs over correlated subqueries
- Severity: LOW · Confidence: MEDIUM (not EXPLAINed)
- File: packages/entity-projects13/src/Integrations/EntityCountsIntegration.php:33-47
- Recommendation: run EXPLAIN first. If confirmed, use EXISTS only when a search term is present.

### PERF-006 (INFO): Duplicate exists query on the projects index
- File: packages/projects13/src/Http/Controllers/ProjectController.php:36-37

Checked and fine:
- The entity and project lists aggregate in SQL.
- Comments use `withCount`, and the drawer loads 20 at a time.
- Select-option endpoints are capped at 25.
- `stateCounts` reuses the paginator total.
- No Redis, queues or external HTTP.
