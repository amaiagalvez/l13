---
name: new-resource
description: Scaffold a full CRUD resource following the canonical Entity pattern. Use when the request is a whole new entity (a model, migration, controllers, policy, list, form, trash and tests, e.g. "CRUD de facturas"). For a single change inside an existing resource, use new-feature instead.
disable-model-invocation: true
argument-hint: Resource singular, StudlyCase (e.g. Invoice)
---

# NEW RESOURCE

Resource: $ARGUMENTS (ask if empty). Build the full CRUD set mirroring `Entity` — the canonical resource — never a different structure.

1. Read `AGENTS.md`, `.github/instructions/*.instructions.md` and `.github/docs/architecture/ARCHITECTURE.md`.
2. **Pick the owning package first.** A new resource belongs to a domain package (`entities13`, `projects13`) or to the `entity-projects13` bridge when it spans both. Everything below happens there, under that package's PSR-4 root (`Entities13\`, `Projects13\`, `EntityProjects13\` → `src/`) — never under `projects/demo13/app`.
3. TDD first: write failing tests mirroring `entities13/tests/{Feature/Entities,Unit/Validation}` and the bridge's shared `entity-projects13/tests/Feature/{ResourceAccessTest,ResourceWriteAccessTest,ResourceActivationTest}.php`. Run them narrowest-first inside the package.
4. Generate the complete set (paths relative to the owning package):
   - `database/migrations/*_create_x_table.php` (unique-among-active index, FKs) + `database/factories/XFactory.php` with the existing states (`trashed()`, `archived()`)
   - `src/Models/X.php`: `#[Fillable]`, `#[UsePolicy]`, `casts()`, typed relationships, soft deletes
   - `src/Http/Requests/{XRequest,XListRequest,XRestoreRequest}` (rules + `authorize()` via policy), `src/Policies/XPolicy.php`
   - `src/Http/Controllers/{XController,XTrashController,XArchivedController}` (the archived one extends `Basics13\Http\Controllers\ArchivedController`), `src/Queries/Xs/XListQuery.php`, `src/Transformers/XListTransformer.php`
   - `routes/web.php`: the resource routes plus `xs/archived`, `xs/archived/{x}` (activate) and `xs/{x}/archive`; `resources/views/xs/{list,form}.blade.php` using `<x-list.table>` and `<x-forms.tracked-resource>` (each list renders its tabs through `<x-list.tabs>`, fed by `XListTransformer`); `resources/lang/{es,eu,fr,en}.json` in the owning package (default `eu`), translation keys namespaced as `basics13::messages.*`, `entities13::messages.*`, `projects13::messages.*`, `entity-projects13::messages.*`
5. Validate everything the user sends: every store/update field has a FormRequest rule; catch `QueryException` → `UniqueConstraintViolation::rethrowAsValidationError()`; never read raw input in controllers.
6. Run and report real results: `DX ./vendor/bin/pint --dirty --format agent`, the new tests, and the package guard rails listed in the `docker-commands` skill (`bash testing13/phpunit.sh projects13` / `entity-projects13`). They are the ones that catch a resource that drifts from the canonical set, so a green new test suite is not enough. Never claim a pass without running it.
7. Ask before adding dependencies, base folders or abstractions.
