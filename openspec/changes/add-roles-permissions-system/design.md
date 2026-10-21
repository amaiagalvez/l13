# Design: Dynamic Roles and Permissions System

## Change ID
`add-roles-permissions-system`

## Database Schema

### roles table
```sql
CREATE TABLE roles (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    description TEXT NULL,
    is_system BOOLEAN NOT NULL DEFAULT FALSE,
    active BOOLEAN NOT NULL DEFAULT TRUE,
    deleted_at TIMESTAMP NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    UNIQUE KEY roles_slug_active_unique (slug, active) WHERE active = TRUE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### permissions table
```sql
CREATE TABLE permissions (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    description TEXT NULL,
    module VARCHAR(100) NOT NULL,
    active BOOLEAN NOT NULL DEFAULT TRUE,
    deleted_at TIMESTAMP NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    UNIQUE KEY permissions_slug_active_unique (slug, active) WHERE active = TRUE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### role_user pivot table
```sql
CREATE TABLE role_user (
    role_id BIGINT UNSIGNED NOT NULL,
    user_id BIGINT UNSIGNED NOT NULL,
    assigned_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    assigned_by BIGINT UNSIGNED NULL,
    PRIMARY KEY (role_id, user_id),
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (assigned_by) REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### permission_role pivot table
```sql
CREATE TABLE permission_role (
    permission_id BIGINT UNSIGNED NOT NULL,
    role_id BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (permission_id, role_id),
    FOREIGN KEY (permission_id) REFERENCES permissions(id) ON DELETE CASCADE,
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Models

### Role Model
```php
namespace Basics13\Models;

use Basics13\Concerns\TracksAuditColumns;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\SoftDeletes;

#[Fillable(['name', 'slug', 'description', 'is_system', 'active'])]
#[UsePolicy(RolePolicy::class)]
#[UseFactory(RoleFactory::class)]
class Role extends Model
{
    use SoftDeletes, TracksAuditColumns;

    protected $casts = [
        'is_system' => 'boolean',
        'active' => 'boolean',
    ];

    public function users(): BelongsToMany
    {
        return $this->belongsToMany(
            config('auth.providers.users.model'),
            'role_user',
            'role_id',
            'user_id'
        )->withTimestamps()->withPivot('assigned_by');
    }

    public function permissions(): BelongsToMany
    {
        return $this->belongsToMany(Permission::class, 'permission_role');
    }

    // Scopes
    public function scopeActive($query) { return $query->where('active', true); }
    public function scopeSystem($query) { return $query->where('is_system', true); }
    public function scopeCustom($query) { return $query->where('is_system', false); }
}
```

### Permission Model
```php
namespace Basics13\Models;

use Basics13\Concerns\TracksAuditColumns;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\SoftDeletes;

#[Fillable(['name', 'slug', 'description', 'module', 'active'])]
#[UsePolicy(PermissionPolicy::class)]
#[UseFactory(PermissionFactory::class)]
class Permission extends Model
{
    use SoftDeletes, TracksAuditColumns;

    protected $casts = ['active' => 'boolean'];

    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class, 'permission_role');
    }

    public function scopeActive($query) { return $query->where('active', true); }
    public function scopeByModule($query, string $module) { return $query->where('module', $module); }
}
```

### HasRoles Concern
```php
namespace Basics13\Concerns;

use Basics13\Models\Role;
use Basics13\Models\Permission;
use Illuminate\Database\Eloquent\Relations\MorphToMany;
use Illuminate\Support\Collection;

trait HasRoles
{
    public function roles(): MorphToMany
    {
        return $this->morphToMany(
            Role::class,
            'model',
            'role_user',
            'user_id',
            'role_id'
        )->withTimestamps()->withPivot('assigned_by');
    }

    public function hasRole(string|array $roles): bool
    {
        $roles = is_array($roles) ? $roles : func_get_args();
        return $this->roles()->whereIn('slug', $roles)->exists();
    }

    public function hasAllRoles(array $roles): bool
    {
        return $this->roles()->whereIn('slug', $roles)->count() === count($roles);
    }

    public function hasPermission(string|array $permissions): bool
    {
        $permissions = is_array($permissions) ? $permissions : func_get_args();
        return $this->roles()
            ->whereHas('permissions', fn ($q) => $q->whereIn('slug', $permissions))
            ->exists();
    }

    public function hasPermissionTo(string $permission): bool
    {
        return $this->hasPermission($permission);
    }

    public function assignRole(string|Role|array $roles): void
    {
        $roles = is_array($roles) ? $roles : func_get_args();
        $roleModels = Role::whereIn('slug', $roles)->get();
        $this->roles()->syncWithoutDetaching($roleModels->pluck('id')->toArray());
    }

    public function removeRole(string|Role|array $roles): void
    {
        $roles = is_array($roles) ? $roles : func_get_args();
        $roleModels = Role::whereIn('slug', $roles)->get();
        $this->roles()->detach($roleModels->pluck('id')->toArray());
    }

    public function syncRoles(array $roles): void
    {
        $roleModels = Role::whereIn('slug', $roles)->get();
        $this->roles()->sync($roleModels->pluck('id')->toArray());
    }

    public function getAllPermissions(): Collection
    {
        return $this->roles()->with('permissions')->get()->flatMap->permissions->unique('id');
    }
}
```

## Permission Slug Convention
Format: `{module}.{action}`
- Modules: `clients`, `projects`, `entities`, `users`, `roles`, `permissions`
- Actions: `view`, `create`, `edit`, `delete`, `archive`, `restore`, `force-delete`, `manage` (all)

Examples:
- `clients.view` - view clients list
- `projects.edit` - create/edit projects
- `projects.manage` - full projects access
- `users.manage` - full user management
- `roles.manage` - role management (superadmin only)
- `permissions.manage` - permission management (superadmin only)

## Default System Roles & Permissions

### Superadmin (slug: `superadmin`, is_system: true)
- All permissions (`*.manage`)

### Admin (slug: `admin`, is_system: true)
- All except: `roles.manage`, `permissions.manage`

### Manager (slug: `manager`, is_system: false)
- `clients.view`, `clients.create`, `clients.edit`
- `projects.view`, `projects.create`, `projects.edit`
- `entities.view`, `entities.create`, `entities.edit`
- `users.view`

### Viewer (slug: `viewer`, is_system: false)
- `clients.view`, `projects.view`, `entities.view`, `users.view`

## Integration with Domain Packages

Each domain package (`entities13`, `projects13`) will:
1. Define its own permission slugs in its seeders
2. Use `Gate::define()` or Policy methods to check permissions
3. Use `auth()->user()->hasPermissionTo('module.action')` in policies/queries

Example in `entities13` Policy:
```php
public function viewAny(User $user): bool
{
    return $user->hasPermissionTo('entities.view');
}

public function update(User $user, Entity $entity): bool
{
    return $user->hasPermissionTo('entities.edit');
}
```

## Package Boundaries Compliance

- `basics13` owns: Role, Permission models, CRUD, user-role assignment
- Domain packages own: permission enforcement (policies, gates, query scopes)
- Bridge package: only adds cross-domain helpers if needed
- No domain → domain dependencies

## Migration Strategy for Existing Installations

1. Migrations are additive (new tables only)
2. No changes to existing `users` table
3. Default seeders run on fresh install
4. Existing users get no roles by default (opt-in)
5. Host can run seeders manually after upgrade