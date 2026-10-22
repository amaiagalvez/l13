# Code Review

Date: 2026-10-22 · Mode: READ-ONLY spec review (`/full-review la spec add-roles-permissions-system`) · Orchestrator: `.github/agents/review-orchestrator.agent.md`

Target: the OpenSpec change `openspec/changes/add-roles-permissions-system/`. That means `proposal.md`, `design.md`, `tasks.md`, the six delta specs and the nested `specs/basics-roles-permissions/design.md`. Nothing in this change is implemented yet: `packages/basics13` has no `Role`, `Permission`, `HasRoles` or `Language`. So every finding is about what the spec would produce if it were applied as written, checked against the real code in `packages/basics13` and the Laravel framework in `vendor/`.

> **Process note.** All 10 specialist subagents (Laravel, Security, Database, Performance, Business Logic, Architecture, Testing, Frontend, Maintainability, DevOps) stopped with an API session limit (HTTP 429) before they returned anything. The orchestrator therefore ran each specialist pass itself, in the same risk order, followed by the devil's-advocate pass. API was skipped because the change exposes no API. Since no subagent produced a report, there are no raw specialist files, only this one. An earlier review of a previous version of this change is in git: `git show 44883e0:.github/reviews/2026-10-21/CODE-REVIEW.md`.

## Executive Summary

The change is **not ready to implement**. `openspec validate --strict` passes, but that only checks structure. The content has three kinds of problem.

**1. As written, it would give away total access or break login for the super-admin.**
- **CRITICAL, SEC-001:** the super-admin is "whichever user has `id = 1`", and self-registration is on by default. On a fresh install where the seeder has not run, the first person to register gets every permission. On an existing install, user 1 is already a real person, who would silently become a hidden, immutable super-admin.
- **HIGH, SEC-002:** hiding user 1 with a global scope also hides it from Laravel's `EloquentUserProvider`, so the super-admin could never log in.
- **HIGH, BUG-001:** the planned "throw on update" guard for user 1 breaks remember-me, 2FA and password changes for that account.
- **HIGH, SEC-003:** any user allowed to edit users can give themselves the `superadmin` role.

**2. The documents contradict each other, so the schema cannot be built as specified.**
- **DB-001:** three incompatible definitions of `role_user`.
- **DB-002:** uniqueness is specified as a partial index, which MariaDB 11.7 does not support, and it also contradicts the project's existing invariant.
- **DB-003:** a CHECK constraint that rejects the wildcard permission the same change seeds.
- **DB-004:** the audit trait the models use needs columns the tables don't have.
- The role/permission matrix differs between documents (BL-002).

**3. Scope and package boundaries.**
- **ARCH-001:** `basics13` would name domain modules, including `clients`, which duplicates `entities` ("cliente" is the Spanish UI name of an entity).
- **ARCH-002:** this one change also carries a language feature, project membership, entity-type permissions and a bridge selector.
- **BL-001:** no migration path is defined, so existing users end up locked out once policies start checking permissions.

**Recommendation:** shrink the change to RBAC in `basics13` only (models, assignment, permission checks, role CRUD, seeded read-only permissions). Keep one design document. Reuse the existing helpers (`addCommonColumns`, `addAuditColumns`, `addUniqueActiveColumnIndex`). Replace "ID = 1" with an explicit, non-registrable bootstrap. Move language, project members and the bridge selector into their own changes.

## Detected Stack

Verified in the repository:
- Laravel 13 and PHP 8.5.
- Livewire 4 with Flux (free) and Tailwind 4.
- PHPUnit and Dusk for tests, Pint for formatting.
- MariaDB 11.7 (`projects/demo13/docker-compose.yml:241`).
- Fortify authentication with self-registration on by default (`packages/basics13/config/basics13.php`).
- Database queue driver, but no application jobs.
- No Redis, Horizon, Pest, Vue or Inertia.
- PHPStan with a baseline in each package.

Facts the spec relies on:
- `UserPolicy` returns `true` for every ability.
- `addAuditColumns()` already hard-FKs to `users`.
- Uniqueness is enforced with a stored generated column (`addUniqueActiveColumnIndex`), unique among non-deleted rows including archived ones.
- `config/basics13.php` already exists.
- "Cliente" is the Spanish UI name of an entity (`openspec/specs/entities-domain`); there is no separate client model or module.
- Projects have no owner/member concept.
- A translation parity test base exists (`packages/basics13/tests/TranslationFilesTestCase.php`).

## Checks Executed

| Check | Result |
|---|---|
| `openspec validate add-roles-permissions-system --strict` | PASS ("Change 'add-roles-permissions-system' is valid"). It validates structure only. |
| Framework behaviour check: `vendor/laravel/framework/src/Illuminate/Auth/EloquentUserProvider.php:188-197` | `newModelQuery()` uses `$model->newQuery()`, so global scopes apply to authentication (evidence for SEC-002). |
| Framework behaviour check: `EloquentUserProvider.php:100` | `updateRememberToken()` calls `$user->save()` (evidence for BUG-001). |
| Framework behaviour check: `Support/Onceable.php:47-50` | `once()` is keyed by the calling object instance (input to the rejected PERF claim). |
| PHPUnit, Pint, PHPStan, `composer audit`, npm build, Dusk | NOT RUN. The target is an unimplemented spec, so no changed code exists for them to check. |

