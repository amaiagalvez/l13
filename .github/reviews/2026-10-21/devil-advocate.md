# Devil's Advocate — consolidation (2026-10-21)

Line numbers were spot-checked for every HIGH and CRITICAL finding, and all match. The orchestrator verified the following by execution: the SEC-001 exploit, the `config:cache` failure, the CI triggers (master, PR and dispatch only), the skip causes and the PHPStan state.

Merges:
- SEC-001 ← SEC-001, FE-009
- SEC-002 ← SEC-002, SEC-003
- config:cache ← LAR-001, ARCH-001
- LIKE escape ← LAR-002, SEC-004, DB-003, BUG-006, API-002, MAINT-002, TEST-002
- state counts ← LAR-003, MAINT-001, TEST-003
- comment lock ← DB-001, BUG-003, API-006
- filter input ← API-001, LAR-004, SEC-005, FE-004, API-003, ARCH-004
- condition ← ARCH-006, MAINT-005, FE-003, TEST-005
- label props ← MAINT-004, FE-008, ARCH-007, FE-010
- PHPStan ← OPS-003, MAINT-003, TEST-008
- N+1 ← PERF-001, PERF-002, PERF-003
- docs ← DB-004, BUG-007, MAINT-008, MAINT-011, OPS-006
- model hooks ← DB-002, BUG-004
- race exceptions ← LAR-005, API-005, MAINT-010

Downgrades:
- FE-001 and FE-002: HIGH → MEDIUM. Functional bugs in an in-progress feature, with no security impact or data loss.
- TEST-001: HIGH → MEDIUM. A deliberate, documented coverage gap.
- OPS-001: HIGH → MEDIUM. Uncommitted.
- API-001: MEDIUM → LOW.
- SEC-003: MEDIUM → LOW, merged into SEC-002.
- FE-003: MEDIUM → LOW. Latent, because no route ever defines `{entity}` or `{project}`.
- OPS-002, OPS-004, OPS-005: MEDIUM → LOW.
- FE-006: MEDIUM → LOW.
- BUG-002: MEDIUM → LOW. The spec is ambiguous.

Rejected:
- BUG-005, LAR-006, FE-007: product decision in design.md.
- PERF-005: no EXPLAIN.
- PERF-006: negligible.
- MAINT-007: style.
- MAINT-006 config-hook part: intended extension point.
- ARCH-005 `'names'` part: generic word.
- TEST-006 fixture parts: style.
- API-007: report itself says leave as is.
- API-008: speculative until roles exist.
- OPS-008: local file preference.
- OPS-009: unsupported.
- OPS-010: cosmetic.
- SEC-006, OPS-007, OPS-011: documented local-dev only.
- FE-011 round-trip part: preference.

Overengineering vetoes:
- No static "isActiveFilter" predicate. Operator validation is enough.
- No cross-repo CI dispatch. Record it as debt.
- No integration scope for the timeline N+1. Pass `$project` instead.
- No callable class-strings to replace the config closures, because they keep the SEC-001 pattern.
- No roles to fix SEC-002.
