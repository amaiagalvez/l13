# DevOps Review — 2026-10-24 (read-only)

Saved by the orchestrator from the devops-reviewer subagent's report (the subagent session had no file-write tool). Scope verified: `projects/demo13/docker-compose.yml`, `Dockerfile.dusk`, `.env.example`, `.env` (dev), `composer.json`, `.github/{workflows/tests.yml,dependabot.yml}`; all four packages' CI and `composer.json`; `testing13/` scripts; `basics13` log-viewer gate. `composer audit` was executed afterwards by the orchestrator in Docker: **no advisories** (host).

## Findings

- **OPS-001 · MEDIUM · No dependency advisory check in CI · projects/demo13/.github/workflows/tests.yml (whole file)** — No `composer audit`/`npm audit` step in any of the 5 repos' CI; Dependabot is weekly with a 5-day cooldown, so advisory detection lags up to ~12 days. Recommendation: add `composer audit --no-interaction --abort-on-unfixable` to the `ci` job (and a scheduled run). Confidence: HIGH (absence verified by grep across all workflows).
- **OPS-002 · LOW · CI known debt (documented) · projects/demo13/.github/workflows/tests.yml:89–135** — Sibling packages are checked out at default-branch HEAD if the triggering branch does not exist; Dusk suites run only on the host. Matches `ARCHITECTURE.md` "Known Technical Debt". Recorded for completeness. Confidence: HIGH.
- **OPS-003 · LOW · Unnecessary dev dependency · projects/demo13/composer.json:30** — `laravel/sail ^1.53` is unused (project runs on `webdevops/php-apache-dev`; `boost.json` sets `"sail": false`; no other reference). Only adds Dependabot noise and a confusing second Docker story. Recommendation: remove or document why it is kept. Confidence: HIGH.
- **OPS-004 · LOW · Abandoned dev dependency · projects/demo13/docker-compose.yml:273–278** — `mailhog/mailhog:v1.0.1` is the last release of an archived project. Dev-only, bound to `127.0.0.1:8025`, no production risk. Recommendation: replace with Mailpit (drop-in, same UI/SMTP ports) when convenient. Confidence: MEDIUM.
- **OPS-005 · LOW · Dev image reused as prod base · projects/demo13/docker-compose.yml:3,9–10** — `webdevops/php-apache-dev` publishes port 80 on all interfaces; local-development stack per ARCHITECTURE.md "Production Assumptions". Optionally bind `127.0.0.1:80:80` for consistency with db/phpMyAdmin/MailHog. Confidence: HIGH.
- **OPS-006 · LOW · Dev credentials duplicated · docker-compose.yml:244–247; tests.yml:64,70** — Root password and `laravel/laravel` hardcoded in the local stack and both CI jobs; documented dev-only. The duplication invites drift. Recommendation: single source or masked repo variables for CI. Confidence: HIGH.
- **OPS-007 · LOW · Non-strict dev database · docker-compose.yml:242** — `--sql-mode=""` disables MariaDB strict mode locally while CI's MariaDB runs with defaults: invalid values that error in CI/prod truncate silently in local dev. Recommendation: drop the override so dev matches CI/prod. Confidence: MEDIUM.
- **OPS-008 · INFO · Lockfile-deleting helper services · docker-compose.yml:78–82,137–141** — `laravel13-composer` deletes the committed `composer.lock` and rebuilds from unconstrained resolutions. Profile-gated and clearly named; recommendation: back up rather than delete, or require an explicit confirmation. Confidence: HIGH.
- **OPS-009 · LOW · Unconstrained internal package constraints · projects/demo13/composer.json:13–16** — `amaia/*: "*"` float freely if the packages are ever published/versioned or the path repos removed. Recommendation: `^1.0` for symmetry with the packages' own `"version": "1.0.0"`. Confidence: HIGH.

## Verified as good

- Environment: `.env.example` ships `APP_DEBUG=false`, empty `APP_KEY`; dev `.env` debug on as expected.
- Observability: debugbar and pail are `require-dev` only; dusk service forces `DEBUGBAR_ENABLED=false`; log viewer gated by email allowlist (default deny-all) with download/delete hard-denied; `LOG_CHANNEL=daily`.
- CI hygiene: actions pinned by full SHA; `permissions: contents: read`; `persist-credentials: false`; concurrency + timeouts; coverage artifacts uploaded `if: always()`; Dusk job uploads screenshots/console logs/`laravel.log` on failure.
- Docker port hygiene: db (13306), phpMyAdmin (4000), MailHog (8025) loopback-bound.
- `testing13/` destructive operations confined to coverage/output dirs and profile-gated containers; Dusk never auto-run.
