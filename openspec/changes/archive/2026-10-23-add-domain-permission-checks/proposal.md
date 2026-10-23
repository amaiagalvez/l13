## Why

`add-roles-permissions-system` solo autoriza con permisos la gestión de usuarios y roles. Las policies
de los dominios siguen permitiéndolo todo a cualquier usuario verificado. Depende de
`add-roles-permissions-system`.

## What Changes

- `entities13` declara los módulos `entities` y `entity_types`, y `projects13` los módulos `projects` y
  `epics`, con las acciones de `basics-roles-permissions`. En la UI, "cliente" es la entidad: no hay
  un módulo `clients` aparte.
- Cada paquete inserta sus permisos con una migración de datos propia y añade su `{módulo}.manage` al
  rol de sistema `admin`, para que los usuarios existentes no pierdan acceso.
- Las policies de entidad, tipo de entidad, proyecto y epic comprueban `{módulo}.{acción}`. Añadir un
  comentario a un epic exige `epics.edit`. Planning y timeline exigen `projects.view`.
- El bridge permite elegir cliente en el formulario de proyecto a quien puede crear o editar proyectos,
  aunque no tenga `entities.view`.
- **BREAKING:** un usuario sin el permiso correspondiente recibe 403 en esos recursos.

## Capabilities

### New Capabilities

Ninguna.

### Modified Capabilities

- `entities-domain`: autorización por permisos `entities.*` y `entity_types.*`.
- `projects-domain`: autorización por permisos `projects.*`, incluidos planning y timeline.
- `epics-comments`: autorización por permisos `epics.*`.
- `entity-projects-bridge`: el selector de cliente del proyecto se autoriza por el permiso del proyecto.

## Impact

- `entities13`, `projects13`: migraciones de datos, policies, rutas de opciones y tests. Los actores de
  los tests reciben `admin`.
- `entity-projects13`: autorización del endpoint de opciones de cliente y tests cruzados.
- Sin cambios de esquema ni dependencias nuevas. Los dominios solo dependen de `basics13`, nunca entre
  sí.
