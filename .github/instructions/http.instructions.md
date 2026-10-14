---
applyTo: "**/app/Http/**,**/app/Policies/**,**/app/Queries/**,**/app/Transformers/**,packages/*/src/Http/**,packages/*/src/Policies/**,packages/*/src/Queries/**,packages/*/src/Transformers/**"
---

# HTTP / domain rules

These live in the owning package, never in `projects/demo13/app`. Names below are the canonical ones; mirror them, prefixed with the package namespace (`Customers13\`, `Projects13\`, `CustomerProjects13\`).

- Canonical resource: `CustomerController` + `CustomerRequest` + `CustomerPolicy` + `Queries\Customers\CustomerListQuery` + `CustomerListTransformer`, in `customers13/src/`. Mirror it (never reinvent it) when adding a resource.
- Store/update: catch `QueryException` and call `Basics13\Support\Database\UniqueConstraintViolation::rethrowAsValidationError()`. Restore flows return their own conflict response instead.
- `CustomerController::store` also returns JSON `201`/`409` responses for inline customer creation from the project selector; `customers13/tests/Feature/Customers/CustomerCrudTest` fixes that contract.
- Deleting or force-deleting a parent is blocked if children exist, including trashed ones (`withTrashed()->exists()`). Customer→Project and Project→Epic work the same way.
- `active` is not mass assignable: it only changes through the resource's `XArchivedController` (extends `Basics13\Http\Controllers\ArchivedController`), which serves the archived list plus the routes named `{plural}.archived.index`, `{singular}.archive` and `{plural}.archived.activate`. A new resource ships that controller too.
- Form Requests authorize through the resource policy. Policies currently allow every verified user; this is documented in ARCHITECTURE.md, do not add roles unasked.
- List querying lives in `src/Queries`, row shaping in `src/Transformers`; keep controllers thin.
- A `max` rule never carries a literal number: it reads `config('validation.max_length.string')` for a varchar-backed field or `config('validation.max_length.longtext')` for a `longText` one. `customer-projects13/tests/Unit/ValidationCoverageTest.php` fails on a literal or on a key that does not resolve.

> Invariantes de esta zona: `openspec/specs/basics-crud-base/spec.md`, `openspec/specs/basics-trash-archived/spec.md`, `openspec/specs/customers-domain/spec.md`, `openspec/specs/projects-domain/spec.md`.