---
name: php-conventions
description: "PHP style, Chisel markers, type hints, PHPDoc, factories, Pint workflow. Load when editing PHP files."
---

# PHP Conventions

## Chisel
- `chisel.php` / `chisel-paths.php` strip unused code marked with `/* @chisel-... */` … `/* @end-chisel-... */`.
- Those markers are **machine-read**: never remove or reformat them, and never regenerate `chisel.php` unless asked.
- **Integrity test**: `testing13/tests/chisel-integrity-test.php` validates all markers are properly paired. Run via:
  ```bash
  # From workspace root (l13/):
  docker compose -f projects/demo13/docker-compose.yml run --rm --no-deps -e XDEBUG_MODE=off -v "$(pwd)/testing13:/testing13" -w /app-docker --entrypoint php laravel13 /testing13/tests/chisel-integrity-test.php
  ```
  From `projects/demo13/`, use `-f docker-compose.yml` and mount `"$(pwd)/../../testing13:/testing13"` instead. The main service does not mount `/testing13`; the explicit bind mount is required.
  The scan excludes root-level `.git`, `dc-data`, `node_modules`, `storage`, and `vendor` directories.

## Code Style
- Curly braces always.
- Constructor property promotion.
- Explicit return and parameter types.
- Enum keys: TitleCase.
- PHPDoc over inline comments (array shapes in PHPDoc).
- Named routes with `route()`.
- Models get factories (use existing states).
- Faker: follow existing style.

## Pint Workflow
- After editing PHP: `DX ./vendor/bin/pint --dirty --format agent`.
- **Never `--test` while writing code**: it only reports.
- A read-only review may use `--test`, which is the correct check there because it must not rewrite files.