## Critical Findings

### SEC-001 — Super-admin identified by `id = 1` gives total access to the first self-registered user
Severity: CRITICAL
Category: Security
File: openspec/changes/add-roles-permissions-system/specs/basics-super-admin/spec.md
Line: 9
Confidence: HIGH

Problem:
Full, implicit access is tied to the numeric primary key 1, not to an explicit flag set by a trusted process.

Evidence:
- `basics-super-admin/spec.md:9`: "El sistema MUST incluir un usuario con `id = 1` que tiene todos los permisos sin roles asignados".
- `tasks.md:68-76` and `specs/basics-roles-permissions/design.md:99`: "Super-admin bypass: if `$this->id === 1` return true".
- `packages/basics13/config/basics13.php`: `'registration' => true`.
- The super-admin only exists if someone runs `SuperAdminUserSeeder` (`tasks.md:174-179`, "Runs first in seeder order").
- `design.md:271` and `specs/basics-roles-permissions/design.md:433`: on existing installs, "Host can run seeders manually".

Impact:
- **Fresh install where the seeder has not run:** the first person to register gets auto-increment id 1. They then hold every permission, are hidden from every list, and cannot be removed.
- **Existing install:** user 1 is already a real account, which silently gains total access. The seeder's explicit `id = 1` insert either collides with it or overwrites it.

Recommendation:
Do not derive privilege from the id. The simplest replacement is one of these:
- a non-fillable boolean `is_super_admin` set only by the seeder or an artisan command, or
- the existing `superadmin` system role, assigned by the seeder.

Either way, the bootstrap must refuse to run when the account already exists. If the "hidden and immutable" behaviour is still wanted, key it on that flag.

## High Findings

### SEC-002 — Global scope `id != 1` prevents the super-admin from logging in
Severity: HIGH
Category: Security / Correctness
File: openspec/changes/add-roles-permissions-system/tasks.md
Line: 60
Confidence: HIGH

Problem:
The spec hides user 1 with a global scope on the User model and assumes it does not affect authentication.

Evidence:
- `tasks.md:60`: "Add Global Scope: `where('id', '!=', 1)` to hide from all queries (except auth)". The same approach is in `specs/basics-roles-permissions/design.md:122`.
- `vendor/laravel/framework/src/Illuminate/Auth/EloquentUserProvider.php:190-192` builds its queries with `$this->createModel()->newQuery()`, and that applies global scopes.
- The spec does not describe any mechanism for the "(except auth)" exception.

Impact:
- `retrieveById`, `retrieveByCredentials` and `retrieveByToken` never find user 1, so the only privileged account cannot sign in.
- The same scope also hides the account from Fortify's password reset, email verification and passkey lookups.

Recommendation:
Drop the global scope. Exclude the account explicitly in the list queries that must hide it (`UserListQuery` and the permissions summary). That is local and visible, and it leaves authentication untouched.

### BUG-001 — "User 1 is immutable, even by itself" breaks remember-me, 2FA, password and profile for that account
Severity: HIGH
Category: Bug / Business Logic
File: openspec/changes/add-roles-permissions-system/specs/basics-super-admin/spec.md
Line: 10-11
Confidence: HIGH

Problem:
The account's own security settings must be able to save the user row, but the spec forbids any update.

Evidence:
- `basics-super-admin/spec.md:10-11`: "MUST NOT modificarse, archivarse, desactivarse ni eliminarse (tampoco por sí mismo)".
- `tasks.md:64`: "Model events: prevent update/delete/... if `id === 1`".
- `EloquentUserProvider.php:100`: `updateRememberToken()` calls `$user->save()`.
- `basics-auth` (`openspec/specs/basics-auth/spec.md:7-10`) requires `basics13` to provide the account settings flows: profile, password, 2FA and passkeys.

Impact:
- For the super-admin, logging in with "remember me" and logging out (which cycles the token) both throw.
- Enabling 2FA, changing the password or rotating a compromised password are all impossible. The most privileged account would also be the one with the weakest credential hygiene.

Recommendation:
Narrow the rule. Other users must not edit, archive, trash or delete the super-admin. Enforce that in `UserPolicy`, not in a model `updating` event. The account holder keeps the normal self-service flows (`UserRequest` already restricts email and password changes to the account holder).

### SEC-003 — Any user with `users.edit` can assign themselves the `superadmin` role
Severity: HIGH
Category: Security
File: openspec/changes/add-roles-permissions-system/specs/basics-roles-admin/spec.md
Line: 34-37
Confidence: HIGH

Problem:
The spec says who may edit system roles, but not who may assign them. It also never forbids a user from changing their own roles.

Evidence:
- `basics-super-admin/spec.md:32-35` protects only "crear/editar/borrar rol con `is_system = true`".
- The assignment requirement (`basics-roles-admin/spec.md:34-37`) has no authorization rule.
- `basics-auth/spec.md:24-25` (delta) adds a role selector to the user form, and that form is authorized by `UserPolicy::update`.
- The `superadmin` role is seeded with `*.manage`. `admin` holds every `users.*` permission (`specs/basics-roles-permissions/design.md:272`).

