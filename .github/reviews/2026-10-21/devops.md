# DevOps / observability / dependencies — raw specialist report (2026-10-21)

Read-only. Corrections to the brief, as verified by the specialist:
- CI runs PHP 8.4 (projects/demo13/.github/workflows/tests.yml:120,253 and the package workflows).
- `.env` and `dc-data` are gitignored.
- Actions are SHA-pinned with `contents: read` and `persist-credentials: false`.
- `APP_DEBUG` defaults to false.
- log-viewer is gated in production.

### OPS-001: entities13 path repository is container-absolute, which breaks the package CI install
- Severity: HIGH (if committed) · Category: CI / dependencies · Confidence: HIGH
- File: packages/entities13/composer.json:51; packages/entities13/composer.lock:15
- Evidence (orchestrator ran `git diff`): an uncommitted, unstaged change switches `"url": "../packages/basics13"` to `"/packages/basics13"` and adds `"name": "basics13"`. The lock dist URL changes to match. The sibling packages use `../packages/basics13`. `/packages` exists only inside the Docker mount.
- Impact: once committed, `composer install` in packages/entities13/.github/workflows/tests.yml cannot resolve basics13, so pint, phpstan and phpunit never run.
- Recommendation: revert to `../packages/basics13` and regenerate the lock in that layout before committing.

### OPS-002: Host CI checks out packages at a moving HEAD, and package pushes never run Dusk
- Severity: MEDIUM · Confidence: HIGH
- File: projects/demo13/.github/workflows/tests.yml:89-115, 222-248, 310-313; packages/entities13/.github/workflows/tests.yml:76-77
- Problem: package checkouts have no `ref`, and the package workflows skip the Browser suites.
- Recommendation: dispatch the host workflow from package pushes, or record the gap as known debt.

### OPS-003: Package PHPStan gates are red
- Severity: MEDIUM · Confidence: HIGH (orchestrator ran PHPStan)
- File: packages/basics13/.github/workflows/tests.yml:64-65; packages/entities13/.github/workflows/tests.yml:73-74
- Problem:
  - basics13 has 3 errors, all in tests.
  - entities13 has 132 `ignore.unmatched` errors from a staged, uncommitted `phpstan-baseline.neon`.
  - The host `types:check` does not cover the packages.
- Recommendation: fix the basics13 test errors, drop or regenerate the entities13 baseline, and confirm the package CI runs green.

### OPS-004: Dev tooling is in the production `require` of basics13
- Severity: MEDIUM · Confidence: HIGH (presence)
- File: packages/basics13/composer.json:9,12 (`laravel/chisel ^0.1.0`, `laravel/boost ^2.10`); projects/demo13/composer.lock `packages` section
- Impact: `--no-dev` installs still ship Boost (an MCP server) and the 0.x Chisel package.
- Recommendation: move Boost and Chisel to `require-dev` unless they are needed at runtime. This is a dependency change, so it needs approval.

### OPS-005: Compose helper profiles delete the lockfiles
- Severity: MEDIUM · Confidence: HIGH
- File: projects/demo13/docker-compose.yml:78-82 (`rm -f composer.lock`), 137-141 (`rm -f package-lock.json`)
- Impact: running one of these profiles silently upgrades every dependency, and local diverges from CI (`npm ci`).
- Recommendation: install from the existing lock, and keep regeneration in an explicitly named profile.

### OPS-006: ARCHITECTURE.md says CI runs PHP 8.3, and the `^8.3` constraint is never tested
- Severity: LOW · Confidence: HIGH
- File: .github/docs/architecture/ARCHITECTURE.md:176; projects/demo13/composer.json:12
- Recommendation: update the docs, and either narrow the constraint to `^8.4` or add an 8.3 CI leg.

### OPS-007: App and Vite ports bind 0.0.0.0
- Severity: LOW (local dev only) · Confidence: HIGH
- File: projects/demo13/docker-compose.yml:10-11, 129-130, 154-155
- Recommendation: bind to `127.0.0.1` unless LAN access is intended.

### OPS-008: `.env.example` mixes `APP_ENV=local` with debug-level daily logs and hardcoded UIDs, and has no production guidance
- Severity: LOW · File: projects/demo13/.env.example:2,18-23,73-74

### OPS-009: Frontend dependency hygiene
- Severity: LOW · Confidence: MEDIUM
- File: projects/demo13/package.json:12-15
- Problem: `concurrently` is in `dependencies` instead of `devDependencies`, and select2 4.0.13 is paired with jQuery 4 without verified compatibility.

### OPS-010: Starter-kit leftovers in composer.json
- Severity: LOW/INFO · File: projects/demo13/composer.json:3-5, 13-16, 96-99, 118
- Problem: the package is still named `laravel/livewire-starter-kit`, there is a sqlite create hook, the pest plugin is allowed, and `amaia/*` packages are required as `"*"`.

### OPS-011 (INFO): Dusk image and health check notes
- The Dusk image runs an unpinned chromium.
- The app container has no healthcheck.
- There is no failed-job alerting, which is acceptable because no jobs exist.
