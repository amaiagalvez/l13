# Business logic — raw specialist report (2026-10-21)

### BUG-001: An epic's start date can fall after its project's end date
- Severity: MEDIUM · Confidence: HIGH
- File: packages/projects13/src/Http/Requests/EpicRequest.php:64-73
- Spec: openspec/specs/epics-comments ("Sus fechas MUST quedar dentro de las del proyecto")
- Recommendation: when `$project->end_date` is set, add `before_or_equal:<project end>` to `start_date`.

### BUG-002: Editing a project's dates can leave its epics outside the new range
- Severity: MEDIUM · Confidence: HIGH (missing check) / MEDIUM (spec intent)
- File: packages/projects13/src/Http/Requests/ProjectRequest.php:73-74
- Recommendation: bound the edit by the min/max dates of the project's epics, or clarify in the spec that the rule applies only when an epic is saved.

### BUG-003: The comment-store lock is released before the insert
- Duplicate of DB-001.

### BUG-004: Duplicated model `deleting` guards with no-op locks
- Duplicate of DB-002.

### BUG-005: Entity filter options are scoped by the entity's own `active` flag, not by the list being filtered
- Severity: LOW
- File: packages/entity-projects13/src/ServiceProvider.php:113-117 (same pattern in packages/entities13/src/Queries/Entities/EntityListQuery.php:130-134)
- Problem: the active project list contains projects of archived entities, and those entities cannot be picked as a filter.
- Note: this choice was intended by design.md of the advanced-search change.

### BUG-006: Advanced filter LIKE has no ESCAPE clause
- Duplicate of LAR-002.

### BUG-007 (INFO): ARCHITECTURE.md contradicts the code
- File: .github/docs/architecture/ARCHITECTURE.md:54-56, 99-101
- Problems:
  - It says comment authors are nulled and comments cascade. In fact both FKs are RESTRICT, and deletion is blocked.
  - It says the user `active` flag is "never written". In fact packages/basics13/src/Http/Controllers/UserArchivedController.php exists.

Checked and fine:
- Unique names are enforced by both the request and the index.
- The restore conflict is handled.
- Delete guards count trashed children.
- Archive replaces delete in row actions.
- Epic and project date rules are correct.
- Selectors keep the currently selected archived record.
- `active` is never mass assigned.
- Archiving does not cascade.
- Inactive users are blocked.
