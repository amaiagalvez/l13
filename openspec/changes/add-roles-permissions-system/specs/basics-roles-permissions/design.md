# Design: basics-roles-permissions

## Change ID
`add-roles-permissions-system`

## Overview
Este documento describe el diseño técnico para la capability `basics-roles-permissions`, que
implementa un sistema RBAC (Role-Based Access Control) dinámico en `basics13`.

## Database Design

### Tables

#### roles
| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGINT UNSIGNED | PK, AI |
| name | VARCHAR(255) | NOT NULL |
| slug | VARCHAR(255) | NOT NULL |
| description | TEXT | NULL |
| is_system | BOOLEAN | NOT NULL DEFAULT FALSE |
| active | BOOLEAN | NOT NULL DEFAULT TRUE |
| deleted_at | TIMESTAMP | NULL |
| created_at | TIMESTAMP | NULL |
| updated_at | TIMESTAMP | NULL |

**Indexes:**
- UNIQUE `roles_slug_active_unique` (slug, active) WHERE active = TRUE AND deleted_at IS NULL

#### permissions
| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGINT UNSIGNED | PK, AI |
| name | VARCHAR(255) | NOT NULL |
| slug | VARCHAR(255) | NOT NULL |
| description | TEXT | NULL |
| module | VARCHAR(100) | NOT NULL |
| active | BOOLEAN | NOT NULL DEFAULT TRUE |
| deleted_at | TIMESTAMP | NULL |
| created_at | TIMESTAMP | NULL |
| updated_at | TIMESTAMP | NULL |

**Indexes:**
- UNIQUE `permissions_slug_active_unique` (slug, active) WHERE active = TRUE AND deleted_at IS NULL

**Constraints:**
- CHECK `module IN ('users', 'roles', 'permissions')` — basics13 core modules only
- Domain packages (`entities13`, `projects13`) validate module in FormRequest/Policy; no CHECK constraint extension (MySQL limitation)

#### role_user (pivot) — Polymorphic
| Column | Type | Constraints |
|--------|------|-------------|
| role_id | BIGINT UNSIGNED | PK, FK → roles.id CASCADE |
| model_type | VARCHAR(255) | PK, class name (e.g., `App\Models\User`) |
| model_id | BIGINT UNSIGNED | PK, FK → {model_table}.id CASCADE |
| assigned_at | TIMESTAMP | NOT NULL DEFAULT CURRENT_TIMESTAMP |

> **Note:** Polymorphic pivot supports any user model configured via `auth.providers.users.model`. No hardcoded FK to `users` table. `HasRoles` uses `MorphToMany` with `config('auth.providers.users.model')`. Referential integrity enforced at application level (no cross-table FK).

#### permission_role (pivot)
| Column | Type | Constraints |
|--------|------|-------------|
| permission_id | BIGINT UNSIGNED | PK, FK → permissions.id CASCADE |
| role_id | BIGINT UNSIGNED | PK, FK → roles.id CASCADE |

## Model Design

### Role Model
- Location: `packages/basics13/src/Models/Role.php`
- Traits: `SoftDeletes`, `TracksAuditColumns`
- Fillable: `name`, `slug`, `description`, `is_system`, `active`
- Casts: `is_system` → boolean, `active` → boolean
- Relationships:
  - `users()` → BelongsToMany (configurable user model via `role_user`)
  - `permissions()` → BelongsToMany (Permission via `permission_role`)
- Scopes: `active()`, `system()`, `custom()`
- Policy: `RolePolicy`
- Factory: `RoleFactory`

### Permission Model
- Location: `packages/basics13/src/Models/Permission.php`
- Traits: `SoftDeletes`, `TracksAuditColumns`
- Fillable: `name`, `slug`, `description`, `module`, `active`
- Casts: `active` → boolean
- Relationships:
  - `roles()` → BelongsToMany (Role via `permission_role`)
- Scopes: `active()`, `byModule(string $module)`
- Policy: `PermissionPolicy`
- Factory: `PermissionFactory`

