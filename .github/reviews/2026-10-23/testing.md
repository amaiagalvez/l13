# Testing — add-session-log-and-settings (raw)

- TST-001 HIGH HIGH — packages/basics13/tests/Feature/Auth/AuthenticationTest.php:43-60 expects 429 after 5 failures; Limit::none() breaks it; no task replaces it.
- TST-002 HIGH HIGH — ArchitectureTest (tests/Unit/ArchitectureTest.php:14-16, ChecksResourceArchitecture.php:99-155) requires FormRequest on index/store/update; plan lacks requests for SessionLogController@index and SettingController@index. No `{!! !!}` in mail view (ArchitectureTest.php:38-60).
- TST-003 MEDIUM HIGH — Alert tests: with one login locked at 3 failures, 11 failures never counted; window not tested with travel(); race test via pre-set cache key; queued mail → assertQueued.
- TST-004 MEDIUM HIGH — Missing cases in 3.3: unknown login row, user_id filled on wrong password, logout leaves other rows, logout without session_key (actingAs, AuthenticationTest.php:86), real POST login.store flow, remember-me duplicate row.
- TST-005 MEDIUM HIGH — Read-only settings scenario (200 + 403 on PUT) untested; cache test must warm first; max=5 via HTTP update.
- TST-006 MEDIUM HIGH — Dusk 7.2 unjustified (no JS; tests.instructions.md:10); single tab.
- TST-007 LOW MEDIUM — Unit placement: Setting casts without get(); policies with PermissionsActor (RolePolicyTest.php:83-86); transformer with setRelation; UpdateSettingsRequest Feature-only.
- TST-008 LOW HIGH — Newest-first order and absence of write routes untested.
- TST-009 INFO MEDIUM — Host sidebar test OK; 80% is a gate.
