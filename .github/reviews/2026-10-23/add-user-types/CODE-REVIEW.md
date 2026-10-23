# Code Review — OpenSpec change `add-user-types` (planning)

## Executive Summary
Review of the plan only (`openspec/changes/add-user-types/`); no code exists yet. The plan is sound in structure (mirrors `EntityType`, stays inside `basics13`), but the delegated permission `users.manage_own` as designed allows **privilege escalation** (role assignment without a cap) and its scope is under-specified (peer admins, archived types, entry points other than the active list). Fix the plan before applying.

## Detected Stack
Laravel 13 · PHP 8.5 · Livewire 4 · Flux · Tailwind 4 · PHPUnit · Dusk · MySQL/MariaDB · Docker. Affected package: `packages/basics13`.

## Checks Executed
- `openspec validate add-user-types`: PASS.
- PHPUnit / PHPStan / Pint / Dusk: NOT RUN (no code changed by the plan).

## Critical Findings

### SEC-001 — Delegated role assignment escapes the scope
Severity: CRITICAL · Category: Security · File: openspec/changes/add-user-types/design.md (Roles, Risks) · Confidence: HIGH

Problem: design lets `assignsRoles()` accept `users.manage_own`, relying on `assignableRoleRule()` as mitigation.
Evidence: `packages/basics13/src/Http/Requests/UserRequest.php:130-147` only excludes trashed, inactive and system roles; today the gate is `roles.manage` (`:116`).
Impact: a type admin grants a role carrying `users.manage`/`roles.manage` to a user they manage (or via a peer) and obtains global reach.
Recommendation: with only `users.manage_own`, assignable roles MUST have permissions ⊆ the actor's permissions; keep `! editsOwnAccount()`; exclude the actor from its own scope. Add a spec scenario and escalation tests (TEST-002).

## High Findings

### SEC-002 — Scope reaches peer admins and global managers
Severity: HIGH · Category: Security · File: specs/basics-auth/spec.md (delegation requirement) · Confidence: HIGH
Evidence: `UserPolicy::reachable()` (`src/Policies/UserPolicy.php:91-95`) only shields the super-admin.
Impact: a type admin can archive, trash, force-delete or strip roles of other admins of the same type or of global managers.
Recommendation: exclude from "my users" anyone who is `is_admin` in a shared type or holds `users.manage`/`users.edit`, and the actor itself.

### BUG-001 — Contradictory rule for assigning types
Severity: HIGH · Category: Business Logic · File: spec.md (first requirement) vs design.md (Alta delegada) · Confidence: HIGH
Problem: spec says only `users.edit` assigns types; design lets `manage_own` choose them on create; type edits by `manage_own` are undefined.
Impact: removing the last shared type hides users from review; adding foreign types moves users into other scopes.
Recommendation: one rule in the spec: `manage_own` adds/removes only types it administers, never touches `is_admin` nor other types; at least one administered type must remain.

### SEC-003 — Scope not applied to every entry point
Severity: HIGH · Category: Security · File: tasks.md 3.2 · Confidence: HIGH
Evidence: `UserController` index uses `UserListQuery::active()` (`:33`); `selectOptions` (`:50-57`), trash/archived controllers and `PermissionSummaryController` are not mentioned. Trashed users need the scope with `withTrashed`.
Impact: out-of-scope names/emails leak; restore/force-delete may reach them.
Recommendation: apply the scope to active, archived and trashed lists, `users/options`, permissions summary and its export.

### BUG-002 — Archived or trashed types still grant scope
Severity: HIGH · Category: Business Logic · File: design.md (Alcance) · Confidence: HIGH
Recommendation: only active, non-deleted types count for scope; selectors offer only active types (keeping already assigned ones). Add one scenario.

### DB-001 — Pivot `user_id` FK behaviour unspecified
Severity: HIGH · Category: Database · File: design.md (Decisions) · Confidence: HIGH
Impact: force-deleting a user with types fails (restrict) or leaves orphans that block the type forever (no FK).
Recommendation: `user_id` `cascadeOnDelete`, `user_type_id` `restrictOnDelete`; test force delete then type delete.