Impact:
An `admin`, or any custom role holding `users.edit`, opens their own account and selects `superadmin`, which makes them a full superadmin. The same path lets them grant themselves any role that contains `roles.manage`.

Recommendation:
Add one requirement, with tests:
- Assigning or removing a role requires `roles.manage`.
- System roles, and any role holding `*.manage`, can only be assigned by the super-admin.
- Nobody may change their own roles.

### DB-001 — Three incompatible definitions of `role_user`; the required `assigned_by` audit column is missing from the one the specs mandate
Severity: HIGH
Category: Database
File: openspec/changes/add-roles-permissions-system/specs/basics-permission-checks/spec.md
Line: 63-71
Confidence: HIGH

Problem:
The documents disagree on the pivot table, and the version the specs require cannot satisfy the other specs.

Evidence:
- `design.md:41-50`: `user_id` with an FK to `users`, plus `assigned_at` and `assigned_by`. The relation is `belongsToMany(...)->withTimestamps()` (`design.md:89-94`).
- `basics-permission-checks/spec.md:63-71`, `tasks.md:18-22` and the nested design (`:52-58`): polymorphic `model_type`/`model_id` with "No FK on model_id". The columns are only `role_id, model_type, model_id, assigned_at`, with no `assigned_by`.
- `basics-roles-admin/spec.md:37,41` requires `assigned_by = auth()->id()`.
- Both relation snippets call `withTimestamps()`, but no version of the table has `created_at`/`updated_at`.
- `Role::users()` is `BelongsToMany` (`specs/basics-roles-permissions/design.md:74`), while `HasRoles::roles()` is `MorphToMany` (`:94`). Those two relations cannot share the same pivot.

Impact:
Whichever version is implemented breaks another document's scenario. `withTimestamps()` on a table without timestamp columns fails on the first `attach`. With no FK, force-deleting a user leaves orphaned assignments behind.

Recommendation:
Keep the plain version: `belongsToMany` on `user_id` with `->constrained('users')->cascadeOnDelete()`, and `assigned_by` as a nullable FK. Polymorphism gains nothing here, because `basics13` already hard-references `users` in `addAuditColumns()` (`packages/basics13/src/Support/Database/helpers.php:35-37`). Drop `withTimestamps()` or add the columns. Delete the conflicting text from the other documents.

### DB-002 — Slug uniqueness is specified as a partial index MariaDB cannot create, scoped differently in each document, and against the project invariant
Severity: HIGH
Category: Database
File: openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/spec.md
Line: 9-10
Confidence: HIGH

Problem:
The uniqueness rule cannot be built on the project's database, and each document states a different rule.

Evidence:
- `design.md:20,36`: `UNIQUE KEY roles_slug_active_unique (slug, active) WHERE active = TRUE`. MariaDB 11.7 has no filtered or partial indexes. On top of that, `(slug, active)` would forbid two archived roles with the same slug.
- `tasks.md:9,14`: "name (unique), slug (unique)" and a partial index. Full uniqueness contradicts the reuse scenarios.
- `basics-roles-permissions/spec.md:9-10`: unique "solo entre roles activos no eliminados", so a slug can be reused while its owner is archived.
- The project's existing invariant (`.github/docs/architecture/ARCHITECTURE.md`, "Database"; `openspec/specs/basics-trash-archived`) is: unique among non-deleted records, *including archived*.

Impact:
- The migration cannot be written as specified.
- If uniqueness only covers active rows, `hasRole('x')` and `whereIn('slug', …)` match two rows (one archived, one active). Activating the archived one then either breaks the index or needs special-case conflict handling that is not specified.

Recommendation:
Reuse `addUniqueActiveColumnIndex('roles', 'slug')` and `addUniqueActiveColumnIndex('permissions', 'slug')`. That gives the same guarantee as every other resource. Drop the "reuse an archived slug" scenario and the extra `name (unique)` constraint, or make name unique the same way.

### DB-003 — CHECK constraint on `module` rejects the `*` wildcard that the same change seeds, and every domain module
Severity: HIGH
Category: Database / Bug
File: openspec/changes/add-roles-permissions-system/tasks.md
Line: 16
Confidence: HIGH

Problem:
The CHECK constraint and the seeded data contradict each other.

Evidence:
- `tasks.md:16` and `specs/basics-roles-permissions/design.md:47`: "CHECK constraint on module column: `IN ('users', 'roles', 'permissions')`".
- `tasks.md:191`: the same change seeds "`*.manage` (module `*`, action `manage`)".
- `design.md:235` and `tasks.md:324` say domain packages add modules "no CHECK constraint extension".

Impact:
`PermissionSeeder` fails on its own wildcard row, and domain packages cannot insert their permissions at all.

Recommendation:
Remove the CHECK constraint. `module` is free text validated by the seeders that own it (see ARCH-003).

### DB-004 — Role and Permission use `TracksAuditColumns`, but their tables lack the audit (and common) columns
Severity: HIGH
Category: Database / Bug
File: openspec/changes/add-roles-permissions-system/design.md
Line: 9-21
Confidence: HIGH

