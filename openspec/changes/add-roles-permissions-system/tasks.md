# Tasks: Add Dynamic Roles and Permissions System

## Change ID
`add-roles-permissions-system`

## Phase 1: Database & Models (basics13)

- [ ] Create migration `create_roles_table.php`
  - Columns: id, name (unique), slug (unique, for code reference), description, is_system (bool, default false), active (bool, default true), deleted_at, created_at, updated_at
  - Partial unique index on slug where active = true AND deleted_at IS NULL
  - System roles cannot be deleted/archived (DB trigger/CHECK constraint + model event)

- [ ] Create migration `create_permissions_table.php`
  - Columns: id, name (unique), slug (unique, for code reference), description, module (string), active (bool, default true), deleted_at, created_at, updated_at
  - Partial unique index on slug where active = true AND deleted_at IS NULL
  - CHECK constraint on module column: `IN ('users', 'roles', 'permissions')` — basics13 core only

- [ ] Create migration `create_role_user_table.php` (pivot, polymorphic)
  - Columns: role_id, model_type, model_id, assigned_at
  - Composite primary key (role_id, model_type, model_id)
  - FK: role_id → roles.id CASCADE
  - No FK on model_id (polymorphic, app-level integrity)

- [ ] Create migration `create_permission_role_table.php` (pivot)
  - Columns: permission_id, role_id
  - Composite primary key (permission_id, role_id)
  - Foreign keys to permissions and roles

- [ ] Add system role protection: `Role` model `booted()` method with `deleting`/`updating` event that throws if `is_system = true`; optionally DB trigger for defense in depth

- [ ] **Create `Language` enum in `packages/basics13/src/Enums/Language.php`**
  - Cases: EU, ES, FR, EN with values 'eu', 'es', 'fr', 'en'
  - Methods: codes(), options(), flags(), isValid(), default(), name(), nativeName(), flag(), isRtl()
  - Used by: validation, forms, middleware, seeders, tests

- [ ] **Migration: Add `lang` column to `users` table**
  - `string('lang', 5)->nullable()->default(null)->after('email')`
  - Comment: 'Preferred language: eu, es, fr, en'

- [ ] Create `Role` model in `packages/basics13/src/Models/Role.php`
  - Fillable: name, slug, description, is_system, active
  - Casts: is_system (bool), active (bool)
  - Relationships: users() [MorphToMany], permissions()
  - Scopes: active(), system(), custom()
  - Chisel markers: `#[Fillable]`, `#[UsePolicy(RolePolicy::class)]`, `#[UseFactory(RoleFactory::class)]`
  - Soft deletes

- [ ] Create `Permission` model in `packages/basics13/src/Models/Permission.php`
  - Fillable: name, slug, description, module, active
  - Casts: active (bool)
  - Relationships: roles()
  - Scopes: active(), byModule()
  - Chisel markers: `#[Fillable]`, `#[UsePolicy(PermissionPolicy::class)]`, `#[UseFactory(PermissionFactory::class)]`
  - Soft deletes

- [ ] Create `RoleFactory.php` in `packages/basics13/database/factories/`
- [ ] Create `PermissionFactory.php` in `packages/basics13/database/factories/`

- [ ] **Update `User` model for super-admin (ID=1) + lang field**
  - Add Global Scope: `where('id', '!=', 1)` to hide from all queries (except auth)
  - Add `'lang'` to fillable array
  - Add `'lang' => 'string'` to casts
  - Update `UserFactory` to include random lang from `Language::codes()`
  - Model events: prevent update/delete/archive/forceDelete/role-assignment if `id === 1`

- [ ] Add `HasRoles` trait/concern in `packages/basics13/src/Concerns/HasRoles.php`
  - `roles()` relationship (MorphToMany via role_user, related model from config)
  - `hasRole(string|array $roles)` - checks active roles only; **returns true if `$this->id === 1`**
  - `hasAllRoles(array $roles)` - checks active roles only; **returns true if `$this->id === 1`**
  - `hasPermission(string|array $permissions)` - checks via active roles and active permissions; **returns true if `$this->id === 1`**
  - `hasPermissionTo(string $permission)` - alias; **returns true if `$this->id === 1`**; implements wildcard logic (exact → module.manage → *.manage)
  - `hasModuleAccess(string $module)` - returns hasPermissionTo("{$module}.manage")
  - `assignRole($role)` - assign role(s) to user; **throws if `$this->id === 1`**
  - `removeRole($role)` - remove role(s) from user; **throws if `$this->id === 1`**
  - `syncRoles(array $roles)` - sync roles; empty array allowed; **throws if `$this->id === 1`**
  - `getAllPermissions()` - returns collection of all effective permissions (union, active only); **returns all if `$this->id === 1`**
  - `loadPermissions()` - eager-loads permissions for current request (uses once())
  - `refreshPermissions()` - clears request cache (call after role/permission changes in same request)
  - `getRoleAssignments(): Collection` - returns pivot data

