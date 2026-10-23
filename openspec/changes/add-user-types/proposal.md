## Why
Las cuentas no tienen agrupación propia y la gestión de usuarios es todo o nada (`users.*`). Hace falta clasificar usuarios por tipo y delegar su gestión a administradores de cada tipo.

## What Changes
- Nuevo recurso canónico `user-types` en `basics13` (CRUD, archivados, papelera), calcado de `entity-types` de `entities13`: nombre en 4 idiomas, notas, `active`, nombre único entre activos.
- Un usuario puede tener varios tipos (pivote `BAS_user_user_type` con flag `is_admin`). Se asignan, y se marca admin, desde el formulario de usuario por quien tenga `users.edit`.
- Un tipo unido a algún usuario (incluso en la papelera) no se puede eliminar.
- Permisos nuevos `user_types.{view,create,edit,delete,restore,force_delete,archive}`.
- Permiso nuevo `users.manage_own` ("Gestionar Mis Usuarios"): CRUD completo sobre sus usuarios (comparten un tipo activo donde el actor es admin). Asigna solo roles cuyos permisos ya tiene el actor y solo gestiona los tipos que administra.
- Permiso nuevo `users.view_own` ("Ver Mis Usuarios"): solo ver esos usuarios.
- Listados (activos, archivados, papelera), selector de usuarios y resumen de permisos: filtro por tipo y, para quien solo tiene permisos `_own`, limitados a sus usuarios.

## Capabilities
### New Capabilities
- (ninguna; se reutiliza `basics-auth`)

### Modified Capabilities
- `basics-auth`: tipos de usuario y gestión delegada por tipo.

## Impact
- `packages/basics13`: modelo `UserType`, migraciones (tabla, pivote, permisos), controllers (+Trash, +Archived), requests, `UserTypePolicy`, `UserTypeListQuery`, transformer, vistas `user-types/{list,form}`, `UserPolicy`/`UserListQuery`/`UserRequest`, navegación, lang es/eu/fr/en, tests.
- Sin cambios en otros paquetes; se ejecutan sus suites por tocar `basics13`.
