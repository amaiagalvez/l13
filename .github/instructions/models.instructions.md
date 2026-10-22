---
applyTo: "**/app/Models/**,packages/*/src/Models/**"
paths:
  - "**/app/Models/**"
  - "packages/*/src/Models/**"
---

# Model rules

- Models declare `#[Fillable]`, `#[UsePolicy]`, `casts()` and relationships only. Validation lives in Form Requests, authorization in policies, list querying in `src/Queries`. A derived read-only label is allowed when the same string is built for several surfaces (`Project::fullName()`).
- Soft deletes per resource; names are unique among active records, enforced in the database (`Basics13\Support\Database\UniqueConstraintViolation` absorbs the race). Both come from the helpers in `basics13/src/Support/Database/helpers.php` instead of being written out again: `addCommonColumns($table)` adds `notes`, `active`, the timestamps, soft deletes and the `['deleted_at', 'id']` index, `addAuditColumns($table)` adds `created_by`/`updated_by`/`deleted_by`, and `addUniqueActiveNameIndex($table, $scope)` adds the active-name unique index for the current driver.
- A record is refused deletion while it has children, trashed ones included. The guard is a `deleting` hook registered in the owning package's service provider (`Projects13\ServiceProvider::blockDeletionWithChildren()`), which locks the row inside its own transaction: it is what makes the delete safe, so a plain `$model->delete()` is protected too and callers must not lock the row again.
- Relationships use explicit `belongsTo`/`hasMany` return types; trashed children count where deletion is blocked (`withTrashed()`).
- A new column needs three things together: migration, `#[Fillable]`/`casts()` entry and Request rules — `ModelSchemaParityTest` enforces the first two. A column a form edits needs a fourth: it travels to the drawer in the transformer's `editPayload()` (and in `payloadFor()` where the drawer is rehydrated by id), and the field is bound to it with `x-model`; a column nothing carries is emptied every time the form is opened. `notes` is the worked example: a nullable `longText` with `nullable|string|max:` set to `config('basics13.validation.max_length.longtext')`, declared on `User`, `Entity`, `Project`, `Epic` and `EpicComment`.
- `notes` is always nullable, so an insert that leaves it out keeps working.
- All resource models must declare the following traits for parity: `SoftDeletes`, `TracksAuditColumns`, `TransactionalDelete`. These are declared via `use HasFactory, SoftDeletes, TracksAuditColumns, TransactionalDelete;` (see `Entity`, `Project`, `Epic` as reference).
- **Active flag management pattern (5+ layers per model):**
  1. `$attributes = ['active' => true]` — default value declaration in the model
  2. `casts(): ['active' => 'boolean']` — cast rule ensuring boolean type
  3. Migration `addCommonColumns($table)` (via `basics13/src/Support/Database/helpers.php`) — adds `active` column to DB schema
  4. Unique active name index `addUniqueActiveNameIndex($table, $scope)` — DB-level unique constraint on `active_name` (scope varies by resource)
  5. Bridge `EntityProjectsIntegration::hasActiveRelatedEntities()` — checks `Model::where('active', true)->exists()` for related entities
  6. Bridge `EntityProjectsIntegration::noRelatedEntitiesBanner()` — renders UI banner when no active records exist
  7. `TracksAuditColumns::bootTracksAuditColumns()` — never touches `active` (trash and restore keep it); it only stamps `created_by`/`updated_by`/`deleted_by`
  8. Resource `XArchivedController` (extends `Basics13\Http\Controllers\ArchivedController`) — handles archived state transitions and `active` flag changes
  9. Form UI — displays/hides fields and options based on `active` state
  10. Validation rules — may reference `active` state in form request rules
- A parent relation is never optional: `PRO_epics.project_id` is a NOT NULL foreign key; `PRO_projects.entity_id` is nullable in the database (SQLite cannot add a NOT NULL column to a filled table) with `restrictOnDelete`, but the application requires it. Every `name` is NOT NULL. Read those relations directly — no `?->`, no `'—'` fallback — and never write a test that fabricates the missing state with `setRelation()`.
  > Invariantes de esta zona: `openspec/specs/basics-trash-archived/spec.md`, `openspec/specs/entities-domain/spec.md`, `openspec/specs/projects-domain/spec.md`, `openspec/specs/epics-comments/spec.md`.
