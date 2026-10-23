## 1. Datos
- [x] 1.1 Migraciones `BAS_user_types` (índice único de nombre entre activos) y pivote `BAS_user_user_type` con `is_admin`, PK `(user_id, user_type_id)`, FK `user_id` cascade y `user_type_id` restrict, índice `(user_type_id, user_id)`
- [x] 1.2 Migración de permisos `user_types.*`, `users.view_own`, `users.manage_own` con `insertOrIgnore` (`user_types.*` al rol `admin`)
- [x] 1.3 Modelo `UserType` + factory + seeder (y tipo con admin de demo en demo13); `userTypes()` en `HasRoles`, `UserType::users()` contra el modelo configurado; guardia de borrado

## 2. Recurso user-types
- [x] 2.1 `UserTypeController`, `UserTypeTrashController`, `UserTypeArchivedController`, requests, `UserTypePolicy`, `UserTypeListQuery`, `UserTypeSelectOptionsQuery` (solo activos), transformer y rutas, siguiendo `entities13` EntityType
- [x] 2.2 Vistas `user-types/{list,form}` y enlace de navegación protegido por `user_types.view`

## 3. Gestión delegada
- [x] 3.1 Scope `managedBy` (tipos activos, excluye actor, admins compartidos y gestores globales); `UserPolicy` acepta `_own` dentro del alcance tras el permiso global
- [x] 3.2 Aplicar el alcance en `UserListQuery` (activos, archivados, papelera), `users/options` y el resumen de permisos con su export; filtro por tipo
- [x] 3.3 `UserRequest`/`UserController`: tipos e `is_admin` con `users.edit`; con `manage_own` solo tipos que administra y al menos uno; alta, unión y roles en una transacción
- [x] 3.4 Roles con `manage_own`: `assignsRoles()` mantiene `! editsOwnAccount()`; `assignableRoleRule()` limita a roles cuyos permisos tiene el actor
- [x] 3.5 Vista de formulario/listado de usuarios con tipos

## 4. i18n y tests
- [x] 4.1 Textos y etiquetas de los permisos nuevos en `resources/lang/{es,eu,fr,en}.json`
- [x] 4.2 Feature: CRUD/trash/archived de user-types; guardia de borrado con usuario activo y en papelera (delete y force delete); force delete de un usuario libera el tipo; carrera de inserción duplicada → error de validación
- [x] 4.3 Feature: matriz de autorización por ruta de usuarios (index, store, update, destroy, archive/activate, trash restore/destroy, options, resumen) × {view_own dentro/fuera, manage_own dentro/fuera, gestor global}; miembro no admin excluido, sin filas duplicadas, tipo archivado sin alcance
- [x] 4.4 Feature: escalada denegada (rol con permisos ajenos, `is_admin`, tipos ajenos, propia cuenta, admin compartido, super-admin, suplantación)

## 5. Verificación
- [x] 5.1 Suite de `basics13`, PHPStan nivel 7, Pint y suites de entities13, projects13, entity-projects13 y demo13; pedir al usuario el comando Dusk si hay JS
