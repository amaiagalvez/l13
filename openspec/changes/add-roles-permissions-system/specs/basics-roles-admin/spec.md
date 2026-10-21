# Gestión de roles

## Purpose
Gestión en `basics13` de roles, permisos y su asignación a usuarios; el enforcement vive en cada dominio.

## ADDED Requirements

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

### Requirement: Traducciones en 4 locales con paridad
Todas las cadenas de Roles y Permisos MUST tener traducción en `es`, `eu`, `fr`, `en` con paridad
verificada por test unitario existente.

#### Scenario: Etiquetas en euskera obligatorias
- **WHEN** se añaden claves de traducción para roles/permisos
- **THEN** `eu.json` tiene todas las claves que `es.json`
- **THEN** test de paridad pasa