- [ ] Update `User` model to use `HasRoles` concern
  - Import and use the concern
  - Ensure it works with configurable user model via `ResolvesUserModel`

- [ ] **Create middleware `SetUserLocale` in `packages/basics13/src/Http/Middleware/`**
  - Reads `auth()->user()->lang`
  - Validates with `Language::isValid($user->lang)` before setting
  - Sets `app()->setLocale($user->lang ?? Language::default())`
  - Register in kernel (web group)

- [ ] Create config file `config/basics13.php` with `route_prefix` (default 'admin') and placeholders for future extensibility

## Phase 2: CRUD for Roles & Permissions (basics13)

- [ ] Create Form Requests:
  - `RoleRequest.php` (store/update validation)
  - `RoleListRequest.php` (list query params)
  - `RoleTrashDestroyRequest.php`
  - `RoleRestoreRequest.php`
  - `PermissionRequest.php` (store/update validation; module validated in basics13 core list)
  - `PermissionListRequest.php` (list query params)
  - `PermissionTrashDestroyRequest.php`
  - `PermissionRestoreRequest.php`
  - `UserPermissionsSummaryRequest.php` (filters for permissions summary)
  - `UserRequest.php` (store/update validation; lang ∈ Language::codes())

- [ ] Create Policies:
  - `RolePolicy.php` (viewAny, view, create, update, delete, restore, force_delete)
  - `PermissionPolicy.php` (viewAny, view, create, update, delete, restore, force_delete)
  - Superadmin bypass via `before()` method

- [ ] Create Controllers:
  - `RoleController.php` (index, create, store, show, edit, update)
  - `RoleTrashController.php` (index, destroy, restore)
  - `RoleArchivedController.php` (extends ArchivedController)
  - `PermissionController.php` (index, create, store, show, edit, update)
  - `PermissionTrashController.php` (index, destroy, restore)
  - `PermissionArchivedController.php` (extends ArchivedController)
  - `UserPermissionsSummaryController.php` (index)

- [ ] Create Queries:
  - `RoleListQuery.php` (extends ListQueryBase)
  - `PermissionListQuery.php` (extends ListQueryBase)
  - `UserPermissionsSummaryQuery.php` (extends ListQueryBase, eager loads roles.permissions, computes effective permissions per user grouped by module; discovers modules dynamically from Permission table)

- [ ] Create Transformers:
  - `RoleListTransformer.php` (extends ListTransformer)
  - `PermissionListTransformer.php` (extends ListTransformer)
  - `UserPermissionsSummaryTransformer.php` (extends ListTransformer, outputs permissions_by_module with dynamic modules)

## Phase 3: Views (basics13)

- [ ] Create Flux views for Roles:
  - `resources/views/roles/list.blade.php` (extends shared list layout)
  - `resources/views/roles/form.blade.php` (create/edit form)
  - Include: name, slug, description, is_system (read-only), active toggle

- [ ] Create Flux views for Permissions:
  - `resources/views/permissions/list.blade.php`
  - `resources/views/permissions/form.blade.php`
  - Include: name, slug, description, module select (validated against core + dynamically allowed), active toggle

- [ ] Create User Permissions Summary view:
  - `resources/views/users/permissions-summary.blade.php`
  - Matrix: Users (rows) × Modules (columns) — **modules dynamic from Permission table**
  - Columns: User, Roles, then one column per module (discovered dynamically)
  - Each cell shows: badges for actions user has (view, create, edit, delete, manage)
  - "manage" badge implies all actions
  - Filter by role, search by user name/email
  - Export to CSV button

- [ ] **Update User views for `lang` field**
  - `resources/views/users/form.blade.php`: add language selector using `Language::options()`
  - `resources/views/users/list.blade.php`: add lang column (flag + code) using `Language::flag($user->lang)` (ISO code for SVG asset)
  - Translation keys for: "Language", "Preferred language"
  - Locale native names come from `Language::nativeName()` (not translation files)

- [ ] Add navigation/menu entries for Roles, Permissions, and User Permissions Summary

## Phase 4: Translations (basics13)

- [ ] Add translation keys in `resources/lang/{es,eu,fr,en}.json` for:
  - Role: singular, plural, fields, actions, messages
  - Permission: singular, plural, fields, actions, messages
  - Modules: clients, projects, entities, users, roles, permissions
  - Actions: assign, revoke, sync
  - **Language field: "Language", "Preferred language"**
  - **Locale native names are in `Language` enum (not translation files): `Language::nativeName()`**

