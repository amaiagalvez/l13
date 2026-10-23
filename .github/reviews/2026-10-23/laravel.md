# Laravel — add-session-log-and-settings (raw)

- LAR-001 HIGH HIGH — RateLimiter `:timer` set via cache->add on first hit only (vendor/laravel/framework/src/Illuminate/Cache/RateLimiter.php:165-167; tooManyAttempts 130-135): lockout lasts the remainder of the first-failure window, not lockout_minutes. On reaching max: clear() + increment(key, lockout*60, max). Test with spaced failures via travel().
- LAR-002 HIGH HIGH — projects/demo13/config/cache.php:134 `serializable_classes => false`, default store database (:18): caching Setting models/collections returns incomplete objects. Cache scalar arrays only; array store in tests hides it (:39).
- LAR-003 MEDIUM MEDIUM — Limit::none(): Fortify skips its own throttle when fortify.limiters.login set (AuthenticatedSessionController.php:86; config/fortify.php:118) → no per-IP limit. Keep coarse per-IP limiter.
- LAR-004 MEDIUM MEDIUM — Login fires on remember-cookie restore (SessionGuard.php:196-202): spurious rows, overwrites session_key, clear() resets counter. Skip when viaRemember().
- LAR-005 LOW HIGH — design.md:45-46 wrong: inactive eject throws ValidationException, never Failed; password path already filtered by authenticateUsing (FortifyServiceProvider.php:136-139, 154-176). Listener order matters.
- LAR-006 LOW MEDIUM — insertOrIgnore seeding doesn't fire saved; forget cache in seed migrations; @update saves per model.

Verified OK: session_key survives regeneration; Logout fires before invalidate (Logout.php:17-19; Fortify destroy); Failed user null with authenticateUsing (AttemptToAuthenticate.php:114), credentials include password — read only username; lockout check in authenticateUsing also covers 2FA pre-check; Cache::add atomic on database store; report() independent of queue.
