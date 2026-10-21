# Project guidelines

Laravel 13 · PHP 8.4 · Livewire 4 · Flux (free) · Tailwind 4 · PHPUnit · Dusk · Pint.
Be concise. Create docs only if explicitly asked. Follow sibling files' structure, naming and conventions; reuse existing components. Do not add base folders or change dependencies without approval. Use descriptive names.
No speculative architecture: no repository, DTO, interface, service or domain layer, no new abstraction and no extra indirection without a concrete need; ask first. Full list in `.github/docs/review-rules.md` and `.github/docs/architecture/ARCHITECTURE.md` (read it before structural changes).

## Workspace
- `l13/` holds only the AI setup (`AGENTS.md`, `.github/`, `openspec/`, client folders). The app `projects/demo13/` (auth, `User`, settings) and each `packages/{basics13,entities13,projects13,entity-projects13}` are separate git repos, gitignored here: commit a change in the repo that owns it.
- Application paths are relative to `projects/demo13/`, package paths to the owning package, customization paths to `l13/`.

## Package boundaries (mandatory)
- `entities13` and `projects13` are independent: never depend on each other or on `entity-projects13`.
- Entity/project coupling lives only in the bridge `entity-projects13`: bridge -> domains, never the reverse.
- `basics13` is domain-neutral shared infrastructure; never move cross-domain business logic there.
- Coupling includes non-import forms (schema, payload keys, UI conditions, tests…). Before editing `packages/**`, read `.github/instructions/packages.instructions.md`. If the boundaries cannot be respected, stop and ask.

## Commands
Everything runs in Docker from `projects/demo13/` (from `l13/`: `docker compose -f projects/demo13/docker-compose.yml ...`). `DX` = `docker compose exec -e XDEBUG_MODE=off laravel13`. Always pass `--no-interaction`; create files with `DX php artisan make:*`. Details, test runs and guard rails: skill `docker-commands`.

## Project map
Entities → Projects → Epics (+ epic comments). Soft deletes with trash, restore and force delete; `active` flag with archived list; names unique among active records (DB-enforced).
Per resource, in its owning package: `XController` + `XTrashController` + `XArchivedController`, `src/Http/Requests`, `src/Policies`, `src/Queries/*/XListQuery`, `src/Transformers`, `database/{factories,migrations,seeders}`, `resources/views/xs/{list,form}.blade.php`, `resources/lang/{es,eu,fr,en}.json`, `tests/{Feature,Unit,Browser}`.

## Where the rules live
- Zone rules: `.github/instructions/*.instructions.md`, the single source. Copilot applies them by `applyTo`, Claude by `paths` (`.claude/rules`), OpenCode through the `zone-rules-*` skills. Read the matching one before editing a path.
- Skills: `.github/skills/`; agents: `.github/agents/`. Client folders only link to them: edit only `.github/`.
- Prefer Boost MCP tools over shell (`database-query`, `database-schema`, `search-docs`, `browser-logs`). Use `search-docs` before version-specific Laravel-ecosystem APIs. No verification scripts or tinker when tests cover it.

## Spec-Driven Development (OpenSpec)
Specs are invariants the code, tests and skills do not already express. Changes go in `openspec/changes/<id>/`; never edit `openspec/specs/` by hand. Project context and rules: `openspec/config.yaml`. Workflow: `openspec-*` skills. Manual: `.github/docs/spec_sdd.md`.

Read only the row you touch:

| Touching | Read |
|----------|------|
| Models, casts, columns, relationships | `specs/basics-trash-archived`, `specs/<domain>-domain` |
| Controllers, requests, policies, queries | `specs/basics-crud-base`, `specs/basics-trash-archived`, `specs/<domain>-domain` |
| Trash, restore, archive, force delete | `specs/basics-trash-archived` |
| Auth, users, account settings | `specs/basics-auth` |
| Flux views, shared components | `specs/basics-shared-views` |
| Epics and comments | `specs/epics-comments` |
| Planning and timeline (read-only) | `specs/planning-timeline` |
| Entity↔project relations, cross-package imports | `specs/entity-projects-bridge`, `specs/package-boundaries` |
