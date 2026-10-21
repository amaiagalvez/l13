# Verificación de permisos

## Purpose
Cómo se resuelve si un usuario tiene un permiso: wildcard `manage`, caché por request y modelo de usuario configurable.

## ADDED Requirements

### Requirement: Permiso wildcard `manage` por módulo y superadmin `*.manage`
`{module}.manage` MUST conceder todas las acciones del módulo y `*.manage` las de todos los módulos.
`hasPermissionTo('module.action')` MUST devolver true si algún rol activo del usuario tiene el permiso
exacto, `module.manage` o `*.manage`. `hasModuleAccess($module)` MUST equivaler a
`hasPermissionTo("{$module}.manage")`.

#### Scenario: Wildcard manage implica todas las acciones
- **WHEN** rol tiene permiso `projects.manage`
- **WHEN** se verifica `$user->hasPermissionTo('projects.edit')`
- **THEN** retorna true (manage implica edit, create, delete, etc.)

#### Scenario: Superadmin wildcard `*.manage`
- **WHEN** usuario tiene rol `superadmin` con permiso `*.manage`
- **WHEN** se verifica `$user->hasPermissionTo('cualquier.modulo.accion')`
- **THEN** retorna true

#### Scenario: Helper hasModuleAccess
- **WHEN** se llama `$user->hasModuleAccess('projects')`
- **THEN** retorna true si usuario tiene `projects.manage` o `*.manage`
- **THEN** retorna false si solo tiene permisos específicos (`projects.view`, `projects.edit`)

#### Scenario: Resolución por prioridad
- **WHEN** usuario tiene rol con `projects.view` y otro rol con `projects.manage`
- **WHEN** se verifica `$user->hasPermissionTo('projects.edit')`
- **THEN** retorna true (module.manage tiene prioridad sobre match exacto de otro permiso)

### Requirement: Verificación de permisos con cache por request (once())
El método `hasPermissionTo($slug)` MUST resolver permisos consultando la relación
`roles.permissions`. Para evitar N+1 en listados, el sistema USA `once()` helper de Laravel
para cachear el resultado por request. Múltiples llamadas en el mismo request retornan
el valor cacheado. Cambios en permisos de roles NO se reflejan en el mismo request;
se reflejan en el siguiente request O tras llamar `refreshPermissions()`.

#### Scenario: Cache por request evita N+1
- **WHEN** un listado itera 50 entidades y verifica `$user->hasPermissionTo('entities.edit')` en cada fila
- **THEN** se ejecuta UNA sola consulta para resolver permisos (cache por request via `once()`)
- **THEN** no hay 50 consultas separadas

#### Scenario: Cambios en mismo request requieren refresh explícito
- **WHEN** admin quita permiso `projects.delete` del rol `manager`
- **WHEN** usuario con rol `manager` verifica `$user->hasPermissionTo('projects.delete')` en el mismo request
- **THEN** retorna true (valor cacheado)
- **WHEN** se llama `$user->refreshPermissions()` y luego se verifica
- **THEN** retorna false (cache invalidado)
- **WHEN** en un nuevo request el usuario verifica `$user->hasPermissionTo('projects.delete')`
- **THEN** retorna false (cache invalidado por nuevo request)

#### Scenario: Comportamiento documentado y predecible
- **WHEN** código llama `hasPermissionTo()` múltiples veces en un request
- **THEN** siempre retorna el mismo valor (cacheado)
- **WHEN** código necesita valor fresco tras cambios
- **THEN** llama `refreshPermissions()` explícitamente

### Requirement: Integración con paquete de usuario configurable (pivote polimórfico)
El sistema MUST funcionar con el modelo de usuario configurado en `auth.providers.users.model`
(vía concern `ResolvesUserModel`). La tabla pivote `role_user` es **polimórfica** (`model_type`, `model_id`),
no referencia FK hardcodeada a `users`. El concern `HasRoles` usa `MorphToMany` con
`config('auth.providers.users.model')` para la clase del modelo relacionado en runtime.

#### Scenario: Host usa modelo User personalizado
- **WHEN** host configura `App\Models\CustomUser` en `auth.providers.users.model`
- **WHEN** se ejecuta migración `create_role_user_table`
- **THEN** tabla `role_user` tiene columnas `model_type` (string) y `model_id` (bigint) — polimórfica
- **THEN** `HasRoles` concern usa `MorphToMany` con `config('auth.providers.users.model')`
- **THEN** la integridad referencial se valida a nivel de aplicación (no FK cross-table)