Problem:
The models use a trait that writes columns the specified tables do not have.

Evidence:
- The models use `SoftDeletes, TracksAuditColumns` (`design.md:81`, `:122`).
- `packages/basics13/src/Concerns/TracksAuditColumns.php:12-24` stamps `created_by`, `updated_by` and `deleted_by` on create, update and trash.
- The specified `roles` and `permissions` tables (`design.md:9-37`; `specs/basics-roles-permissions/design.md:17-44`) have none of those columns. They also lack `notes` and the `(deleted_at, id)` index from `addCommonColumns()`.

Impact:
The first insert fails with an unknown-column error, and the tables diverge from every other resource that uses the shared list, trash and archive code.

Recommendation:
Build both tables with `addCommonColumns($table)` and `addAuditColumns($table)`, as the domain tables do (for example `packages/entities13/database/migrations/2026_10_16_000000_create_ent_entity_types_table.php:20-28`).

### BL-001 — No migration path: existing users get no roles, so permission-based policies lock everyone out, yet the proposal states "no breaking changes"
Severity: HIGH
Category: Business Logic
File: openspec/changes/add-roles-permissions-system/proposal.md
Line: 64-65
Confidence: HIGH

Problem:
The proposal claims the change is additive, while the design turns off access for every existing user.

Evidence:
- `proposal.md:65`: "None - additive only. Existing installations work without roles/permissions until configured."
- `design.md:271`: "Existing users get no roles by default (opt-in)".
- The change adds `RolePolicy` and `PermissionPolicy` and changes `UserPolicy` ("`UserPolicy::delete()` rechaza", `basics-roles-admin/spec.md:60`).
- Phase 8 moves the domain policies to `hasPermissionTo()` (`tasks.md:256-257`).
- Today `UserPolicy` (`packages/basics13/src/Policies/UserPolicy.php`) and `ARCHITECTURE.md` ("every authenticated verified user") grant everything.

Impact:
- Once policies check permissions, every existing user without a role gets 403 everywhere, and only the super-admin can repair it.
- Users who self-register also get no role. The product's behaviour for them is unspecified.

Recommendation:
Specify the transition explicitly:
- a default role for existing users and for new registrations, assigned by the migration or seeder, **or**
- a rule that policies keep allowing everything until the host enables RBAC.

Mark the change as breaking for domain policies, and update the `ARCHITECTURE.md` authorization section when the change is archived.

### ARCH-001 — `basics13` would name domain modules (including `clients`, a duplicate of `entities`) and this change specs cross-domain behaviour that depends on a concept that does not exist
Severity: HIGH
Category: Architecture
File: openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/spec.md
Line: 49-50
Confidence: HIGH

Problem:
The domain-neutral base package would carry domain knowledge, and the bridge delta depends on project membership that nothing provides.

Evidence:
- The `basics13` specs list domain modules: `basics-roles-permissions/spec.md:49` ("`clients`, `projects`, `entities`, `entity_types`") and `basics-roles-admin/spec.md:69`.
- The `basics13` translation task (`tasks.md:165`) adds "Modules: clients, projects, entities".
- `clients` and `entities` name the same resource: "cliente" is the Spanish UI name of an entity (`openspec/specs/entities-domain/spec.md`), and `entities13` has no separate client model. Two modules for one resource means two permission sets that can disagree (`clients.edit` without `entities.edit`).
- `entity-projects-bridge/spec.md:11` filters entities to "proyectos donde el usuario es owner o member", but owner/member only appears in Phase 8a (`tasks.md:261-320`), a future change to `projects13`.
- `specs/basics-roles-permissions/design.md:425` adds a `SelectOptionsProvider` interface in `basics13` that has no current consumer.
- AGENTS.md: "`basics13` is domain-neutral shared infrastructure; never move cross-domain business logic there", and coupling "includes non-import forms (schema, payload keys, UI conditions, tests…)".

Impact:
- The package boundary is broken through names, translations and UI columns.
- The bridge delta cannot be implemented or archived on its own, because it describes behaviour of a feature that does not exist.

Recommendation:
- `basics13` speaks only of `users`, `roles` and `permissions`. The summary discovers other modules from data, as `tasks.md:145` already says.
- Remove `clients` everywhere and use only `entities` for that resource.
- Drop the `entity-projects-bridge` delta and `SelectOptionsProvider` from this change. Propose them together with project membership.

## Medium Findings

### ARCH-002 — Not an atomic change: language preferences, project members, entity-type permissions and route-prefix config ride along
Severity: MEDIUM
Category: Architecture
File: openspec/changes/add-roles-permissions-system/tasks.md
Line: 31-91, 254-334
Confidence: HIGH

Problem:
The change bundles several unrelated features.

