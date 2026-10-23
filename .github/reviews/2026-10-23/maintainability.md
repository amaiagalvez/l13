# Maintainability — add-session-log-and-settings (raw)

- MNT-001 MEDIUM HIGH — tasks.md:20,24 put Login/Failed/Logout listeners and lockout check into FortifyServiceProvider (254 lines, already has a Login closure at 154-176). Use one concrete class in src/Auth; provider only wires.
- MNT-002 MEDIUM HIGH — Login key normalized in 3 places (design.md:52-53); "mismo criterio que hoy" is wrong: current key includes IP (FortifyServiceProvider.php:232). Single lockoutKey().
- MNT-003 LOW HIGH — 5 min / 300 s / 10 / default 3 duplicated. Class constants.
- MNT-004 LOW MEDIUM — Naming: Setting/settings.* vs /configuration vs routes/settings.php (account); SessionLog on BAS_sessions. Partly preference.
- MNT-005 MEDIUM HIGH — Immutability via model events bypassed by query-builder update (design.md:37-38); natural Logout implementation would bypass. Document + test.
- MNT-006 MEDIUM MEDIUM — design.md:45-46 relies on listener order; inactive eject throws ValidationException after Login, does not dispatch Failed → no row (inferred).
- MNT-007 LOW MEDIUM — Cache forever + insertOrIgnore seeds don't forget cache; `class` column free text, informational.
