# Frontend — add-session-log-and-settings (raw)

Flux free: no flux:tab / date-picker stubs; navbar exists.

- FE-001 MEDIUM HIGH — Plan uses flux:navbar for package tabs; basics13 already has `x-list.tabs` (components/list/tabs.blade.php:9-23). Reuse it, translated package labels, define unknown ?package= behavior.
- FE-002 MEDIUM HIGH — Settings full-page form: tracked-resource form is drawer-only (forms/tracked-resource.blade.php:11,30); unsaved-changes warning needs `form[data-track-changes]` (resources/js/app.js:251-329). Specify it.
- FE-003 MEDIUM HIGH — Per-type inputs unspecified: boolean needs hidden 0; integer min=1; email nullable; read-only hides submit.
- FE-004 MEDIUM HIGH — name_{locale}/description_{locale} without fallback → blank labels.
- FE-005 MEDIUM HIGH — sidebar.blade.php:59 Administration group renders only if $seesUsers || $seesRoles; users with only sessions.view/settings.view never see entries.
- FE-006 MEDIUM HIGH — Translation clashes: es.json "Settings": "Configuración" (:241, account menu), "Login"/"Logout" are verbs (:140,:143). Use new specific keys.
- FE-007 LOW MEDIUM — Date filter semantics (between needs both dates; after/before exclusive; server timezone); user select cannot reach unknown logins. Consider filter on `login` text.
- FE-008 LOW HIGH — page-header renders empty tabs nav; no actions column; empty-state colspan.
