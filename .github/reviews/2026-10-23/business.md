# Business logic — add-session-log-and-settings (raw)

- BUS-001 HIGH HIGH — RateLimiter::hit decay fixed at first hit (design.md:52-57): failures spread over 14 min → lockout ~1 min while row records 15. Separate lockout key with its own decay; state the counting window in spec.
- BUS-002 MEDIUM HIGH — Request says block when failing "more than" the value; plan blocks on reaching it (3rd). Spike uses "more than 10". Confirm and record.
- BUS-003 HIGH MEDIUM-HIGH — Rejections during lockout are written from authenticateUsing (no Failed event) → spike check never runs for them; event value for locked rejections and whether they count/extend undefined.
- BUS-004 MEDIUM MEDIUM — Limit::none() removes per-IP throttle; password spraying unthrottled (dup of OPS-001).
- BUS-005 MEDIUM HIGH — Super-admin can be locked repeatedly; no recovery/unlock path.
- BUS-006 MEDIUM MEDIUM — Login fires on remember-me, passkeys, registration and planned impersonation (overwrites session_key, clears target counter); inactive eject path does not fire Failed → no row.
- BUS-007 MEDIUM HIGH — Spec "por ninguna vía" contradicted by nullOnDelete FK on force delete and query-builder bypass.
- BUS-008 MEDIUM HIGH — type=integer cannot express min:1; lockout_minutes=0 disables lockout; alert_email empty must be valid.
- BUS-009 LOW HIGH — "cada ventana de 5 minutos" vs cooldown from first alert.
- BUS-010 LOW MEDIUM — Lowering max below live counter; concurrent == max; use >= and write lockout once.
- BUS-011 LOW MEDIUM — Logout other devices / password change leave rows open; list as non-goals.
- BUS-012 LOW MEDIUM — User filter drops unknown_user rows; need login text filter; inclusive "hasta"/timezone.
- BUS-013 LOW MEDIUM — Delta on basics-roles-permissions clashes with add-superadmin-user-impersonation (`impersonate` action) on same requirement; sequence.
- BUS-014 INFO HIGH — basics-login-audit spec 70 lines > ~60 budget; missing scenarios for unknown-login lockout, any IP, locked rejection row.
- Coverage: complete; record "desde dónde" = IP + user agent.
