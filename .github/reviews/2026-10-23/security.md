# Security — add-session-log-and-settings (raw)

- SEC-001 MEDIUM HIGH — Per-login lockout from any IP lets anyone lock any account (incl. admins) indefinitely; no unlock, password reset does not clear.
- SEC-002 MEDIUM HIGH — Lock counted from first failure (RateLimiter.php:165-167, 130-132) (dup LAR-001).
- SEC-003 MEDIUM HIGH — Limit::none() removes all request-rate limit on POST /login (FortifyServiceProvider.php:229-235; AuthenticatedSessionController.php:86; config/fortify.php:118): spraying + DB amplification.
- SEC-004 MEDIUM MEDIUM — Raw login stored/shown; users type passwords in login field; no max length (LoginRequest.php:28). Mask unknown_user logins, cap length.
- SEC-005 MEDIUM HIGH — Alert mail with logins/IPs; alert_email editable by settings.edit holder without sessions.view → exfiltration; queued payload in jobs table; no audit of settings changes. Counts + link only; TracksAuditColumns.
- SEC-006 LOW HIGH — No bounds: lockout_minutes 0 or huge; bounded rules + clamp on read.
- SEC-007 LOW HIGH — Immutability overstated (query builder, nullOnDelete); inactive expulsion not logged via Failed (FortifyServiceProvider.php:154-176).
- SEC-008 LOW MEDIUM — Pre-existing timing oracle in authenticateUsing (136-145) for unknown/inactive; wrap in Timebox.
- INFO: 2FA pre-check covered; passkeys bypass lockout (own 10/min limiter) — state in spec; no trustProxies → ip() is direct peer (safe now); user_agent escaping test.
