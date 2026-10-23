# Performance — add-session-log-and-settings (raw)

- PERF-001 MEDIUM HIGH — Date filter `ListQueryBase::applyDateFilter` (packages/basics13/src/Queries/ListQueryBase.php:253-267) uses `whereDate()` → `DATE(created_at)` cannot use the planned `(created_at)` indexes on an unbounded table. Override with sargable range (`>= from 00:00`, `< to+1day`) in `SessionLogListQuery`.
- PERF-002 MEDIUM HIGH — Spike check counts the 5-minute window on every failure, even after the alert fired (design.md:62-64); during an attack the window is large. Check `Cache::has` first and use bounded existence (`skip(10)->exists()`).
- PERF-003 LOW MEDIUM — Plan does not specify eager loading of `user` (withTrashed) for the list; up to 15 lazy queries per page and null names for trashed users.
- PERF-004 LOW LOW — User filter options must use the existing capped remote select (ListQueryBase.php:35-37), not pluck all users.

Checked, no finding: insert per attempt and extra user lookup vs bcrypt; RateLimiter on database cache; Setting::get; proposed indexes sufficient once PERF-001 is fixed; paginate COUNT acceptable at current scale.
