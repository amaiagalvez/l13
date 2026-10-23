# DevOps — add-session-log-and-settings (raw)

- OPS-001 HIGH HIGH — `login` limiter to `Limit::none()` (design.md:57-58) removes all per-IP throttling; config/fortify.php:117-118 references it. One IP can spray unlimited logins (credential stuffing, bcrypt + insert per attempt, mass lockout). Keep a per-IP limit alongside per-login lockout.
- OPS-002 HIGH HIGH — No queue worker in docker-compose.yml (only `composer dev` queue:listen, composer.json:58); `QUEUE_CONNECTION=database`, `MAIL_MAILER=log` (.env.example:43,55). Queued alert mail never sent. Send synchronously (max 1 per 5 min) or document worker.
- OPS-003 MEDIUM HIGH — Without Sentry, report() only writes logs (no SENTRY_* in .env.example; LOG_CHANNEL=daily, 14 days). State "log only"; avoid logins/IPs in context.
- OPS-004 MEDIUM HIGH/MEDIUM — config/cache.php:134 `serializable_classes => false`: caching Setting models/collections breaks; insertOrIgnore seeds don't fire `saved`. Cache scalar arrays; clear key in seed migrations.
- OPS-005 LOW MEDIUM — Cache::add/RateLimiter atomic on database store; expired cache rows not pruned.
- OPS-006 MEDIUM HIGH/MEDIUM — No trustProxies in bootstrap/app.php:17-41; behind a proxy `ip` column is the proxy IP.
- OPS-007 MEDIUM HIGH — No retention; `deleting` always throws, so no prune possible; attacker-driven growth.
- OPS-008 LOW MEDIUM — No operator unlock path; rollback order of seed migrations unspecified.
