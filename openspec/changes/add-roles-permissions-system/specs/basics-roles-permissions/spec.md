# Spec: basics-roles-permissions

## Purpose
Define el sistema de roles y permisos dinámicos que ofrece `basics13`, permitiendo crear roles
personalizados, asignar múltiples roles a usuarios y definir permisos granulares por módulo.
La gestión (CRUD) vive en `basics13`; el control/validación (enforcement) ocurre en cada paquete
de dominio.

## Requirements

### Requirement: Roles dinámicos con slug único entre activos no eliminados
El sistema MUST permitir crear roles dinámicamente (no hardcoded). Cada rol MUST tener un `slug`
único entre los roles **activos y no eliminados** (índice parcial: `active = true AND deleted_at IS NULL`),
permitiendo reutilizar slugs de roles archivados (`active = false`) o en papelera (`deleted_at NOT NULL`).
Los roles del sistema (`is_system = true`) MUST estar protegidos: no se pueden
archivar, mover a papelera ni borrar definitivamente.
Los roles INACTIVOS (`active = false`) NO conceden permisos a los usuarios asignados.

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

### Requirement: Permiso wildcard `manage` por módulo y superadmin `*.manage`
El permiso `{module}.manage` MUST implicar acceso a todas las acciones del módulo.
El permiso `*.manage` (module `*`, action `manage`) MUST implicar acceso a TODOS los módulos y acciones (superadmin).
`hasPermissionTo('module.action')` MUST implementar la siguiente lógica de resolución:
1. **Match exacto**: usuario tiene rol con permiso `module.action` (activo)
2. **Module manage**: usuario tiene rol con permiso `module.manage` (activo)
3. **Superadmin wildcard**: usuario tiene rol con permiso `*.manage` (activo)

El método `hasModuleAccess(string $module): bool` MUST ser alias de `hasPermissionTo("{$module}.manage")`.

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

### Requirement: CRUD canónico para Roles y Permisos
Roles y Permisos MUST ser recursos canónicos en `basics13` con la estructura estándar:
- Controller + TrashController + ArchivedController (extends ArchivedController)
- Form Requests (store/update, list, trash destroy, restore)
- Policy (viewAny, view, create, update, delete, restore, force_delete)
- ListQuery + Transformer
- Vistas Flux (list.blade.php, form.blade.php)
- Soft deletes + papelera + flag `active` + restore + force delete guardado

#### Scenario: Listado de roles con filtros
- **WHEN** se accede a `/admin/roles`
- **THEN** se muestra tabla con: nombre, slug, descripción, badges (sistema/activo), acciones
- **THEN** filtros: búsqueda por nombre/slug, filtro por activo/sistema
- **THEN** paginación, ordenación, acciones en fila (editar, archivar, papelera)

#### Scenario: Formulario de rol
- **WHEN** se crea/edita un rol
- **THEN** campos: nombre (requerido), slug (auto-generado desde nombre, editable, único), descripción, checkbox "activo"
- **THEN** si `is_system = true`: slug y activo son solo lectura
- **THEN** selector de permisos (multi-select agrupado por módulo)

#### Scenario: Listado de permisos
- **WHEN** se accede a `/admin/permissions`
- **THEN** tabla agrupada por módulo: nombre, slug, descripción, activo, acciones
- **THEN** filtros por módulo, búsqueda, estado activo

### Requirement: Asignación de roles a usuarios
El sistema MUST proveer API para asignar/remover/sincronizar roles a cualquier usuario (modelo
configurable via `auth.providers.users.model`). La asignación se registra en `role_user` con
`assigned_at` y `assigned_by` (auditoría).

#### Scenario: Asignar rol vía API
- **WHEN** se llama `$user->assignRole('manager')`
- **THEN** se crea registro en `role_user` con `assigned_by = auth()->id()`
- **THEN** `$user->hasRole('manager')` retorna true

#### Scenario: Asignar múltiples roles
- **WHEN** se llama `$user->assignRole(['admin', 'manager'])`
- **THEN** usuario tiene ambos roles
- **THEN** permisos efectivos = unión de permisos de ambos roles

#### Scenario: Sincronizar roles (reemplazar)
- **WHEN** se llama `$user->syncRoles(['viewer'])`
- **THEN** roles previos removidos, solo queda `viewer`

