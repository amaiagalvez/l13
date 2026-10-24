# Design

## Context

La barra lateral de `demo13` decide la visibilidad de administración con `can('viewAny', ...)`, y Work y Data no la comprueban. Las policies dependen de permisos `{módulo}.{acción}`:

- Projects, Planning y Timeline: `projects.view`.
- Epics: `epics.view`.
- Entities: `entities.view`.
- Entity types: `entity_types.view`.
- Users: `users.view`, `users.view_own` o `users.manage_own`.
- Roles, Permissions y Permissions summary: `roles.view`.
- User types: `user_types.view`.
- Administration dashboard y Sessions: `sessions.view`.
- Configuration: `settings.view`.
- Work dashboard: ninguno.

`HasRoles` lee los slugs una vez por petición y resuelve `manage` en memoria (`hasPermissionTo`). La caché de demo13 es `database`, así que cada lectura de caché cuesta una consulta.

## Goals / Non-Goals

**Goals:** mover la barra lateral a datos con la misma visibilidad que exigen las policies, sin añadir consultas al pintarla.

**Non-Goals:** CRUD de menús, más de dos niveles, menús por tipo de usuario, el menú de usuario de la cabecera, el Dashboard del host (queda fijo en la vista).

## Decisions

- **Tablas**:
  - `BAS_menus`: `key` único, `parent_key` nullable, `package`, `label`, `icon`, `route`, `active_pattern`, `active_except` nullable, `sort`.
  - `BAS_menu_permission`: pivote con FK y cascade.

  El padre se enlaza por `key`, sin FK, para que el orden de las migraciones entre paquetes no importe. No hay soft delete ni `active`: son datos de siembra, no un recurso canónico.
- **Permisos como pivote "alguno de"**: cubre "Users" sin lógica especial.
- **Siembra por migración en cada paquete**:
  - Cada paquete hace `upsert` de su grupo e ítems por `key`.
  - Sincroniza el pivote de cada ítem en una transacción: borra sus filas y las reinserta.
  - Lanza una excepción si algún slug no existe, porque un ítem sin permisos sería público.
  - `down()` borra solo las claves del paquete.

  Las migraciones de menús llevan una fecha posterior a las de permisos del mismo paquete. `basics13` siembra `administration`, `projects13` siembra `work` y `entities13` siembra `data`, así que ningún paquete conoce la agrupación de otro.
- **Caché de la lista, no del menú de cada usuario**:
  - `NavigationMenu::items()` guarda en `Cache::rememberForever('basics13.menu.items')` todos los ítems con los slugs de sus permisos y ya ordenados.
  - `NavigationMenu::flush()` borra esa clave al final de cada migración de siembra.
  - Durante la petición el valor se memoriza para que la caché se lea una sola vez.
  - `NavigationMenu::for(User)` filtra la lista en memoria: primero `Route::has`, luego `$user->hasPermissionTo($slug)` para cada slug (que resuelve `manage` y el super-admin) y por último oculta los grupos vacíos.

  Alternativa descartada: una caché por usuario con un contador de versión global. Con el store `database` costaba 2 o 3 consultas por carga, frente a 0 de esta solución. También obligaba a invalidar desde `syncRoles`, `removeRole`, los eventos de `Role`, el pivote de permisos y el flag de super-admin, y dejaba entradas huérfanas. Ahora la visibilidad se calcula siempre con los permisos de la petición, así que nunca está desactualizada.
- **Ruta activa**: `request()->routeIs(active_pattern) && ! (active_except && request()->routeIs(active_except))`. `active_except` es una sola cadena y la usan Users (`users.permissions-summary.*`) y Sessions (`sessions.dashboard`). No se define ningún formato propio.
- **Componente**: `<x-basics13::sidebar-menu />` (Blade anónimo) pinta `flux:sidebar.group` y `flux:sidebar.item` con `__()`, `wire:navigate` y `data-test="nav-{key}"`. Las claves conservan los `data-test` actuales: `work-dashboard`, `admin-dashboard`, `users`, `user-types`, `roles`, `permissions`, `permissions-summary`, `sessions`, `configuration`. Las nuevas son `projects`, `epics`, `planning`, `timeline`, `entities` y `entity-types`.
- **Etiquetas**: cada paquete reutiliza sus propias claves en `resources/lang/{es,eu,fr,en}.json`.

## Risks / Trade-offs

- [Work y Data dejan de verse sin permiso] → es lo pretendido (hoy esas páginas dan 403); declarado en el proposal.
- [Si una migración de siembra cambia los ítems sin llamar a `flush()`, la lista cacheada queda antigua] → `flush()` va al final de cada migración y lo comprueba un test de siembra por paquete.
- [Una vez cacheada, la lista no toca la base de datos; mientras no esté cacheada, es una consulta] → coste único tras cada siembra.

## Migration Plan

Las migraciones crean las tablas y siembran los ítems. La vista del host se cambia en el mismo despliegue. Rollback: revertir la vista y hacer `migrate:rollback` en orden inverso de paquetes (dominios primero, `basics13` al final).
