# Project Context

## Purpose
Monorepo Laravel 13: Customers → Projects → Epics (+ epic comments). App en `projects/demo13/`,
dominios en `packages/`. CRUD canónico con soft deletes, papelera y flag de estado, con fronteras
estrictas entre paquetes.

## Tech Stack
- Laravel 13, PHP 8.4, Livewire 4, Flux (free), Tailwind 4
- PHPUnit 12, Dusk 8, Pint, Larastan/PHPStan nivel 7
- Todo en Docker: `DX` = `docker compose exec -e XDEBUG_MODE=off laravel13` (desde `projects/demo13`)

## Project Conventions
- Estilo, Chisel (`#[Fillable]`, `#[UsePolicy]`, `#[UseFactory]`) y Pint/PHPStan → skill `php-conventions`
- Petición HTTP, policies, queries y transformers → `zone-rules-http`
- Vistas, Flux, i18n (es/eu/fr/en, paridad verificada, euskera obligatorio) → `zone-rules-views` + `ux-design`
- Tests (Feature > Unit, Dusk solo JS, carreras con `UniqueConstraintViolation`) → `test-rules`
- Sin arquitectura especulativa (repository, DTO, service layer) → `.github/docs/review-rules.md`

## Architecture Patterns
- `basics13` = infra neutral · `customers13`/`projects13` = dominios independientes ·
  `customer-projects13` = bridge. Detalle normativo: `specs/package-boundaries`.
- Cada dominio expone `*Integration` + `Null*Integration` para funcionar sin el bridge.
- Un recurso = Controller + Trash/Archived + Requests + Policy + ListQuery + Transformer + vistas.

## Domain Context
- Nombre único entre registros **activos** (índice parcial), no global
- Soft delete + restore + borrado definitivo bloqueado si hay relaciones
- Copia en euskera obligatoria; 4 locales con paridad verificada por test
- Vistas, `lang/` y tests viven **en los paquetes**, no en `demo13`

## Important Constraints
- Frontera de paquetes: bridge → dominios, nunca al revés
- Una sesión = una tarea. Specs antes que código; proposals antes que implementar
- Comandos y clientes (Copilot/Claude/OpenCode) → `AGENTS.md`

## External Dependencies
- Boost MCP (laravel-boost): queries, schema, docs
- Reglas: `.github/instructions/*.instructions.md` · skills: `.github/skills/`
- Arquitectura: `.github/docs/architecture/ARCHITECTURE.md` · manual SDD: `.github/docs/spec_sdd.md`

## OpenSpec
Mapa de capabilities → `AGENTS.md` (sección *Spec-Driven Development*). Las specs son invariantes, no
código: si una regla ya está en el código, una skill o un test, no se repite en `specs/`.