#### Scenario: Sincronizar roles (reemplazar) — empty array allowed
- **WHEN** se llama `$user->syncRoles([])`
- **THEN** roles previos removidos, usuario queda sin roles
- **THEN** operación permitida (acción explícita de admin); UI debe confirmar antes de ejecutar

#### Scenario: Protección de último superadmin en Policy
- **WHEN** se intenta borrar/desactivar último usuario con rol `superadmin`
- **THEN** `RolePolicy::delete()` / `UserPolicy::delete()` rechaza la operación
- **THEN** error: "Cannot remove last superadmin"

### Requirement: Vista resumen de permisos por usuario
El sistema MUST proveer una pantalla de resumen que muestre, para cada usuario, sus roles asignados
y los permisos efectivos (unión de todos sus roles) agrupados por módulo.

#### Scenario: Matriz de permisos por usuario
- **WHEN** un admin accede a `/admin/users/permissions-summary`
- **THEN** ve tabla con filas = usuarios, columnas = módulos (users, roles, permissions, clients, projects, entities, entity_types...)
- **THEN** cada celda muestra badges de acciones: view, create, edit, delete, archive, restore, force_delete, manage
- **THEN** si tiene `manage`, solo muestra badge ⭐ manage (implícito todo)
- **THEN** puede filtrar por rol, buscar por nombre/email, mostrar/ocultar columnas módulo
- **THEN** puede exportar a CSV

#### Scenario: Permisos efectivos = unión de roles
- **WHEN** usuario tiene roles `editor` (entities.create, entities.edit) y `viewer` (entities.view)
- **THEN** en columna `entities` se muestran: view, create, edit
- **THEN** no hay duplicados, unión de permisos

### Requirement: Selectores cruzados sin permiso explícito (en bridge)
Para formularios que referencian recursos de otro dominio (ej: `projects.entity_id` → `entities`),
el sistema NO requiere permiso `*.select` explícito. En su lugar, las queries de opciones
(`EntitySelectOptionsQuery`) viven en el **bridge `entity-projects13`** y filtran contextualmente:

#### Scenario: Selector de entidad en formulario de proyecto
- **WHEN** usuario con `projects.create` pero SIN `entities.view` abre formulario de proyecto
- **WHEN** `EntitySelectOptionsQuery` (en `entity-projects13`) resuelve opciones
- **THEN** retorna solo entidades de proyectos donde el usuario es owner o member
- **THEN** usuario NO ve entidades ajenas en el select
- **WHEN** usuario envía formulario con `entity_id` válido de sus proyectos
- **THEN** validación pasa (ProjectFormRequest verifica acceso via policy/query)
- **WHEN** usuario intenta `entity_id` de proyecto ajeno
- **THEN** validación falla: "La entidad seleccionada no es accesible"

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

### Requirement: Usuario super-admin fijo (ID=1) oculto e inmutable
El sistema MUST incluir un usuario especial con `id = 1` que actúa como super-administrador del sistema:
- **Siempre tiene todos los permisos** (equivalente a `*.manage`) sin necesidad de roles asignados
- **NO aparece en listados** de usuarios (ni en listado activo, archivado, papelera, ni en resumen de permisos)
- **NO puede ser eliminado, archivado, desactivado ni modificado** por nadie (incluido él mismo)
- **NO se le pueden asignar/quitar roles** (su poder es intrínseco, no vía roles)
- **Es el único usuario que puede gestionar roles/permisos del sistema** (roles `is_system = true`)

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

### Requirement: Traducciones en 4 locales con paridad
Todas las cadenas de Roles y Permisos MUST tener traducción en `es`, `eu`, `fr`, `en` con paridad
verificada por test unitario existente.

#### Scenario: Etiquetas en euskera obligatorias
- **WHEN** se añaden claves de traducción para roles/permisos
- **THEN** `eu.json` tiene todas las claves que `es.json`
- **THEN** test de paridad pasa

### Requirement: Tests de Feature y Unit
MUST cubrir: CRUD roles/permisos, asignación usuario-rol, verificación permisos, carreras de
unicidad (slug duplicado), protección roles sistema, paridad traducciones.

#### Scenario: Carrera de slug duplicado
- **WHEN** dos requests concurrentes crean rol con mismo slug
- **THEN** uno succeeds, otro falla con `UniqueConstraintViolation`
- **THEN** no se crean roles duplicados