Evidence:
- The `Language` enum, `users.lang`, the `SetUserLocale` middleware and a language column with flags are added to `basics-auth` (`tasks.md:31-38, 85-89, 152-156`; `basics-auth/spec.md:26, 42-49`).
- `config/basics13.php` with `route_prefix` (`tasks.md:91`): that file already exists, and the existing user routes are unprefixed `users/...` while the specs use `/admin/users`.
- Phases 8a and 8b (project members, entity types).
- `openspec/config.yaml`: "Un change = un cambio atómico = un PR". `proposal.md:56` even cites "PHP 8.4" in a PHP 8.5 project.
- `design.md:269` says "No changes to existing `users` table" while the tasks add `lang`.

Impact:
The change cannot be merged, reviewed or archived as one PR. Unrelated risk, such as locale middleware on every request, ships with the authorization change.

Recommendation:
Split into separate changes: `add-user-language` (enum, column, middleware), project membership, and the domain adoption of permissions. Drop `route_prefix` unless the existing user routes move too.

### ARCH-003 — Full CRUD for permissions conflicts with permissions being slugs hard-coded in policies
Severity: MEDIUM
Category: Architecture / Business Logic
File: openspec/changes/add-roles-permissions-system/specs/basics-roles-admin/spec.md
Line: 9-31
Confidence: MEDIUM

Problem:
Editing permissions through the UI does not make sense when policy code checks fixed slugs.

Evidence:
- The policies check literal slugs (`design.md:250-258`: `hasPermissionTo('entities.edit')`).
- The change still specifies create, edit, archive, trash and force-delete for `Permission` (`basics-roles-admin/spec.md:9-31`; `tasks.md:100-118`).

Impact:
- Renaming, archiving or deleting a permission silently revokes access that code still checks.
- Creating a permission has no effect, because nothing checks it.
- This roughly doubles the controllers, requests, policies and views of the change, for no benefit.

Recommendation:
Treat permissions as seeded, read-only data owned by the package that checks them: a list, without the form, trash or archive. Only roles are editable, together with their permission set.

### ARCH-004 — Host user models "MUST use `HasRoles`" is a breaking requirement for hosts, contrary to the proposal
Severity: MEDIUM
Category: Architecture
File: openspec/changes/add-roles-permissions-system/specs/basics-auth/spec.md
Line: 7-8, 17-18
Confidence: HIGH

Problem:
The spec places a new hard requirement on hosts' own user models, while the proposal says nothing breaks.

Evidence:
- `basics-auth/spec.md:17-18`: "el modelo configurado MUST usar la concern `HasRoles`".
- `proposal.md:65`: "Breaking Changes: None".
- `basics-auth` currently lets the host plug in any Fortify-compatible model (`openspec/specs/basics-auth/spec.md:20-32`).

Impact:
A host with its own user model breaks as soon as a policy calls `hasPermissionTo()`.

Recommendation:
Declare it as breaking and document the upgrade step. Alternatively, have the policies call a `Gate` ability defined in `basics13` that resolves permissions for any user model, which needs no trait in the host. Pick one approach and state it in a single place.

### BL-002 — Default role/permission matrix and action vocabulary differ between documents
Severity: MEDIUM
Category: Business Logic
File: openspec/changes/add-roles-permissions-system/specs/basics-super-admin/spec.md
Line: 44
Confidence: HIGH

Problem:
Each document gives different permission sets for the same default roles, and spells one action two ways.

Evidence:

| Source | `manager` | `viewer` |
|---|---|---|
| `design.md:230-237` | `clients/projects/entities.{view,create,edit}`, `users.view` | `clients/projects/entities/users.view` |
| `tasks.md:196-197` and nested design `:273-274` | `users.{view,create,edit}`, `roles.view`, `permissions.view` | `users.view`, `roles.view`, `permissions.view` |
| `basics-super-admin/spec.md:44` | "users.view" | "solo users.view" |

The action is written `force-delete` in `design.md:212` and `force_delete` in the specs and tasks.

Impact:
The seeders and tests cannot satisfy every document at once. A policy that checks `projects.force-delete` never matches a seeded `projects.force_delete`.

Recommendation:
Keep the matrix in one place (the spec scenario that the tests assert) and remove it from the others. Fix the action spelling to `force_delete`.

### BL-003 — Two overlapping super-admin concepts with contradictory rules
Severity: MEDIUM
Category: Business Logic
File: openspec/changes/add-roles-permissions-system/specs/basics-roles-admin/spec.md
Line: 58-61
Confidence: HIGH

Problem:
The spec describes a super-admin user and a super-admin role, and their rules conflict.

Evidence:
- A fixed super-admin user (id 1) "único que MAY gestionar los roles y permisos del sistema" (`basics-super-admin/spec.md:11-12`).
- Also a `superadmin` system role holding `roles.manage` and `*.manage` (`tasks.md:182, 194`).
- A "last superadmin" protection for users with that role (`basics-roles-admin/spec.md:58-61`).
- `hasRole('cualquier-rol')` returns true for id 1 (`basics-super-admin/spec.md:17`).

Impact:
- A `superadmin`-role holder has `roles.manage` but is forbidden from managing system roles, so the role means less than its name says.
- "Last superadmin" protection is pointless while user 1 exists.
- `hasRole()` returning true for every role makes negative checks (`! hasRole('viewer')`) wrong for user 1.

Recommendation:
Keep one concept. After SEC-001, the simplest is the `superadmin` system role plus a last-holder guard, under a lock. Drop the "virtual `hasRole`" rule.

