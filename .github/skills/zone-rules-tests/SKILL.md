---
name: zone-rules-tests
description: "Pointer to the test zone rules (prefer Unit tests without database, Feature for HTTP and real rows, Dusk only for JS behavior, duplicate-insert races, Dusk against laravel_dusk with non-English locale). Load when editing tests/**."
---

# Zone Rules: Tests

Two files own this zone. Read them in this order; this skill holds no rules of its own, so it cannot drift from them.

1. **`.github/instructions/tests.instructions.md`** — the always-on rules for this zone. Copilot injects it automatically by `applyTo`.
2. **`.github/skills/test-rules/SKILL.md`** — the full decision table for test location and the writing checklist.

Package suites run in their own PHPUnit (sqlite in memory), not through `php artisan test`: see the `docker-commands` skill.
