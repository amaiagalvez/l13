---
applyTo: "**/tests/**"
paths:
  - "**/tests/**"
---

# Test rules

- Prefer Unit tests without database: anything testable without queries (rules, policies, requests, query structure, transformers, config, translations) goes in `tests/Unit`, built with `new Model`/`make()`. `tests/Unit` never uses a database trait or runs a query; the shared `Basics13\Tests\TestCase` fails the test if it does. Feature (`tests/Feature`) is for HTTP and for what needs real rows; declare its database trait in each test, never in a base `TestCase`.
- Dusk (`tests/Browser`) only for JS behavior. Tests live in the **owning package**, not in `projects/demo13/tests` (that one only holds auth, settings, dashboard and users).
- Duplicate-insert races are tested per resource for store, update and restore by simulating a duplicate-key `QueryException`.
- Dusk runs against `laravel_dusk` with a non-English default locale, so assert on translated text.
- Run it as `docker compose exec -e XDEBUG_MODE=off laravel13-dusk php artisan dusk [tests/Browser/...php]`, never in the default service: only that one gives the runner and the browser server the same `laravel_dusk` database and its own config cache.
