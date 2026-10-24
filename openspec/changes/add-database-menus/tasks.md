# Tasks

## 1. Modelo y tablas (basics13)

- [x] 1.1 Migraciones `BAS_menus` (`key` único, `active_except` nullable) y `BAS_menu_permission` (FK con cascade), y modelo `Basics13\Models\Menu` con `permissions()`; verificar con un Feature test de la relación
- [x] 1.2 Migración de siembra de `basics13` (fecha posterior a sus permisos): grupo `administration` e ítems `admin-dashboard` (`sessions.view`), `users` (`users.view`, `users.view_own`, `users.manage_own`; excepción `users.permissions-summary.*`), `user-types` (`user_types.view`), `roles`, `permissions` y `permissions-summary` (`roles.view`), `sessions` (`sessions.view`; excepción `sessions.dashboard`) y `configuration` (`settings.view`). Debe hacer `upsert`, sincronizar el pivote, fallar ante un slug inexistente, llamar a `flush()` y, en `down()`, borrar sus claves. Verificar con tests de siembra repetida, permiso retirado en una versión nueva y slug inexistente
- [x] 1.3 Comprobar que las etiquetas de administración existen en `resources/lang/{es,eu,fr,en}.json` de `basics13`, y añadir "Administration" si falta; verificar con el render en `eu` de 2.2

## 2. Cálculo y componente (basics13)

- [x] 2.1 `Basics13\Support\NavigationMenu`: `items()` con `rememberForever` y memoización por petición, `flush()`, y `for(User)` que filtra con `Route::has`, `hasPermissionTo` y oculta grupos vacíos. Verificar con Feature tests de "Gestión delegada", "manage concede el ítem", "Grupo vacío" (sin permisos y con ruta no registrada), super-admin y "Permiso retirado" (archivando el rol por su ruta HTTP)
- [x] 2.2 Componente `<x-basics13::sidebar-menu />` con `active_pattern`/`active_except`, `wire:navigate` y `data-test="nav-{key}"`. Verificar con Feature tests: render en `eu`, cada caso de exclusión activa, y "Recarga" con permisos ya cargados y la lista ya cacheada → 0 consultas (`DB::enableQueryLog` alrededor del segundo render)

## 3. Menús de dominio

- [x] 3.1 `projects13`: migración de siembra (fecha posterior a sus permisos) del grupo `work` y los ítems `work-dashboard` (sin permisos), `projects`, `planning` y `timeline` (`projects.view`) y `epics` (`epics.view`), con sus etiquetas en sus propios JSON. Verificar con un test de siembra y con uno que compruebe que `projects` sale anidado bajo `work` en `NavigationMenu::for`. No referenciar `entities13`
- [x] 3.2 `entities13`: migración de siembra del grupo `data` y los ítems `entities` (`entities.view`) y `entity-types` (`entity_types.view`), con sus etiquetas en sus propios JSON. Verificar con un test de siembra. No referenciar `projects13`

## 4. Host (demo13)

- [x] 4.1 Sustituir los grupos de `layouts/app/sidebar.blade.php` por `<x-basics13::sidebar-menu />`, dejando fijo el Dashboard. Verificar con un Feature test que, como super-admin, encuentre los 9 `data-test` actuales (`nav-work-dashboard`, `nav-admin-dashboard`, `nav-users`, `nav-user-types`, `nav-roles`, `nav-permissions`, `nav-permissions-summary`, `nav-sessions`, `nav-configuration`), y que un usuario con solo `users.view_own` vea Work dashboard y Users, pero no Projects ni Roles
- [x] 4.2 Ejecutar `php artisan migrate` en Docker y comprobar que la siembra termina sin errores

## 5. Verificación final

- [x] 5.1 Suites de `basics13`, `entities13`, `projects13`, `entity-projects13` y `demo13`, más PHPStan nivel 7 y Pint; cobertura ≥ 80 %
- [x] 5.2 Pedir al usuario que ejecute Dusk de la navegación (`testing13/dusktest*.sh`) y revisar el resultado

## Workflow follow-up

- Tras el merge, archivar y añadir `basics-navigation-menus` al mapa de specs de `AGENTS.md`.
