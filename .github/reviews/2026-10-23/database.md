# Database — add-session-log-and-settings (raw)

Package tests run on SQLite in memory (packages/basics13/phpunit.xml:35-36); prod/host MariaDB.

- DB-001 MEDIUM HIGH — Plan seeds only sessions.view, settings.view, settings.edit but grants settings.manage; 000006:60-62 indexes $permissionIds without guard → undefined key. Seed MODULES×ACTIONS.
- DB-002 MEDIUM HIGH — RateLimiter timer from first hit (dup LAR-001). Separate locked key via Cache::add when hit()===max.
- DB-003 MEDIUM HIGH — Double logout (two tabs/double submit): find-then-save re-sets logged_out_at → guard throws → 500. Use conditional update whereNull('logged_out_at'); unique session_key.
- DB-004 MEDIUM HIGH — Spec "por ninguna vía" false: query-builder bypass, nullOnDelete on force delete (UserTrashController.php:49-52); design.md:32 "en la práctica se conserva" false.
- DB-005 MEDIUM MEDIUM — `login` not truncated; LoginRequest has no max → MariaDB strict "Data too long" 500, row lost.
- DB-006 LOW HIGH — No updated_at: must set UPDATED_AT = null or insert fails / guard always throws.
- DB-007 LOW HIGH — Native enum columns differ from convention; use string + PHP enum casts.
- DB-008 LOW MEDIUM — Forever cache vs insertOrIgnore seeds; use upsert keeping value; forget cache in seeds.
- DB-009 LOW MEDIUM — null/'' integer cast to 0 → max_login_attempts=0 locks everyone; fall back to default; min:1.
- DB-010 LOW MEDIUM — INSERT IGNORE silently truncates long description in varchar on MariaDB; use text.
- DB-011 LOW HIGH/MEDIUM — Order by created_at, id; date-to filter exclusive of last day / sargable form.
- DB-012 INFO — Spike count cheap with index; Cache::add atomic on database/redis store.
- DB-013 INFO HIGH — Action list omits `impersonate` already seeded by 000006:19.
