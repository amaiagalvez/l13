# Proposal: Add Dynamic Roles and Permissions System

## Change ID
`add-roles-permissions-system`

## Summary
Add a dynamic roles and permissions system where:
- Roles can be created dynamically (not hardcoded)
- Users can have multiple roles assigned
- Roles grant specific permissions (view users, manage roles, superadmin, etc.)
- Role/permission management lives in `basics13` package (core modules: users, roles, permissions)
- Domain packages (`entities13`, `projects13`) define their own permission modules (clients, projects, entities)
- Permission enforcement/control happens in each domain package

## Motivation
Currently the system only has basic authentication via `basics13` with a simple User model. There's no authorization layer. We need a flexible RBAC system that:
- Allows admins to create custom roles via UI
- Supports multiple roles per user
- Keeps permission definitions in the base package (core modules) but allows domain packages to extend
- Follows package boundaries (bridge → domains, never domain → domain)

## Scope
**In scope (basics13):**
- New `Role` and `Permission` models in `basics13`
- Many-to-many: User ↔ Role, Role ↔ Permission
- CRUD for roles and permissions in `basics13` (list, create, update, archive, trash)
- Policy gates / helper methods for permission checks (`hasPermissionTo`, `hasModuleAccess`)
- Integration with existing `basics-auth` capability (optional role selector in user form)
- Core permissions for modules: users, roles, permissions

**In scope (domain packages - separate PRs):**
- Domain-specific permission modules (clients, projects, entities)
- Permission enforcement in domain policies/queries

**Out of scope:**
- UI for assigning roles to users (can be added later)
- Passport/Socialite integration

## Affected Capabilities
- **Modified:** `basics-auth` - extends user model with roles relationship
- **New:** `basics-roles-permissions` - role and permission model
- **New:** `basics-permission-checks` - permission resolution (wildcard, per-request cache, configurable user model)
- **New:** `basics-roles-admin` - role/permission CRUD, user assignment and permissions summary
- **New:** `basics-super-admin` - fixed, hidden and immutable user ID=1
- **Modified:** `entity-projects-bridge` - cross-domain selectors without explicit permission

## Package Boundaries
- `basics13` owns: Role, Permission models, their CRUD, user-role assignment API
- Domain packages (`entities13`, `projects13`) own: permission enforcement via policies/gates
- `entity-projects13` bridge: may need cross-domain permission helpers if needed
- No domain-to-domain dependencies

## Implementation Approach
1. Create migrations for `roles`, `permissions`, `role_user` (polymorphic), `permission_role` tables
2. Add `Role` and `Permission` models with Chisel markers
3. Add `Language` enum (PHP 8.4) for locale management
4. Add `HasRoles` concern to User model (MorphToMany via polymorphic `role_user`)
5. Create Form Requests, Policies, Controllers for Role/Permission CRUD
6. Add Flux views for role/permission management + User Permissions Summary (dynamic modules)
7. Add translation files (es/eu/fr/en with parity)
8. Write Feature tests for CRUD, permission checking (wildcard, cache, active filter)
9. Update `basics-auth` spec to reflect the extension

## Breaking Changes
None - additive only. Existing installations work without roles/permissions until configured.

## Rollout Plan
1. Merge proposal → implement in `basics13`
2. Each domain package adopts permission checks independently
3. Bridge package adds any cross-domain helpers if needed