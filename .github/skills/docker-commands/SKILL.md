---
name: docker-commands
description: "Docker commands, DX alias, MCP Boost, test execution, build commands. Load when running commands in Docker."
---

# Docker Commands

## Base Commands
Everything runs in Docker. Run from `projects/demo13/` (`cd projects/demo13` from workspace root).
From `l13/`, use `docker compose -f projects/demo13/docker-compose.yml ...` instead.

`DX` = `docker compose exec -e XDEBUG_MODE=off laravel13`

### Start
```bash
docker compose up -d laravel13
```
If `docker compose exec laravel13 ...` fails because the container will not stay up (a cron in the base image cannot drop privileges), run the command in a throwaway container instead:
```bash
docker compose run --rm --no-deps -e XDEBUG_MODE=off --entrypoint ./vendor/bin/phpunit laravel13 --testsuite Unit
```

### Artisan / Composer / PHPUnit
```bash
DX php artisan ...
DX composer ...
```
- Always pass `--no-interaction`
- Create files with `DX php artisan make:*`

### Package Versions
```bash
DX composer show --direct
```
Or check `package.json`. Use APIs of the installed major version.

## Test Execution
- Narrowest: `DX php artisan test --compact <path|--filter=name>`
- Parallel locally (never in CI — shared DB): `DX php artisan test --parallel --compact <path>` (flags before path)
- Rerun after each fix

### Architecture Tests (run before calling a change done)
```bash
DX php artisan test --compact tests/Unit/ArchitectureTest tests/Unit/ValidationCoverageTest tests/Unit/ModelSchemaParityTest tests/Unit/ResourceUniformityTest
```
They guard: per-resource set, validated input, model↔schema parity, cross-resource uniformity.

### CI Coverage
- CI enforces ≥80% total code coverage, uploads Clover report
- Local check: `docker compose run --rm laravel13-phpunit`

### Static Analysis
```bash
DX ./vendor/bin/phpstan analyse
```
Level 9.

## Build Commands

### Profile-based services (original, run from VS Code tasks)
- Composer clean install: `docker compose --profile composer run --rm laravel13-composer`
- Pint reformat: `docker compose --profile reformat run --rm laravel13-reformat`
- PHPStan review: `docker compose --profile review run --rm laravel13-review`
- NPM clean build: `docker compose --profile npm-all run --rm laravel13-npm-all`
- NPM incremental build: `docker compose --profile npm run --rm laravel13-npm`
- Migrate fresh + seed: `docker compose --profile migrate-all run --rm laravel13-migrate-all`
- Migrate: `docker compose --profile migrate run --rm laravel13-migrate`
- PHPUnit with coverage: `docker compose --profile phpunit run --rm laravel13-phpunit`

### One-off commands via main service (alternative)
- NPM build: `docker compose run --rm --no-deps --entrypoint npm laravel13-npm run build`
- Pint: `docker compose run --rm laravel13 ./vendor/bin/pint`
- PHPStan: `docker compose run --rm laravel13 ./vendor/bin/phpstan analyse`
- Migrate: `docker compose run --rm laravel13 php artisan migrate`
- Migrate fresh + seed: `docker compose run --rm laravel13 php artisan migrate:fresh --seed`
- Tests with coverage: `docker compose run --rm laravel13 php artisan test --coverage`
- If UI doesn't refresh, ask user to start dev: `-p 5173:5173 ... run dev -- --host 0.0.0.0`

## MCP Boost (Laravel Boost)
The MCP server runs via `docker compose run --rm` (throwaway container) for reliability:
```bash
docker compose -f projects/demo13/docker-compose.yml run --rm -e XDEBUG_MODE=off --entrypoint php laravel13 artisan boost:mcp
```
This avoids the "container not staying up" issue with `docker compose exec`.