### TEST-001 — No route-by-route authorization matrix
Severity: HIGH · Category: Testing · File: tasks.md 4.2 · Confidence: HIGH
Recommendation: data-provider Feature test over every users route (index, store, update, destroy, archive/activate, trash restore/destroy, options, permissions summary) × {view_own in/out scope, manage_own in/out scope, global manager}. Include member-not-admin, no duplicate rows with two shared types, type filter combined with scope.

## Medium Findings

### ARCH-001 — `userTypes()` relation owner unclear with host-swappable user model
Severity: MEDIUM · File: tasks.md 1.3 · Evidence: `UserPolicy` docblock and `ResolvesUserModel`; relation must survive `auth.providers.users.model`.
Recommendation: declare it in `src/Concerns/HasRoles.php`; `UserType::users()` against the configured model.

### ARCH-002 — Missing tasks
Severity: MEDIUM · File: tasks.md
Recommendation: add nav entry with its permission gate, i18n labels for the new permissions/module, `_own` in the permissions summary, demo13 seeder for a type admin.

### DB-002 — No primary key on the pivot
Severity: MEDIUM · File: design.md · Recommendation: `primary(['user_id','user_type_id'])` to prevent duplicates with conflicting `is_admin`.

### DB-003 — Delegated create not atomic
Severity: MEDIUM · File: design.md (Alta delegada) · Recommendation: user insert, type attach and roles in one `DB::transaction`, or a typeless user falls out of every scope.

### BUG-003 — Users without type
Severity: MEDIUM · File: spec.md · Recommendation: state they are reachable only with global `users.*`.

### TEST-002 — Escalation tests (pairs with SEC-001)
Severity: MEDIUM (blocking for SEC-001) · Recommendation: manage_own cannot assign roles beyond own permissions, set `is_admin`, assign foreign types, edit itself, or touch super-admin/peer admins.

### TEST-003 — Delete guard and unique-name tests
Severity: MEDIUM · Recommendation: type joined to active and to trashed user (delete and force delete); duplicate-insert race returns validation error, archived/trashed duplicates allowed.

## Low Findings
- **SEC-004** — State that `manage_own` never grants impersonation (`UserPolicy.php:100-105`).
- **SEC-005** — `users.manage` implies `_own` (`HasRoles.php:54`); policy must check global permission first; test global manager is not scope-limited.
- **DB-004** — With DB-002 PK, the index can be `(user_type_id, user_id)`; follow the existing `insertOrIgnore` pattern for the permission migration.

## Informational Findings
- Placing user types in `basics13` with `BAS_` prefix respects package boundaries.
- Spec delta is ~31 lines; room for the added scenarios within the ~60-line budget.

## Security
SEC-001 (critical), SEC-002, SEC-003, SEC-004, SEC-005.
## Bugs / Correctness
BUG-001, BUG-002, BUG-003.
## Database
DB-001, DB-002, DB-003, DB-004.
## Performance
No findings; the scope `EXISTS` is served by the planned index.
## Architecture
ARCH-001, ARCH-002.
## Testing
TEST-001, TEST-002, TEST-003.
## Production
Not applicable (no deploy/config change).
## Maintainability
No findings.

## Rejected Findings
- Guard in `booted()` vs service provider: rejected; `EntityType::booted()` (`packages/entities13/src/Models/EntityType.php:95-106`) is the existing precedent.
- Scope placement: rejected; design already places it in query + policy membership check.
- Naming base traits/migration helpers in tasks: rejected; mirroring `EntityType` covers it.
- Permission migration test, i18n parity, Unit/Feature split, Dusk decision: rejected (low value or process).
- `HasRoles` not found: rejected; it exists at `packages/basics13/src/Concerns/HasRoles.php`.

## Action Plan
1. **SEC-001 + TEST-002**: cap delegated roles to actor's permissions; exclude actor from scope.
2. **SEC-002**: exclude peer admins and global managers from scope.
3. **BUG-001**: single rule for who assigns types.
4. **SEC-003**: scope every entry point (archived, trash, options, summary/export).
5. **BUG-002**: only active types grant scope.
6. **DB-001**: pivot FKs (cascade user, restrict type).
7. **TEST-001**: authorization matrix.
8. ARCH-001, ARCH-002, DB-002, DB-003, BUG-003, TEST-003.
9. SEC-004, SEC-005, DB-004.
