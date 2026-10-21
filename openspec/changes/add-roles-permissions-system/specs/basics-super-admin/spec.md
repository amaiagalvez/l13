# Super-admin del sistema

## Purpose
Usuario fijo con `id = 1`, con acceso total, oculto e inmutable.

## ADDED Requirements

### Requirement: Usuario super-admin fijo (ID=1) oculto e inmutable
El sistema MUST incluir un usuario con `id = 1` que tiene todos los permisos sin roles asignados.
MUST NOT aparecer en ningún listado ni en el resumen de permisos, MUST NOT modificarse, archivarse,
desactivarse ni eliminarse (tampoco por sí mismo), MUST NOT recibir ni perder roles, y es el único
que MAY gestionar los roles y permisos del sistema (`is_system = true`).

#### Scenario: Usuario ID=1 tiene acceso total implícito
- **WHEN** `auth()->user()->id === 1`
- **THEN** `$user->hasPermissionTo('cualquier.cosa')` retorna true siempre
- **THEN** `$user->hasRole('cualquier-rol')` retorna true siempre (virtual)
- **THEN** no consulta BD para permisos/roles (short-circuit en `HasRoles`)

#### Scenario: Usuario ID=1 oculto en listados
- **WHEN** admin accede a `/admin/users` (listado, archivados, papelera)
- **THEN** usuario ID=1 NO aparece en resultados
- **WHEN** admin accede a `/admin/users/permissions-summary`
- **THEN** usuario ID=1 NO aparece en matriz

#### Scenario: Usuario ID=1 inmutable
- **WHEN** se intenta editar/borrar/archivar usuario ID=1
- **THEN** operación rechazada: "Usuario del sistema protegido"
- **WHEN** se intenta `syncRoles()` en usuario ID=1
- **THEN** operación rechazada: "Usuario del sistema no usa roles"

#### Scenario: Solo ID=1 puede tocar roles/permisos del sistema
- **WHEN** usuario NO es ID=1 intenta crear/editar/borrar rol con `is_system = true`
- **THEN** `RolePolicy` rechaza: "Solo super-admin del sistema puede modificar roles del sistema"
- **WHEN** usuario NO es ID=1 intenta crear/editar/borrar permiso de módulos core
- **THEN** `PermissionPolicy` rechaza

#### Scenario: Instalación fresh incluye usuario super-admin + roles y permisos base
- **WHEN** se ejecuta `php artisan db:seed --class=RolePermissionSeeder` en basics13
- **THEN** existe usuario con `id = 1`, `email = env('SUPER_ADMIN_EMAIL', 'superadmin@example.com')`
- **THEN** usuario ID=1 tiene `active = true`, `email_verified_at = now()`, password hasheado desde env
- **THEN** existen 4 roles (2 system, 2 custom)
- **THEN** existen permisos para 3 módulos base × 8 acciones = 24 permisos (users, roles, permissions) + 1 wildcard `*.manage`
- **THEN** superadmin tiene todos + `*.manage`, admin tiene todos menos roles/permissions, manager/users.view, viewer solo users.view
- **WHEN** se instalan `entities13`/`projects13` y ejecutan sus seeders
- **THEN** se añaden permisos para módulos `clients`, `projects`, `entities`, `entity_types`
- **THEN** roles existentes pueden actualizarse con nuevos permisos vía seeders de dominio
