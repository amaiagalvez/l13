# Architecture — add-session-log-and-settings (raw)

- ARC-001 HIGH HIGH — Impersonation (add-superadmin-user-impersonation tasks.md:20,28; design.md:37,61) calls Auth::login() → Login listener writes a row for the target from admin IP, overwrites session_key, clears target lockout counter. Skip when impersonating; add scenario.
- ARC-002 MEDIUM HIGH — Three changes edit the same seeding requirement; impersonation adds `users.impersonate` (action not in list) via PermissionSeeder (roles design D4 says migrations). Archive in order; impersonation edits same requirement.
- ARC-003 MEDIUM MEDIUM — Limit::none() removes per-IP throttle (dup).
- ARC-004 LOW MEDIUM — Settings cache forever vs insertOrIgnore; consider no cache.
- ARC-005 LOW MEDIUM — name_/description_ columns vs translation files.
- ARC-006 LOW MEDIUM — New base folders src/Mail, src/Exceptions, src/Listeners need approval (AGENTS.md).
- ARC-007 LOW MEDIUM — settings.* permission module reads as account settings; use configuration.
- ARC-008 INFO MEDIUM — BAS_ prefix vs unprefixed siblings (unverified).
