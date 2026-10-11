---
name: zone-rules-tests
description: "Test rules: prefer Feature tests, Dusk only for JS behavior, duplicate-insert races, Dusk against laravel_test with non-English locale. Load when editing tests/**."
---

# Zone Rules: Tests

## General
- Prefer Feature tests; Dusk (`tests/Browser`) only for JS behavior.
- Duplicate-insert races are tested per resource for store, update and restore by simulating a duplicate-key `QueryException`.
- Dusk runs against `laravel_test` with a non-English default locale, so assert on translated text.
- Run Dusk as: `docker compose exec -e XDEBUG_MODE=off laravel13-dusk php artisan dusk [tests/Browser/...php]`, never in the default service: only that one gives the runner and the browser server the same `laravel_test` database and its own config cache.

## Test Location Rules
See `test-rules` skill for the full decision table and checklist.