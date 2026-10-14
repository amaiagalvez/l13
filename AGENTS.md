<!-- OPENSPEC:START -->
# OpenSpec Instructions

These instructions are for AI assistants working in this project.

Always open `@/openspec/AGENTS.md` when the request:
- Mentions planning or proposals (words like proposal, spec, change, plan)
- Introduces new capabilities, breaking changes, architecture shifts, or big performance/security work
- Sounds ambiguous and you need the authoritative spec before coding

Use `@/openspec/AGENTS.md` to learn:
- How to create and apply change proposals
- Spec format and conventions
- Project structure and guidelines

Keep this managed block so 'openspec update' can refresh the instructions.

<!-- OPENSPEC:END -->

# Project guidelines

Laravel 13 · PHP 8.4 · Livewire 4 · Flux (free) · Tailwind 4 · PHPUnit · Dusk · Pint.
Be concise. Create docs only if explicitly asked. Follow sibling files' structure, naming and conventions; reuse existing components. Do not add base folders or change dependencies without approval. Use descriptive names.
No speculative architecture: no repository, DTO, interface, service or domain layer, no new abstraction and no extra indirection without a concrete need; reuse what the resources already have. Ask before any of those. Full list in `.github/docs/review-rules.md` and `ARCHITECTURE.md`.

## Package boundaries (mandatory)
- Domain packages must work independently. `customers13` and `projects13` must not depend on each other or on `customer-projects13`.
- Customer/project coupling belongs in the existing bridge `customer-projects13`, which may reference both domain packages. Dependency direction: bridge -> domains, never domains -> bridge or domain -> domain.
- Existing shared infrastructure in `basics13` is allowed; do not move cross-domain business logic there to bypass the bridge.
- Dependencies include Composer entries, imports/FQCNs, container lookups, relationships, foreign keys, table queries, routes, views, assets, translations, factories, seeders and tests. Domain-specific field names, payload keys and UI conditions also create coupling, even without an import.
- Base packages expose domain-neutral extension points and standalone defaults. Reuse their existing contracts and configuration; the bridge supplies concrete integrations, relationships, schema additions and UI. Follow `CustomerProjectsIntegration`.
- Before changing a package, identify the owner: behavior needing both domains goes in the bridge. Preserve behavior with and without the bridge installed. Test standalone behavior in the base package and cross-domain behavior in the bridge or host integration tests.
- Do not add another abstraction, bridge or package dependency without approval. Existing coupling is not permission to extend it; if the boundaries cannot be respected, stop and ask.
- Before editing `packages/**`, read `.github/instructions/packages.instructions.md` and `.github/docs/architecture/ARCHITECTURE.md`.

## Commands
The workspace root is `l13`; the Laravel application is in `projects/demo13/`, and shared packages are in `packages/`. Application paths below and in skills are relative to `projects/demo13/`; customization paths (`.github/`, `.opencode/`, `.vscode/`) are relative to the workspace root.
Everything runs in Docker. Run the application commands below and in skills from `projects/demo13/` (`cd projects/demo13` from the workspace root). From `l13/`, use `docker compose -f projects/demo13/docker-compose.yml ...` instead. `DX` = `docker compose exec -e XDEBUG_MODE=off laravel13`.
`testing13/mount.sh` maintains `projects/packages -> ../packages` so Composer's relative symlinks created under `/app-docker` also resolve on the host. Keep the Docker paths `/app-docker` and `/packages`.
- Start: `docker compose up -d laravel13`. If `docker compose exec laravel13 ...` fails because the container will not stay up (a cron in the base image cannot drop privileges), run the command in a throwaway container instead: `docker compose run --rm --no-deps -e XDEBUG_MODE=off --entrypoint ./vendor/bin/phpunit laravel13 --testsuite Unit`.
- Artisan/Composer/PHPUnit: `DX php artisan ...`, `DX composer ...`. Always pass `--no-interaction`; create files with `DX php artisan make:*`.
- Package versions: `DX composer show --direct`, or package.json. Use APIs of the installed major version.

## Project map
Customers → Projects → Epics (+ epic comments). Soft deletes, trash list, restore, force delete. `active` flag with its own list plus deactivate/reactivate. Names unique among active records (DB-enforced).
- Per resource: `XController` + `XTrashController` + `XInactiveController` (extends the shared `InactiveController`), `app/Http/Requests`, `app/Policies`, `app/Queries/*/XListQuery`, `app/Transformers`.
- Views: `resources/views/{customers,projects,epics}/{list,form}.blade.php`; shared parts in `resources/views/components/list/`.
- Duplicate-name races: `App\Support\Database\UniqueConstraintViolation`.
- i18n: `lang/*.json`, 4 locales, parity checked by a unit test.
- Tests: `tests/Feature`, `tests/Unit`, Dusk in `tests/Browser`.
- Structural changes: read `.github/docs/architecture/ARCHITECTURE.md` first.

## Skills (source of truth for detailed rules)
Rules are organized as skills in `.github/skills/`. OpenCode loads them on demand; Copilot/Claude read this file and the zone rules below.