### BUG-002 — Contradictory cache semantics, and reference code that implements none of the specified rules
Severity: MEDIUM
Category: Bug
File: openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/spec.md
Line: 70-75
Confidence: HIGH

Problem:
Two specs give opposite rules for when permission changes take effect, and the reference code follows neither of them.

Evidence:
- `basics-roles-permissions/spec.md:70-75`: role changes "MUST reflejarse inmediatamente … sin recargar sesión".
- `basics-permission-checks/spec.md:34-38`: "Cambios en permisos de roles NO se reflejan en el mismo request".
- The reference `HasRoles` in `design.md:145-205` queries on every call (no `once()`), ignores `active`, `deleted_at` and the `manage`/`*.manage` wildcards, and accepts `string|Role|array` but passes `Role` objects to `whereIn('slug', …)`.

Impact:
An implementer who follows `design.md` produces code that fails the specs: inactive roles grant access, and `projects.manage` does not imply `projects.edit`.

Recommendation:
- Keep "per request" as the rule (it is testable), and reword the other scenario to say "next request".
- Remove the reference code from `design.md`, or correct it to load the user's active permission slugs once and check exact, then `module.manage`, then `*.manage` in memory.

### SEC-004 — `active` and `is_system` are mass-assignable and editable from the form, against the project invariant
Severity: MEDIUM
Category: Security
File: openspec/changes/add-roles-permissions-system/design.md
Line: 75, 117
Confidence: HIGH

Problem:
Two protected flags would be writable through ordinary form input.

Evidence:
- `#[Fillable(['name', 'slug', 'description', 'is_system', 'active'])]` (`design.md:75`), and similarly for `Permission` (`:117`).
- The role form has a checkbox "activo" (`basics-roles-admin/spec.md:25`), and both forms have an "active toggle" (`tasks.md:136, 141`).
- `ARCHITECTURE.md`: "The `active` flag is never mass assignable … changes only through the dedicated archive/activate actions".

Impact:
- Any user with `roles.create` can create an `is_system` role that cannot be archived, trashed or deleted. Only the super-admin can clean it up.
- `active` can be flipped from create and update payloads, bypassing the archive/activate actions and their checks.

Recommendation:
Remove both flags from `Fillable` and from the forms. Only the seeder sets `is_system`. `active` changes only through `RoleArchivedController`, as for every other resource.

### BUG-003 — System-role protection via `updating` event and DB trigger blocks legitimate edits and does not stop soft deletes
Severity: MEDIUM
Category: Bug / DevOps
File: openspec/changes/add-roles-permissions-system/tasks.md
Line: 29
Confidence: HIGH

Problem:
The proposed guard is both too broad and in the wrong place.

Evidence:
- `tasks.md:29`: "`deleting`/`updating` event that throws if `is_system = true`; optionally DB trigger".
- `basics-roles-permissions/spec.md:37-40` requires DB-level failure of `DELETE`/`UPDATE … active = false`.
- Meanwhile `basics-super-admin/spec.md:11-12` and `:32-35` let the super-admin edit system roles.

Impact:
- The `updating` throw blocks every edit of a system role (name, description), including the super-admin's.
- `SoftDeletes` issues an `UPDATE deleted_at`, so a `DELETE` trigger never fires on a trash.
- On MariaDB with binary logging, creating triggers in migrations requires `SUPER` or `log_bin_trust_function_creators`. No other migration in the project uses triggers.

Recommendation:
Enforce the rule in `RolePolicy` (`archive`/`delete`/`forceDelete` return false for `is_system`), as the existing delete guards do. Drop the trigger and the model-event throw.

### OPS-001 — Super-admin bootstrap reads `env()` in a seeder with a public default email and no password guard
Severity: MEDIUM
Category: DevOps / Security
File: openspec/changes/add-roles-permissions-system/tasks.md
Line: 174-179
Confidence: HIGH

Problem:
The bootstrap reads environment values in a way that silently fails, and has no safe defaults.

Evidence:
`tasks.md:176-177` and `basics-super-admin/spec.md:40`:
- `env('SUPER_ADMIN_EMAIL', 'superadmin@example.com')`
- `env('SUPER_ADMIN_PASSWORD')` passed to `Hash::make()`

Impact:
- With `config:cache`, `env()` outside config files returns `null`. The seeder then creates the account with the default address and a hash of an empty or null password.
- If the variable is simply missing, the same happens without any error.

Recommendation:
Read the values through a `config()` key. Make the seeder or command fail when the password is empty or the email is the default. Better still, make it an interactive artisan command, which also solves SEC-001.

### TEST-001 — Test plan misses the security invariants that matter and plans DB tests as "Unit"
Severity: MEDIUM
Category: Testing
File: openspec/changes/add-roles-permissions-system/tasks.md
Line: 201-244
Confidence: HIGH

Problem:
The plan tests the easy paths and skips the risky ones.