- [ ] Verify parity across 4 locales (existing test should catch this)

## Phase 5: Seeders & Default Data (basics13)

- [ ] **Create `SuperAdminUserSeeder.php` for fixed ID=1 user**
  - Creates user with `id = 1` (explicit ID)
  - Email from `env('SUPER_ADMIN_EMAIL', 'superadmin@example.com')`
  - Password from `env('SUPER_ADMIN_PASSWORD')` (hashed via `Hash::make()`)
  - `active = true`, `email_verified_at = now()`, `lang = 'es'` (default)
  - Runs first in seeder order (before roles/permissions)

- [ ] Create `RoleSeeder.php` with system roles:
  - Superadmin (slug: superadmin, is_system: true)
  - Admin (slug: admin, is_system: true)
  - Manager (slug: manager, is_system: false)
  - Viewer (slug: viewer, is_system: false)

- [ ] Create `PermissionSeeder.php` with base permissions (basics13 modules only):
  - users.view, users.create, users.edit, users.delete, users.archive, users.restore, users.force_delete, users.manage
  - roles.view, roles.create, roles.edit, roles.delete, roles.archive, roles.restore, roles.force_delete, roles.manage
  - permissions.view, permissions.create, permissions.edit, permissions.delete, permissions.archive, permissions.restore, permissions.force_delete, permissions.manage
  - *.manage (module `*`, action `manage`) — for superadmin wildcard

- [ ] Create `PermissionRoleSeeder.php` linking default roles to permissions (basics13 only):
  - Superadmin → all 24 permissions + *.manage
  - Admin → all except `roles.*`, `permissions.*`
  - Manager → `users.{view,create,edit}`, `roles.view`, `permissions.view`
  - Viewer → `users.view`, `roles.view`, `permissions.view`

> **Note:** Domain packages (`entities13`, `projects13`) will add their own permission seeders for modules `clients`, `projects`, `entities`, `entity_types` in separate PRs. They may extend default role permissions.

## Phase 6: Tests (basics13)

- [ ] Unit tests for `HasRoles` concern methods
- [ ] Feature tests for Role CRUD (list, create, update, archive, trash, restore, force_delete)
- [ ] Feature tests for Permission CRUD
- [ ] Feature tests for user-role assignment
- [ ] Feature tests for permission checking via `hasPermission()`
  - Wildcard logic: hasPermissionTo('projects.edit') true when user has projects.manage
  - Wildcard logic: hasPermissionTo('any.module.action') true when user has *.manage
  - Active filtering: inactive role/permission does not grant access
- [ ] Test duplicate slug race condition (UniqueConstraintViolation)
- [ ] Test system role protection (cannot delete/archive)
- [ ] Test translation parity
- [ ] Feature test: User Permissions Summary view
  - Matrix renders with correct permissions per user
  - Filters work (search, role, module columns)
  - Effective permissions = union of roles
  - CSV export matches displayed data
  - Manage badge shown when user has module.manage
  - Dynamic modules: columns appear for any seeded module
- [ ] **Cache contract tests:**
  - hasPermissionTo() uses once() cache per request
  - Multiple calls in same request = 1 query
  - refreshPermissions() clears cache
  - Changes visible on next request (not same request)
- [ ] **Tests for `lang` field:**
  - User create/edit validates lang ∈ `Language::codes()`
  - User list shows lang column with flag from `Language::flag()` (ISO code)
  - `SetUserLocale` middleware uses `Language::isValid()` before setting locale
  - Fallback to `Language::default()` when lang is null or invalid
  - Language selector in user form uses `Language::options()`
  - `Language` enum methods tested: cases(), codes(), options(), flags(), isValid(), default(), name(), nativeName(), flag(), isRtl()

- [ ] **Tests for super-admin user ID=1:**
  - Seeder creates user with id=1, correct email/password from env
  - User ID=1 hidden from UserListQuery (active, archived, trash)
  - User ID=1 hidden from UserPermissionsSummaryQuery
  - hasPermissionTo() returns true for any permission without DB query
  - hasRole() returns true for any role without DB query
  - assignRole/removeRole/syncRoles throw exception for ID=1
  - User ID=1 cannot be updated/deleted/archived via UserController
  - RolePolicy/PermissionPolicy allow ID=1 to manage system roles/permissions
  - RolePolicy/PermissionPolicy deny non-ID=1 from managing system roles/permissions
  - User ID=1 not in UserPermissionsSummary CSV export

## Phase 7: Spec Updates

