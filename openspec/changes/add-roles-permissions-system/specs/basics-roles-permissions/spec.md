# Roles y permisos

## Purpose
Modelo de roles y permisos de `basics13`: roles dinámicos, permisos por módulo y su relación.

## ADDED Requirements

### Requirement: Roles dinámicos con slug único entre activos no eliminados
Los roles MUST crearse dinámicamente. El `slug` MUST ser único solo entre roles activos no
eliminados (índice parcial), así que el de un rol archivado o en la papelera se puede reutilizar.
Un rol del sistema (`is_system = true`) MUST NOT archivarse, ir a la papelera ni borrarse, y un rol
inactivo MUST NOT conceder permisos.

#### Scenario: Crear rol personalizado
- **WHEN** un admin crea un rol "Gestor de Clientes" con slug `client-manager`
- **THEN** el rol se crea con `is_system = false`, `active = true`
- **THEN** puede asignarse a usuarios inmediatamente

#### Scenario: No duplicar slug entre activos no eliminados
- **WHEN** se intenta crear un rol con slug `admin` existiendo ya un rol activo y no eliminado con ese slug
- **THEN** se rechaza por conflicto de unicidad (partial unique index)

#### Scenario: Reutilizar slug de rol archivado
- **WHEN** existe un rol archivado (`active = false`, `deleted_at IS NULL`) con slug `old-manager`
- **WHEN** se crea un nuevo rol con slug `old-manager`
- **THEN** se permite porque el índice único es parcial (solo activos no eliminados)

#### Scenario: Reutilizar slug de rol en papelera
- **WHEN** existe un rol en papelera (`deleted_at NOT NULL`) con slug `trashed-role`
- **WHEN** se crea un nuevo rol con slug `trashed-role`
- **THEN** se permite porque el índice único excluye eliminados

#### Scenario: Rol del sistema protegido
- **WHEN** se intenta archivar/mover a papelera/borrar el rol `superadmin` (`is_system = true`)
- **THEN** la operación se rechaza con error de autorización

#### Scenario: Rol del sistema protegido a nivel de base de datos
- **WHEN** se ejecuta `DELETE FROM roles WHERE is_system = true` o `UPDATE roles SET active = false WHERE is_system = true` directamente en BD
- **THEN** la operación falla por constraint/trigger a nivel de base de datos
- **THEN** los roles del sistema nunca pueden ser desactivados ni eliminados por ninguna vía

#### Scenario: Rol inactivo no concede permisos
- **WHEN** usuario tiene rol `editor` con `entities.edit` pero el rol está `active = false`
- **THEN** `$user->hasPermissionTo('entities.edit')` retorna false
- **WHEN** el rol se activa (`active = true`)
- **THEN** `$user->hasPermissionTo('entities.edit')` retorna true

### Requirement: Permisos por módulo con slug único entre activos no eliminados
Los permisos MUST agruparse por `module` (ej: `users`, `roles`, `permissions`, `clients`, `projects`, `entities`, `entity_types`). Cada permiso MUST tener un `slug` único entre permisos **activos y no eliminados** (índice parcial: `active = true AND deleted_at IS NULL`).
El slug sigue la convención `{module}.{action}` (ej: `projects.edit`, `clients.view`).
Los permisos INACTIVOS (`active = false`) NO son considerados en `hasPermissionTo()` ni asignables a nuevos roles.

#### Scenario: Crear permiso para módulo
- **WHEN** se crea permiso "Ver listado de proyectos" con slug `projects.view`, module `projects`
- **THEN** el permiso está disponible para asignar a roles

#### Scenario: Convención de slugs
- **WHEN** se definen permisos para un módulo
- **THEN** usan el patrón `{module}.{action}` donde action ∈ {view, create, edit, delete, archive, restore, force_delete, manage}

#### Scenario: Permiso inactivo no concede acceso
- **WHEN** permiso `projects.edit` existe pero está `active = false`
- **THEN** `$user->hasPermissionTo('projects.edit')` retorna false aunque usuario tenga rol con ese permiso
- **WHEN** el permiso se activa (`active = true`)
- **THEN** `$user->hasPermissionTo('projects.edit')` retorna true si usuario tiene rol con ese permiso

### Requirement: Relación many-to-many Role ↔ Permission
Un rol MUST poder tener múltiples permisos y un permiso MUST poder pertenecer a múltiples roles.
La asignación se gestiona vía tabla pivote `permission_role`. Los cambios en permisos de un rol
MUST reflejarse inmediatamente en `hasPermission()` de los usuarios con ese rol.

#### Scenario: Asignar permisos a rol
- **WHEN** se editan los permisos del rol `manager` añadiendo `projects.delete`
- **THEN** usuarios con rol `manager` ganan permiso `projects.delete` instantáneamente
- **THEN** `$user->hasPermissionTo('projects.delete')` retorna true sin recargar sesión

#### Scenario: Permiso heredado de múltiples roles
- **WHEN** usuario tiene roles `viewer` (con `clients.view`) y `editor` (con `clients.edit`)
- **THEN** `$user->hasPermissionTo('clients.view')` retorna true
- **THEN** `$user->hasPermissionTo('clients.edit')` retorna true
- **THEN** `$user->getAllPermissions()` incluye ambos

### Requirement: Tests de Feature y Unit
MUST cubrir: CRUD roles/permisos, asignación usuario-rol, verificación permisos, carreras de
unicidad (slug duplicado), protección roles sistema, paridad traducciones.

#### Scenario: Carrera de slug duplicado
- **WHEN** dos requests concurrentes crean rol con mismo slug
- **THEN** uno succeeds, otro falla con `UniqueConstraintViolation`
- **THEN** no se crean roles duplicados