Evidence:
- Phase 6 has no test that a non-super-admin cannot assign a system role or change their own roles (SEC-003).
- It has no test that the super-admin can log in, use remember-me and 2FA (SEC-002, BUG-001).
- It has no test of behaviour for users without roles or for new registrations (BL-001), and no 403 tests on the new routes.
- "Unit tests for `HasRoles` concern methods" (`tasks.md:203`) need roles in the database, so `.github/skills/test-rules/SKILL.md` places them under Feature.
- Some planned items cannot be tested in this stack, such as the DB trigger rejecting direct SQL (`basics-roles-permissions/spec.md:37-40`).
- Others test trivial code, such as `isRtl()` returning false (`tasks.md:232`).

Impact:
The planned suite would pass with SEC-001 to SEC-003 present.

Recommendation:
Add those cases as Feature tests in `basics13`, plus a host test in `demo13` for a custom user model. Drop the trigger and enum-trivia tests. Reuse the existing duplicate-insert race pattern and `TranslationFilesTestCase`.

### MAINT-001 — Two design documents and three copies of the same rules disagree; specs exceed the budget
Severity: MEDIUM
Category: Maintainability
File: openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md
Line: 1
Confidence: HIGH

Problem:
Most of the contradictions in this report exist because the same rules are written several times.

Evidence:
- There is a root `design.md` plus a non-standard `specs/basics-roles-permissions/design.md`, and they contradict each other (DB-001, BL-002, and "No changes to existing users table" against `lang`).
- The same scenarios are repeated across specs: multiple `assignRole` (`basics-auth/spec.md:76`, `basics-roles-admin/spec.md:45`), `syncRoles([])` (`basics-auth:81`, `basics-roles-admin:54`), the `manage` wildcard (`basics-auth:85`, `basics-permission-checks:14`), and user 1 hidden and immutable (`basics-auth:51-59`, `basics-super-admin`).
- Word counts against the ~500-word budget in `openspec/config.yaml`: `basics-auth` 757, `basics-roles-permissions` 749, `basics-roles-admin` 602, `basics-permission-checks` 520.
- `proposal.md`, `design.md` and `tasks.md` are in English, while the config requires Spanish.

Impact:
Implementers have to guess which document wins. Every inconsistency in this report traces back to the duplication.

Recommendation:
- Delete the nested design and keep one `design.md` with only the decisions that need justifying (schema and bootstrap).
- State each rule once, in the capability that owns it.
- Remove implementation detail from the specs (method lists, `once()`, emojis, "Chisel markers") so that they state invariants only.

## Low Findings

### BUG-004 — Reference `Language` enum does not compile and maps Basque to the Spanish flag
Severity: LOW
Category: Bug / Frontend
File: openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md
Line: 171, 180
Confidence: HIGH

Problem:
The reference enum has a compile error and a wrong flag mapping.

Evidence:
- `return isset(self::tryFrom($code));` is a PHP compile error, because `isset()` cannot take an expression.
- `self::EU->value => 'ES'` labels Euskara with the Spanish flag.

Impact:
Copying the reference code produces a fatal error. In a Basque-first UI (see the `ux-design` skill), the language column would also be mislabelled.

Recommendation:
This belongs in the separate language change (ARCH-002). There, use `self::tryFrom($code) !== null` and language codes or names, not country flags.

### FE-001 — Permission badges as emojis and a wide dynamic matrix do not follow the Flux/mobile list conventions
Severity: LOW
Category: Frontend
File: openspec/changes/add-roles-permissions-system/specs/basics-roles-permissions/design.md
Line: 380-388
Confidence: MEDIUM

Problem:
The summary page design ignores the project's list and accessibility conventions.

Evidence:
- The badges use emojis (👁️ ➕ ✏️ 🗑️ 📦 ♻️ 💀 ⭐).
- The table has one column per module, discovered at runtime.
- Recent list work (packages commit `a871610`, "cards on mobile") and the Flux icon conventions in `.github/instructions/` point to icons with text labels and card layouts on mobile.

Impact:
Screen readers announce the emoji names, and a matrix with an open-ended number of columns overflows on phones.

Recommendation:
Use Flux badges with text labels. On mobile, show each user as a card listing their modules. Scope the CSV export to the current filters.

### PERF-001 — Specify how permissions are loaded once per user instance
Severity: LOW
Category: Performance
File: openspec/changes/add-roles-permissions-system/specs/basics-permission-checks/spec.md
Line: 34-44
Confidence: MEDIUM

Problem:
The spec promises one query per request, but the reference code issues one query per check, and the wildcard rule needs three lookups per check.

Evidence:
- The scenario at `:40-44` promises one query for 50 rows.
- The reference `hasPermission()` (`design.md:170-175`) runs `roles()->whereHas(...)->exists()` on every call.
- The wildcard rule needs exact, `module.manage` and `*.manage` lookups.
- `once()` is keyed by the object instance (`Support/Onceable.php:47-50`), so it works for the authenticated user. However, `refreshPermissions()` can only call `Once::flush()`, which clears every memoized value in the process.

Impact:
On lists, each row's policy checks (edit, archive, delete) issue several queries if the plan is implemented as in the reference code.

Recommendation:
Load the set of active permission slugs into a property on the user instance the first time it is needed, and answer every check from it in memory. `refreshPermissions()` unsets that property. This needs no `once()` and has no global side effects.

