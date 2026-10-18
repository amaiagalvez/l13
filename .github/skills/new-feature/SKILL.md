---
name: new-feature
description: Implement a feature end to end with tests, Pint and build. Use for behavior that is not a whole CRUD resource (a new action, filter, column or email). For a complete resource with model, migration, controller, policy, views and tests, use new-resource instead.
disable-model-invocation: true
argument-hint: describe the feature
---

# NEW FEATURE

Implement the feature the user describes in this Laravel 13 · Livewire 4 · Flux · Tailwind monorepo.

1. Read `AGENTS.md`. Read `.github/docs/architecture/ARCHITECTURE.md` first for structural changes. Read `.github/instructions/*.instructions.md` files whose `applyTo` matches the paths you will touch.
2. **Locate the owner before writing anything.** A change touching both `entities13` and `projects13` belongs in the `entity-projects13` bridge; each domain package must keep working without the other. If the boundary cannot be respected, stop and ask.
3. Plan briefly: files to create/change, validation, authorization, i18n keys, test cases. Ask if the scope is ambiguous.
4. Write failing tests first, in the owning package (`tests/Feature/...` there, not `projects/demo13`), run them narrowest-first. App-only changes use `DX php artisan make:test --phpunit Name` + `DX php artisan test --compact <path>`; package changes run in the package with its own PHPUnit (see `docker-commands`).
5. Implement the smallest change following sibling-file conventions inside the owning package: `XController` + `XTrashController` + `XArchivedController` (when the model has an `active` flag), Form Requests, policy, `src/Queries/*/XListQuery`, `src/Transformers/XListTransformer`. Reuse `<x-list.*>` from `basics13` and use Flux components where one exists; keep controllers thin.
6. Every user-facing string goes in the owning package's `resources/lang/*.json` for all 4 locales (same key and `:placeholders`, default `eu`). Keep existing `data-test` names; row actions pass the edit payload via `data-payload`.
7. After PHP edits: `DX ./vendor/bin/pint --dirty --format agent`. Rerun the touched tests after each fix.
8. If views or `resources/js` changed: `docker compose run --rm --no-deps --entrypoint npm laravel13-npm run build`.
9. Finish with a short report: files changed, tests added, commands executed and their real results. Never claim a test passed unless it ran.

Before adding a dependency, base folder or new abstraction (repository, DTO, service layer), ask the user: the architecture rejects them without a concrete need. Use `search-docs` before version-specific Laravel APIs.