- [ ] Update `specs/basics-auth/spec.md` - add requirement for roles relationship on User
- [ ] Create new specs `basics-roles-permissions`, `basics-permission-checks`, `basics-roles-admin` and `basics-super-admin`, and add the cross-domain selector requirement to `entity-projects-bridge`
- [ ] Create `specs/basics-roles-permissions/design.md` with technical design
- [ ] Validate specs with `openspec validate --strict`
- [ ] After archiving, add a "Roles y permisos" row to the specs map in `AGENTS.md` (`basics-roles-permissions`, `basics-permission-checks`, `basics-roles-admin`, `basics-super-admin`)

## Phase 8: Domain Package Integration (Future - Separate PRs)

- [ ] entities13: Add permission checks in EntityPolicy, EntityListQuery
- [ ] projects13: Add permission checks in ProjectPolicy, ProjectListQuery
- [ ] entity-projects13: Add cross-domain permission helpers if needed
- [ ] Each domain defines its own permission slugs in their seeders

### Phase 8a: projects13 - Project Members (Shared Projects)

**Database & Models**
- [ ] Create migration `create_project_members_table.php`
  - Columns: id, project_id, user_id, role (enum: owner, editor, viewer), added_at, added_by
  - Composite unique: (project_id, user_id)
  - FK: project_id → projects.id CASCADE, user_id → users.id CASCADE, added_by → users.id SET NULL
  - Index: (user_id, role)

- [ ] Create `ProjectMember` model in `packages/projects13/src/Models/ProjectMember.php`
  - Fillable: project_id, user_id, role, added_by
  - Relationships: project(), user(), addedBy()
  - Chisel markers: `#[Fillable]`, `#[UsePolicy]`, `#[UseFactory]`

- [ ] Update `Project` model
  - Add `members()` HasMany → ProjectMember
  - Add `owners()`, `editors()`, `viewers()` BelongsToMany with wherePivot('role')
  - Add helper methods: `isOwner(User)`, `canEdit(User)`, `canView(User)`

**Permissions & Seeders**
- [ ] Extend `PermissionSeeder.php` with member permissions:
  - projects.members.view, projects.members.create, projects.members.edit, projects.members.delete
  - projects.members.manage
- [ ] Update `PermissionRoleSeeder.php`:
  - Admin/Superadmin → projects.members.manage
  - Manager → projects.members.{view,create,edit}

**Policies**
- [ ] Update `ProjectPolicy.php`
  - view(): resource-level + owner OR member (any role)
  - update(): resource-level + owner OR editor/owner member
  - delete(): resource-level + owner only
  - manageMembers(): projects.manage permission + owner
  - addMember/removeMember(): delegate to manageMembers()

- [ ] Create `ProjectMemberPolicy.php`
  - viewAny/view/create/update/delete/restore/force_delete
  - Owner check via project ownership

**Controllers & Requests**
- [ ] Create `ProjectMemberController.php` (index, store, update, destroy)
- [ ] Create `ProjectMemberRequest.php` (user_id exists, role in:owner,editor,viewer)
- [ ] Add authorize() using ProjectMemberPolicy

**Queries**
- [ ] Update `ProjectListQuery.php`
  - Filter: WHERE created_by = user_id OR EXISTS project_members WHERE user_id = user_id (non-admin)

**Views (Flux)**
- [ ] Add members drawer in `resources/views/projects/partials/members-drawer.blade.php`
  - Table: user, role (badge), added_at, actions (edit role, remove)
  - Button: "Añadir miembro" → modal
  - Validate: cannot remove last owner, cannot demote last owner

**Tests**
- [ ] Feature: ProjectPolicy view/update/delete with members
- [ ] Feature: ProjectMember CRUD (add, change role, remove)
- [ ] Feature: Last owner protection (cannot remove/demote)
- [ ] Feature: ProjectListQuery filters by membership
- [ ] Dusk: Members drawer - add/edit/remove member

### Phase 8b: entities13 - Entity Types Permissions & Select Options

- [ ] PermissionSeeder: 8 actions for entity_types module (no CHECK constraint extension)
- [ ] EntityTypePolicy: viewAny, view, create, update, delete, archive, restore, force_delete
- [ ] EntityListQuery: filter by entities.view permission (active only)
- [ ] EntityTypeListQuery: filter by entity_types.view permission (active only)
- [ ] **EntitySelectOptionsQuery lives in `entity-projects13` bridge** (not entities13)
  - Returns entities for select dropdowns in other packages (projects, etc.)
  - **Contextual filtering (Option B):** No separate permission required
    - Superadmin/admin: all active entities
    - Other users: only entities from projects they own or are members of
  - Used by `projects13` ProjectFormComponent for entity select via basics13 interface
- [ ] Seeders: update default roles with entity permissions