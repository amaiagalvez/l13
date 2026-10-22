---
name: php-conventions
description: "PHP style, type hints, PHPDoc, factories, Pint workflow. Load when editing PHP files."
---

# PHP Conventions

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