| Domain | Skill | Zone rules (applyTo) |
|--------|-------|----------------------|
| Core conventions | `project-rules` | — |
| PHP style, Chisel, Pint | `php-conventions` | — |
| Docker, DX, MCP Boost | `docker-commands` | — |
| Test location, traits, checklist | `test-rules` | `tests.instructions.md` |
| Models, relationships, soft deletes | `zone-rules-models` | `models.instructions.md` |
| HTTP, controllers, policies, queries | `zone-rules-http` | `http.instructions.md` |
| Views, forms, lists, Flux, i18n, UX | `zone-rules-views` | `views.instructions.md` |
| Test types, Dusk, races | `zone-rules-tests` | `tests.instructions.md` |
| Laravel best practices | `laravel-best-practices` | — |
| Testing best practices | `testing-best-practices` | — |
| Livewire, Flux UI, Tailwind, UX | `livewire-development`, `fluxui-development`, `tailwindcss-development`, `ux-design` | — |
| Guided flows | `new-feature`, `new-resource`, `full-review`, `fix-review`, `fix-tests`, `consistency-review`, `ux-implement` | — |
| SDD workflow | `openspec-workflow` | — |

## Spec-Driven Development (OpenSpec)

Specs = invariantes que el código, los tests y las skills ya no expresan. Cambios = proposals en
`openspec/changes/<id>/`; `openspec/specs/` no se edita a mano. Contexto: `openspec/project.md`.
Workflow completo y formato: `@/openspec/AGENTS.md`. Manual de uso: `.github/docs/spec_sdd.md`.
Client commands: `/openspec-proposal`, `/openspec-apply`, `/openspec-archive`.

Mapa (lee solo la fila que toque, no el fichero entero):

| Toca | Lee |
|------|------|
| Modelos, casts, columnas, relaciones | `specs/basics-support`, `specs/<dominio>-domain` |
| Controllers, requests, policies, queries | `specs/basics-crud-base`, `specs/basics-trash-archived`, `specs/<dominio>-domain` |
| Papelera, restore, archivado, force delete | `specs/basics-trash-archived` |
| Vistas Flux, componentes compartidos | `specs/basics-shared-views` |
| Epics y comentarios | `specs/epics-comments` |
| Planning y timeline (solo lectura) | `specs/planning-timeline` |
| Relations customer↔project, imports entre paquetes | `specs/customer-projects-bridge`, `specs/package-boundaries` |

Reglas: un proposal = un cambio atómico = un PR · deltas con `## ADDED|MODIFIED|REMOVED Requirements`
y un `#### Scenario:` por requirement · bug que restaura lo especificado se arregla directo · tras
mergear, `openspec archive <id> --yes` actualiza las specs.

## When to Load Each Skill (OpenCode)

| Task | Skills to Load |
|------|----------------|
| **Any conversation** | `project-rules` (auto-loaded via `opencode.jsonc`) |
| Edit PHP files (models, controllers, etc.) | `php-conventions` |
| Run Docker commands, tests, builds, MCP | `docker-commands` |
| Write or modify tests | `test-rules` + `testing-best-practices` |
| Edit `app/Models/**` | `zone-rules-models` |
| Edit `app/Http/**`, `Policies`, `Queries`, `Transformers` | `zone-rules-http` |
| Edit `resources/views/**`, `lang/**` | `zone-rules-views` + `ux-design` |
| Edit `tests/**` | `zone-rules-tests` |
| New CRUD resource | `new-resource` + `laravel-best-practices` |
| New feature (action, filter, column, email) | `new-feature` + `laravel-best-practices` |
| SDD: create / apply / archive a proposal | `openspec-workflow` |
| Full code review | `full-review` |
| Fix review findings | `fix-review` |
| Fix failing tests | `fix-tests` |
| Check cross-resource consistency | `consistency-review` |
| Implement UX improvements | `ux-implement` + `ux-design` |
| Livewire component work | `livewire-development` + `fluxui-development` |
| Tailwind styling | `tailwindcss-development` |
| Fortify auth changes | `fortify-development` |

**Usage:** `skill("skill-name")` loads the skill for the current conversation. Multiple skills can be loaded.

## Boost / docs
- Prefer Boost MCP tools over shell: `database-query` (read-only), `database-schema`, `get-absolute-url`, `browser-logs`, `search-docs`.
- `search-docs` before version-specific Laravel-ecosystem APIs (skip for copy/styling). Broad topic queries, no package names in the query, scope with `packages`. Reuse earlier results.
- Zone rules live in `.github/instructions/*.instructions.md` (`models`, `http`, `views`, `tests`) at the workspace root and cover the nested application and packages: Copilot injects them automatically by `applyTo`; any other agent reads the one matching the paths it edits before touching them.
- Reviewer agent sources live in `.github/agents/`. OpenCode and Claude read direct symlinks in `.opencode/agents/` and `.claude/agents/`. Skills are symlinked from `.github/skills/` to each client's native skills directory. The linked paths are read-only in VS Code; edit only the originals in `.github/`. Content changes are visible through the links immediately. Run `bash testing13/agents/link-opencode-assets.sh` and `bash testing13/agents/link-claude-assets.sh` to rebuild links after adding, deleting or renaming assets; both scripts locate the repository when run from another directory. The scripts do not adapt agent metadata or permissions between clients.
- Guided flows (new-feature, new-resource, full-review, fix-review, fix-tests, consistency-review, ux-implement) are skills in `.github/skills/<name>/SKILL.md`, not `/commands`: activate them by skill, not with a slash command.
- Activate the matching skill in `.github/skills` for its domain. No verification scripts/tinker when tests cover it.