---
name: php-conventions
description: "PHP style, Chisel markers, type hints, PHPDoc, factories, Pint workflow. Load when editing PHP files."
---

# PHP Conventions

## Chisel
- `chisel.php` / `chisel-paths.php` strip unused code marked with `/* @chisel-... */` … `/* @end-chisel-... */`.
- Those markers are **machine-read**: never remove or reformat them, and never regenerate `chisel.php` unless asked.
- **Integrity test**: `projects/demo13/tests/chisel-integrity-test.php` validates all markers are properly paired. It lives next to the code it scans, and runs in the `demo13` CI job. Run via:
  ```bash
  # From projects/demo13/:
  php tests/chisel-integrity-test.php
  # Or in Docker:
  docker compose -f docker-compose.yml run --rm --no-deps -e XDEBUG_MODE=off --entrypoint php laravel13 tests/chisel-integrity-test.php
  ```
  From workspace root (l13/), prefix with `-f projects/demo13/docker-compose.yml`. The scan excludes root-level `.git`, `dc-data`, `node_modules`, `storage`, and `vendor` directories.

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