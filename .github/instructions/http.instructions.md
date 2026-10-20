---
applyTo: "**/app/Http/**,**/app/Policies/**,**/app/Queries/**,**/app/Transformers/**,packages/*/src/Http/**,packages/*/src/Policies/**,packages/*/src/Queries/**,packages/*/src/Transformers/**"
---

# HTTP / domain rules

These live in the owning package, never in `projects/demo13/app`. Names below are the canonical ones; mirror them, prefixed with the package namespace (`Entities13\`, `Projects13\`, `EntityProjects13\`).

- Canonical resource: `EntityController` + `EntityRequest` + `EntityPolicy` + `Queries\Entities\EntityListQuery` + `EntityListTransformer`, in `entities13/src/`. Mirror it (never reinvent it) when adding a resource.
- Store/update: catch `QueryException` and call `Basics13\Support\Database\UniqueConstraintViolation::rethrowAsValidationError()`. Restore flows return their own conflict response instead.
- `EntityController::store` also returns JSON `201`/`409` responses for inline entity creation from the project selector; `entities13/tests/Feature/Entities/EntityCrudTest` fixes that contract.
- Deleting or force-deleting a parent is blocked if children exist, including trashed ones (`withTrashed()->exists()`). Entity→Project and Project→Epic work the same way.
- `active` is not mass assignable: it only changes through the resource's `XArchivedController` (extends `Basics13\Http\Controllers\ArchivedController`), which serves the archived list plus the routes named `{plural}.archived.index`, `{singular}.archive` and `{plural}.archived.activate`. A new resource ships that controller too.
- Form Requests authorize through the resource policy. Policies currently allow every verified user; this is documented in ARCHITECTURE.md, do not add roles unasked.
- List querying lives in `src/Queries`, row shaping in `src/Transformers`; keep controllers thin.
- A `max` rule never carries a literal number: it reads `config('basics13.validation.max_length.string')` for a varchar-backed field or `config('basics13.validation.max_length.longtext')` for a `longText` one. `entity-projects13/tests/Unit/ValidationCoverageTest.php` fails on a literal or on a key that does not resolve.

> Invariantes de esta zona: `openspec/specs/basics-crud-base/spec.md`, `openspec/specs/basics-trash-archived/spec.md`, `openspec/specs/entities-domain/spec.md`, `openspec/specs/projects-domain/spec.md`.
