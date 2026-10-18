---
name: project-rules
description: "Core project conventions: stack, structure, naming, architecture guardrails. Load first in every conversation."
---

# Project Rules (Core)

## Stack

Laravel 13 · PHP 8.4 · Livewire 4 · Flux (free) · Tailwind 4 · PHPUnit · Dusk · Pint.

## Conventions

- Be concise. Create docs only if explicitly asked.
- Follow sibling files' structure, naming and conventions; reuse existing components.
- Do not add base folders or change dependencies without approval.
- Use descriptive names.
- **No speculative architecture**: no repository, DTO, interface, service or domain layer, no new abstraction and no extra indirection without a concrete need; reuse what the resources already have. Ask before any of those.
- Full list in `.github/docs/review-rules.md` and `ARCHITECTURE.md`.

## Package Boundaries

- Follow the mandatory package boundaries in [AGENTS.md](../../../AGENTS.md#package-boundaries-mandatory).
- `entities13` and `projects13` stay independent; entity/project coupling belongs in `entity-projects13`. The bridge knows the domains, never the reverse.
- Shared infrastructure in `basics13` is allowed, not cross-domain business logic.
- Before editing `packages/**`, read [package instructions](../../instructions/packages.instructions.md) and [architecture](../../docs/architecture/ARCHITECTURE.md). This applies to implicit coupling in schema, payloads, UI and tests as well as imports and Composer dependencies.

## Workspace Layout

- Workspace root: `l13/` — this repo holds only the AI setup: `AGENTS.md`, `.github/`, `openspec/`, `.vscode/`.
- Laravel app: `projects/demo13/` (a separate repo; it only hosts auth/Fortify, `User` and settings).
- Shared packages: `packages/{basics13,entities13,projects13,entity-projects13}` — each a separate repo.
- `projects/`, `packages/` and `testing13/` are gitignored from `l13`, so a change in a package is committed **in that package's repo**, never here.
- Application paths in skills are relative to `projects/demo13/`; package paths are relative to the owning package under `packages/`.
- Customization paths (`.github/`, `.opencode/`, `.vscode/`) are relative to workspace root (`l13/`)
- `testing13/mount.sh` maintains `projects/packages -> ../packages` so Composer's relative symlinks created under `/app-docker` also resolve on the host. Keep the Docker paths `/app-docker` and `/packages`.

## Project Map

Entities → Projects → Epics (+ epic comments).

- Soft deletes, trash list, restore, force delete.
- `active` flag with its own archived list plus archive/activate.
- Names unique among active records (DB-enforced).

Per resource, inside its owning package (`X` = resource name):

- `src/Models/X.php`, `src/Policies/XPolicy.php`, `src/Http/Requests/{XRequest,XListRequest,XRestoreRequest}`
- `src/Http/Controllers/{XController,XTrashController,XArchivedController}` (archived extends `Basics13\Http\Controllers\ArchivedController`)
- `src/Queries/Xs/XListQuery.php`, `src/Transformers/XListTransformer.php`,
  `database/factories/XFactory.php`, `database/migrations/`, `database/seeders/`
- Views: `resources/views/xs/{list,form}.blade.php`; shared parts come from `<x-list.*>` / `<x-forms.*>` in `basics13`
- Duplicate-name races: `Basics13\Support\Database\UniqueConstraintViolation`
- i18n: `resources/lang/*.json`, 4 locales (en/es/eu/fr), parity checked by a test
- Tests live in the owning package's `tests/{Feature,Unit}`; Dusk in `tests/Browser`
- Structural changes: read `.github/docs/architecture/ARCHITECTURE.md` first

## Boost / Docs

- Prefer Boost MCP tools over shell: `database-query` (read-only), `database-schema`, `get-absolute-url`, `browser-logs`, `search-docs`.
- `search-docs` before version-specific Laravel-ecosystem APIs (skip for copy/styling). Broad topic queries, no package names in the query, scope with `packages`. Reuse earlier results.
- Zone rules live in `.github/instructions/*.instructions.md` (`models`, `http`, `views`, `tests`, `packages`) at the workspace root and cover the nested application and packages: Copilot injects them automatically by `applyTo`; any other agent reads the one matching the paths it edits before touching them.
- Reviewer agent sources live in `.github/agents/`. OpenCode and Claude read direct symlinks in `.opencode/agents/` and `.claude/agents/`. Skills are symlinked from `.github/skills/` to each client's native skills directory. The linked paths are read-only in VS Code; edit only the originals in `.github/`. Content changes are visible through the links immediately. Run `bash testing13/agents/link-opencode-assets.sh` and `bash testing13/agents/link-claude-assets.sh` to rebuild links after adding, deleting or renaming assets; both scripts locate the repository when run from another directory. The scripts do not adapt agent metadata or permissions between clients.
- Guided flows (new-feature, new-resource, full-review, fix-review, fix-tests, consistency-review, ux-implement) are skills in `.github/skills/<name>/SKILL.md`, not `/commands`: activate them by skill, not with a slash command.
- Activate the matching skill in `.github/skills` for its domain. No verification scripts/tinker when tests cover it.
