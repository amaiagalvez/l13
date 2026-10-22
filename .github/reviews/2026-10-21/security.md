# Security — raw specialist report (2026-10-21)

### SEC-001: Livewire public state is invoked as a PHP callable (remote code execution)
- Severity: CRITICAL · Confidence: HIGH [verified by orchestrator]
- File: packages/basics13/src/Livewire/AdvancedSearchModal.php:22, 27, 96-97, 198-199 (rendered from packages/basics13/resources/views/components/list/advanced-search-modal.blade.php:37, 48)
- Problem: `$fields` and `$scope` are public and not `#[Locked]`. `getVisibleFields()` runs `$field['condition']($this->scope)`, and `getSelectOptions()` runs `$field['options']($this->scope)`. After `mount()`, those values are whatever the client sends.
- Verification: in a throwaway container, `Livewire::test(AdvancedSearchModal::class, ...)->set('fields.name.condition', 'print_r')->set('scope', 'LW-INJECTION-MARKER')->set('show', true)` rendered HTML containing `LW-INJECTION-MARKER`. The server called the client-chosen function with the client-chosen argument. Using `system` instead of `print_r` gives shell execution. The code is committed and pushed in basics13 (`develop` 500fbeb, origin/develop). `master` does not contain it.
- Impact: any authenticated, verified user can execute arbitrary commands, read `.env` (APP_KEY, DB credentials) and take over the database. Combined with open registration (SEC-002), that means anyone.
- Recommendation: never call values taken from component state. Remove the runtime `condition`/`options` calls (`normalizeField()` already resolves them in `mount()`), and mark `$fields` and `$scope` `#[Locked]`. Add a regression test that sets `fields.*.condition` and asserts nothing runs and/or a `CannotUpdateLockedPropertyException`.

### SEC-002: Open self-registration plus "any user can update any account" (including the password) allows account takeover
- Severity: HIGH · Confidence: HIGH [verified]
- File: packages/basics13/config/basics13.php:7 (`'registration' => true`, with no host override in projects/demo13/config); packages/basics13/src/Policies/UserPolicy.php:37-40 (`update` returns true); packages/basics13/src/Http/Requests/UserRequest.php:42-48, 66-74 (an optional `password` on edit, with no current-password check); packages/basics13/routes/users.php:33
- Exploit:
  1. Register and verify an email address.
  2. Send `PUT /users/{id}` with a new password.
  3. Log in as the victim. Only the victim's own 2FA would stop this.
- Note: "every user can do everything" is documented. Public registration combined with password-reset-by-peer is not.
- Recommendation: disable registration in the host (or by default), and/or stop letting one user set another user's password or email.

### SEC-003: Log-viewer access is granted by email, and email can be changed without re-verification
- Severity: MEDIUM · Confidence: MEDIUM
- File: packages/basics13/src/ServiceProvider.php:109-116; packages/basics13/src/Http/Controllers/UserController.php:62-66
- Recommendation: gate on a user id or flag, or reset `email_verified_at` when the email changes.

### SEC-004: LIKE escape is missing in the advanced text filter
- Severity: LOW
- Duplicate of LAR-002.

### SEC-005: Array filter values cause a 500
- Severity: LOW
- Duplicate of API-001/LAR-004. Also packages/basics13/resources/views/components/list/page-header.blade.php:67 (illegal offset).

### SEC-006 (INFO): Docker development hardening
- File: projects/demo13/docker-compose.yml:10-11, 22, 242, 244
- Problems: 0.0.0.0 ports, Xdebug trigger, hardcoded root password, `sql-mode=""`.

Checked and fine:
- No SQL injection in filters: columns come from the server.
- `active` is not mass assignable.
- Comment author is taken from auth.
- Restore and force delete are authorized.
- No IDOR in nested routes.
- No unescaped Blade output in the reviewed views, and `innerHTML` only receives same-origin fragments.
- Login, 2FA and passkey rate limits are in place, and inactive users are blocked.
