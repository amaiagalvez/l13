# Proposal

## Why

El menú lateral está escrito a mano en `projects/demo13/resources/views/layouts/app/sidebar.blade.php`: la app conoce las rutas y los permisos de todos los paquetes, y cada paquete nuevo obliga a editar la vista del host. Además, los ítems de Work y Data se muestran a todos aunque su página responda 403. Guardar los menús en la base de datos permite que cada paquete aporte los suyos, unidos a los permisos que ya siembra, y que cada usuario vea solo lo que puede abrir.

## What Changes

- Tablas de menús en `basics13`: ítems de dos niveles (grupo y enlace) con clave de traducción, icono, ruta con nombre, patrones de ruta activa y de exclusión, orden y paquete dueño, y una relación muchos a muchos con permisos (basta con tener uno).
- Cada paquete siembra su grupo, sus ítems y sus etiquetas con una migración idempotente, como hace con los permisos: `basics13` el grupo de administración, `projects13` el de trabajo y `entities13` el de datos. La siembra falla si un permiso no existe.
- El menú visible se calcula en cada petición con los permisos que el usuario ya tiene cargados: un ítem sin permisos lo ve cualquier usuario autenticado, un grupo sin hijos visibles no aparece y el super-admin lo ve todo.
- La lista de ítems (no el menú de cada usuario) se guarda en caché, de modo que pintar el menú no añade consultas. Esa caché solo se invalida al sembrar.
- **Cambio de comportamiento**: los ítems de Work y Data pasan a exigir el mismo permiso que su página (`projects.view`, `epics.view`, `entities.view`, `entity_types.view`). Work dashboard sigue visible para todos.
- **BREAKING** (solo para el host): la barra lateral de `demo13` deja de listar enlaces a mano y pinta el menú calculado. El Dashboard del host queda fijo en la vista.
- Sin interfaz de gestión de menús: solo se cargan por siembra.

## Capabilities

### New Capabilities
- `basics-navigation-menus`: menús sembrados por paquete y visibilidad por permisos. Es una capability nueva porque ninguna existente cubre la navegación: `basics-roles-permissions` decide si un usuario tiene un permiso, no qué se le muestra, y `basics-shared-views` trata de listados y formularios.

### Modified Capabilities
<!-- Ninguna: la resolución de permisos no cambia. -->

## Impact

- `packages/basics13`: migraciones (tablas y siembra de administración), modelo `Menu`, cálculo del menú, componente de vista y tests.
- `packages/projects13`, `packages/entities13`: migración de siembra de su grupo e ítems, y tests.
- `projects/demo13`: la barra lateral usa el menú y conserva los `data-test` actuales.
- Sin dependencias nuevas; usa la caché configurada de Laravel.