### HasRoles Concern
- Location: `packages/basics13/src/Concerns/HasRoles.php`
- Provides methods on User model (or configurable user model):
  - `roles()` → MorphToMany (related model: `config('auth.providers.users.model')`, pivot: `role_user`, polymorphic)
  - `hasRole(string|array): bool` — checks active roles only; **returns true immediately if `$this->id === 1`**
  - `hasAllRoles(array): bool` — **returns true immediately if `$this->id === 1`**
  - `hasPermission(string|array): bool` — checks via active roles and active permissions; **returns true immediately if `$this->id === 1`**
  - `hasPermissionTo(string $permission): bool` — alias; implements wildcard logic:
    1. **Super-admin bypass:** if `$this->id === 1` return true
    2. Exact match: user has role with permission `module.action`
    3. Module manage: user has role with permission `module.manage`
    4. Superadmin wildcard: user has role with permission `*.manage` (module `*`, action `manage`)
  - `hasModuleAccess(string $module): bool` — returns `hasPermissionTo("{$module}.manage")`
  - `assignRole($role): void` — **throws if `$this->id === 1`** ("Usuario del sistema no usa roles")
  - `removeRole($role): void` — **throws if `$this->id === 1`**
  - `syncRoles(array $roles): void` — **throws if `$this->id === 1`**
  - `getAllPermissions(): Collection` — returns union of all active role permissions; **returns all permissions if `$this->id === 1`**
  - `loadPermissions(): void` — eager-loads permissions for current request (uses Laravel's `once()` helper to cache per-request)
  - `refreshPermissions(): void` — clears request cache (call after role/permission changes in same request)
  - `getRoleAssignments(): Collection` — returns pivot data for audit trail

> **Wildcard Logic:** `hasPermissionTo('projects.edit')` returns true if user has any role with `projects.edit` OR `projects.manage` OR `*.manage`.
> **Active Filtering:** Only roles with `active=true` AND `deleted_at IS NULL` grant permissions. Only permissions with `active=true` AND `deleted_at IS NULL` are considered.
> **Super-admin (ID=1):** Bypasses all checks; has implicit `*.manage`; hidden from queries via global scope.

### User Model Extension (super-admin ID=1 + lang field)
- **Super-admin user (ID=1):**
  - Created by seeder with fixed `id = 1`
  - Email from `env('SUPER_ADMIN_EMAIL', 'superadmin@example.com')`
  - Password from `env('SUPER_ADMIN_PASSWORD')` (hashed)
  - `active = true`, `email_verified_at = now()`
  - **Hidden from all queries:** Global scope `where('id', '!=', 1)` on User model (except for auth)
  - **Immutable:** Model events prevent update/delete/archive/role-assignment
  - **Permission bypass:** `HasRoles` checks `if ($this->id === 1) return true` at start of `hasPermissionTo()`/`hasRole()`

- **Migration:** Add `lang` column to `users` table
  - Type: `string(5)` (e.g., 'eu', 'es', 'fr', 'en')
  - Nullable: true (falls back to `config('app.locale')`)
  - Default: null
- **User Model:** Add to fillable and casts
  - Fillable: add `'lang'`
  - Casts: `'lang' => 'string'`
- **Language Class:** `Basics13\Support\Language` (or `Basics13\Enums\Language`)
  - Centralizes supported locales, native names, flags, RTL, etc.
  - Used by validation, forms, middleware, seeders
- **Validation:** In `UserRequest`, `lang` ∈ `Language::codes()`
- **Middleware:** `SetUserLocale` reads `auth()->user()->lang` and sets `app()->setLocale()`
- **Form:** User create/edit includes language selector from `Language::options()`

#### Language Class Reference Implementation
```php
// packages/basics13/src/Enums/Language.php

namespace Basics13\Enums;

enum Language: string
{
    case EU = 'eu';
    case ES = 'es';
    case FR = 'fr';
    case EN = 'en';

    public static function codes(): array
    {
        return array_column(self::cases(), 'value');
    }

    public static function options(): array
    {
        return [
            self::EU->value => self::EU->nativeName(),
            self::ES->value => self::ES->nativeName(),
            self::FR->value => self::FR->nativeName(),
            self::EN->value => self::EN->nativeName(),
        ];
    }

    public static function flags(): array
    {
        return [
            self::EU->value => 'ES',  // ISO 3166-1 alpha-2 for flag assets
            self::ES->value => 'ES',
            self::FR->value => 'FR',
            self::EN->value => 'GB',
        ];
    }

    public static function isValid(string $code): bool
    {
        return isset(self::tryFrom($code));
    }

    public static function default(): string
    {
        return config('app.locale', self::ES->value);
    }

    public function name(): string
    {
        return match ($this) {
            self::EU => 'Basque',
            self::ES => 'Spanish',
            self::FR => 'French',
            self::EN => 'English',
        };
    }

    public function nativeName(): string
    {
        return match ($this) {
            self::EU => 'Euskara',
            self::ES => 'Español',
            self::FR => 'Français',
            self::EN => 'English',
        };
    }

    public function flag(): string
    {
        return self::flags()[$this->value] ?? '';
    }

    public function isRtl(): bool
    {
        return false; // none of our locales are RTL
    }
}
```

## Permission Slug Convention

Format: `{module}.{action}`

**Modules (basics13 core):**
- `users` - Gestión de usuarios
- `roles` - Gestión de roles
- `permissions` - Gestión de permisos

**Modules (domain packages — added by respective packages via seeders, no CHECK constraint):**
- `clients` - Gestión de clientes (entities13)
- `projects` - Gestión de proyectos (projects13)
- `entities` - Gestión de entidades (entities13)
- `entity_types` - Gestión de tipos de entidad (entities13)

> Domain packages add modules via seeders only. Module validation in FormRequest/Policy, not CHECK constraint.

**Actions:**
- `view` - Ver listado
- `create` - Crear nuevo
- `edit` - Editar existente
- `delete` - Mover a papelera (soft delete)
- `archive` - Archivar/desarchivar
- `restore` - Restaurar de papelera
- `force_delete` - Borrado definitivo
- `manage` - Acceso total (implícitamente incluye todos)

## Default Data (Seeders)

### System Roles (is_system = true)
| Slug | Name | Description |
|------|------|-------------|
| superadmin | Superadministrador | Acceso total al sistema |
| admin | Administrador | Gestión completa excepto roles/permisos |

### Custom Roles (is_system = false)
| Slug | Name | Description |
|------|------|-------------|
| manager | Gestor | CRUD en usuarios, roles, permisos |
| viewer | Visualizador | Solo lectura en usuarios, roles, permisos |

### Permissions in basics13 (24 total: 3 modules × 8 actions)
Modules: `users`, `roles`, `permissions`
Each module gets: `view`, `create`, `edit`, `delete`, `archive`, `restore`, `force_delete`, `manage`

### Wildcard Permission
- `*.manage` (module `*`, action `manage`) — for superadmin

### Role-Permission Matrix (basics13 only)
| Role | Permissions |
|------|-------------|
| superadmin | All 24 permissions + `*.manage` |
| admin | All except `roles.*`, `permissions.*` |
| manager | `users.{view,create,edit}`, `roles.view`, `permissions.view` |
| viewer | `users.view`, `roles.view`, `permissions.view` |

> **Note:** Domain packages (`entities13`, `projects13`) add their own permissions for modules `clients`, `projects`, `entities`, `entity_types` via their own seeders. They may extend the default role permissions in their seeders.

## HTTP Layer

### Routes (basics13)
Route prefix configurable via `config('basics13.route_prefix', 'admin')`:
```
GET     /{prefix}/roles                → RoleController@index
GET     /{prefix}/roles/create         → RoleController@create
POST    /{prefix}/roles                → RoleController@store
GET     /{prefix}/roles/{role}         → RoleController@show
GET     /{prefix}/roles/{role}/edit    → RoleController@edit
PUT     /{prefix}/roles/{role}         → RoleController@update
DELETE  /{prefix}/roles/{role}         → RoleController@destroy

GET     /{prefix}/roles/archived       → RoleArchivedController@index
PUT     /{prefix}/roles/archived/{role} → RoleArchivedController@restore

GET     /{prefix}/roles/trash          → RoleTrashController@index
DELETE  /{prefix}/roles/trash/{role}   → RoleTrashController@destroy
PUT     /{prefix}/roles/trash/{role}   → RoleTrashController@restore

# Same pattern for /{prefix}/permissions
```

### Form Requests
- `RoleRequest` - validation for store/update
- `RoleListRequest` - list query params
- `RoleTrashDestroyRequest` - force delete confirmation
- `RoleRestoreRequest` - restore validation
- Same for Permission

### Policies
- `RolePolicy` - uses `before()` for superadmin bypass
- `PermissionPolicy` - uses `before()` for superadmin bypass
- Actions: viewAny, view, create, update, delete, restore, force_delete

### Queries
- `RoleListQuery` - extends `ListQueryBase`, filters: search, active, is_system
- `PermissionListQuery` - extends `ListQueryBase`, filters: search, module, active

### Transformers
- `RoleListTransformer` - columns: name, slug, description, is_system, active, permissions_count
- `PermissionListTransformer` - columns: name, slug, description, module, active, roles_count
- `UserPermissionsSummaryTransformer` - transforms user with roles + effective permissions grouped by module

## HTTP Layer (Additional)

### User Permissions Summary
```
GET     /{prefix}/users/permissions-summary  → UserPermissionsSummaryController@index
```

**Controller:** `UserPermissionsSummaryController`
- `index(UserPermissionsSummaryRequest $request, UserPermissionsSummaryQuery $query)`
- Returns paginated matrix data

**Form Request:** `UserPermissionsSummaryRequest`
- Validation for filters: search, role_ids[], modules[]

**Query:** `UserPermissionsSummaryQuery` (extends `ListQueryBase`)
- Eager loads: `roles.permissions`
- Filters: search (name/email), role_ids, active
- Computes effective permissions per user: union of all role permissions
- Groups by module for transformer

**Transformer:** `UserPermissionsSummaryTransformer`
- Output per user (modules dynamic):
  ```json
  {
    "id": 1,
    "name": "Juan Pérez",
    "email": "juan@example.com",
    "roles": [{"slug": "admin", "name": "Administrador"}],
    "permissions_by_module": {
      "users": ["view", "create", "edit", "manage"],
      "roles": ["view", "manage"],
      "permissions": ["view"],
      "projects": ["view", "create", "edit"],
      "entities": ["view"]
      // ... any other modules from Permission table
    }
  }
  ```

## Views

### Role Views
- `resources/views/roles/list.blade.php` - extends shared list layout
- `resources/views/roles/form.blade.php` - create/edit form with permission multi-select

### Permission Views
- `resources/views/permissions/list.blade.php` - grouped by module
- `resources/views/permissions/form.blade.php` - simple form

### User Permissions Summary View
- `resources/views/users/permissions-summary.blade.php`
- **Route:** `GET /{prefix}/users/permissions-summary` → `UserPermissionsSummaryController@index`
- **Matrix layout:** Users (rows) × Modules (columns) — **modules discovered dynamically**
- **Columns:**
  - User (name, email, avatar)
  - Roles (badges with role names)
  - One column per module: discovered from `Permission::distinct('module')->pluck('module')` (includes basics13 core + any domain modules seeded)
- **Cell content:** Permission badges per module
  - `view` → 👁️ badge
  - `create` → ➕ badge
  - `edit` → ✏️ badge
  - `delete` → 🗑️ badge
  - `archive` → 📦 badge
  - `restore` → ♻️ badge
  - `force_delete` → 💀 badge
  - `manage` → ⭐ badge (implies all, shows alone)
- **Filters:**
  - Search: user name, email
  - Role filter: multi-select from active roles
  - Module filter: show/hide columns
- **Actions:**
  - Click user → link to user edit (with role assignment)
  - Export CSV: User, Email, Roles, Module:Actions...
- **Data source:** `UserPermissionsSummaryQuery` + `UserPermissionsSummaryTransformer`

## Integration Points

### With basics-auth (User model)
- User model uses `HasRoles` concern
- `ResolvesUserModel` concern ensures configurable user model works
- Migration `role_user` references default `users` table; `HasRoles` resolves model at runtime

### With Domain Packages (entities13, projects13)
Each domain package independently:
1. Defines its permission slugs in its own seeders
2. Uses `$user->hasPermissionTo('module.action')` in Policies
3. Uses `$user->hasPermissionTo('module.action')` in ListQueries for filtering
4. No direct dependency on `basics13` models (only uses auth user methods)

### Package Boundaries
- `basics13` → owns Role, Permission, CRUD, assignment API
- `entities13` → owns EntityPolicy, EntityListQuery (checks permissions), EntitySelectOptionsQuery (contextual)
- `projects13` → owns ProjectPolicy, ProjectListQuery (checks permissions), ProjectMemberController
- `entity-projects13` → may add cross-domain helpers only if needed
- **Never**: domain → domain, domain → basics13 (except via user model methods)

### Cross-Domain Selectors (Contextual Queries)
Para selectores en formularios de un dominio que referencian otro dominio (ej: `project.entity_id` selecciona `entities`):

- **Opción B elegida:** Query contextual sin permiso extra `*.select`
- `EntitySelectOptionsQuery` vive en **`entity-projects13` bridge** (puede referenciar ambos dominios)
  - Superadmin/admin → todas las entidades activas
  - Usuario normal → solo entidades de proyectos donde es owner/member
- `projects13` usa `EntitySelectOptionsQuery` vía container/interface definida en basics13 (`SelectOptionsProvider`), sin importar modelos de `entities13` directamente
- Validación en `ProjectFormRequest` verifica que la entidad seleccionada sea accesible

## Migration Strategy

1. New tables only (additive)
2. No changes to existing `users` table
3. `role_user` pivot is polymorphic (`model_type`, `model_id`) — supports any user model via `config('auth.providers.users.model')`
4. Seeders run on fresh install; existing installs run manually
5. Default roles/permissions are opt-in (no auto-assignment to existing users)
6. Domain packages add modules via seeders only (no CHECK constraint extension)

## Testing Strategy

### Unit Tests
- `HasRoles` concern methods
- Role/Permission model scopes
- Slug generation/uniqueness

### Feature Tests
- Role CRUD (all states: active, archived, trashed)
- Permission CRUD
- User role assignment (assign, remove, sync)
- Permission checking via `hasPermissionTo()`
- Duplicate slug race condition
- System role protection
- Translation parity

### Browser Tests (Dusk)
- Role create/edit form with permission selection
- Permission list grouped by module
- User edit with role assignment (future)

## Configuration

Optional config file `config/basics13.php`:
```php
return [
    'route_prefix' => env('BASICS13_ROUTE_PREFIX', 'admin'),
];
```

Uses existing:
- `auth.providers.users.model` for user model resolution
- Standard Laravel gate/policy system for enforcement