## Informational Findings

### OPS-002 — Role removals leave no audit trail
Severity: INFO
Category: Observability
File: openspec/changes/add-roles-permissions-system/specs/basics-roles-admin/spec.md
Line: 34-37
Confidence: MEDIUM

Problem:
Only role grants are recorded.

Evidence:
`assigned_by` and `assigned_at` record grants, but `detach`/`sync` delete the pivot row. Nothing records who removed a role or when.

Impact:
Privilege changes cannot be reconstructed after the fact.

Recommendation:
If this is needed, a single log line in the assign/remove action is enough. No audit table is required.

## Security

SEC-001 (CRITICAL), SEC-002 (HIGH), SEC-003 (HIGH), SEC-004 (MEDIUM), OPS-001 (MEDIUM). BL-001 also has security impact: there is no default for self-registered users.

## Bugs / Correctness

BUG-001 (HIGH), BUG-002 (MEDIUM), BUG-003 (MEDIUM), BUG-004 (LOW), BL-002 (MEDIUM), BL-003 (MEDIUM).

## Database

DB-001, DB-002, DB-003 and DB-004 (all HIGH). Concurrency: the race on duplicate slugs is covered by the unique index once DB-002 is fixed. The "last superadmin" guard (BL-003) needs `lockForUpdate()` inside a transaction, like the existing delete guards.

## Performance

PERF-001 (LOW). Nothing else is demonstrable before implementation.

## Architecture

ARCH-001 (HIGH), ARCH-002, ARCH-003 and ARCH-004 (MEDIUM).

## Testing

TEST-001 (MEDIUM).

## Production

OPS-001 (MEDIUM) and OPS-002 (INFO). BUG-003 covers the trigger/binlog privilege issue.

## Maintainability

MAINT-001 (MEDIUM).

## Rejected Findings

| Candidate | Reason for rejection |
|---|---|
| "`once()` cannot give one query per request" | False as a general claim. `once()` memoizes per object instance, and the authenticated user is a single instance per request. Kept only as the narrower PERF-001. |
| Route `/roles/archived` shadowed by `/roles/{role}` | Unsupported. The existing user routes constrain `{user}` with `whereNumber`, and the same pattern applies. |
| Re-creating a `superadmin` slug after archiving it | Cannot happen: system roles cannot be archived. Superseded by DB-002 anyway. |
| "`openspec validate` fails" | False: it passes with `--strict`. |
| `hasPermission`/`hasPermissionTo` aliases, `hasAllRoles`, `func_get_args`, `isRtl()` unused | Style or speculative-API concerns with no demonstrated defect. Folded into the "state invariants, not methods" advice in MAINT-001. |
| "Resolución por prioridad" scenario wording | Wording only: an OR has no priority, but the expected result is correct. |
| CSV formula injection | No evidence yet that user-controlled names start with `=`/`+`/`-`/`@`, and the export is admin-only. Re-check during implementation. |
| Missing index on `role_user` by user | Depends on DB-001's final shape. With `constrained('users')`, MariaDB creates the FK index automatically. |

## Action Plan

Ordered by impact. Every step is an edit to the change's artifacts, not to code.

1. **Replace the "ID = 1" super-admin** (SEC-001, SEC-002, BUG-001, BL-003, OPS-001). Use either a seeded `superadmin` system role or a non-fillable flag. Bootstrap it through an artisan command that reads `config()` and fails on empty input. Hide the account in list queries, not with a global scope. Restrict only *other* users from editing it.
2. **Specify who may assign roles** (SEC-003): `roles.manage` to assign, the super-admin only for system or `*.manage` roles, and no self-assignment. Add the matching tests (TEST-001).
3. **Define the transition** (BL-001, ARCH-004): a default role for existing and registered users, or "allow all until enabled". Mark host-model and domain-policy changes as breaking.
4. **Fix the schema in one place** (DB-001 to DB-004):
   - `role_user`: plain `belongsToMany` on `user_id` with an FK and `assigned_by`.
   - `roles` and `permissions` built with `addCommonColumns` and `addAuditColumns`.
   - Slugs made unique with `addUniqueActiveColumnIndex`.
   - No CHECK constraint and no triggers.
5. **Reduce the scope** (ARCH-001, ARCH-002, ARCH-003):
   - `basics13` covers only users, roles and permissions.
   - Permissions become seeded and read-only.
   - Language, project members, entity types and the bridge selector move into their own changes. Replace `clients` with `entities`.
6. **Protect the flags** (SEC-004, BUG-003): `active` and `is_system` are not fillable and not editable in forms. Protect system roles in `RolePolicy`.
7. **De-duplicate the artifacts** (MAINT-001, BL-002, BUG-002):
   - One design document and one role/permission matrix.
   - One cache rule ("next request").
   - Fix `force_delete` and remove the incorrect reference code.
   - Translate `proposal.md`, `design.md` and `tasks.md` to Spanish.
   - Bring each spec under ~500 words.
8. **Fix the remaining LOW items** when the related pieces are reworked: BUG-004 in the language change, FE-001 in the summary view, PERF-001 in the implementation notes.
