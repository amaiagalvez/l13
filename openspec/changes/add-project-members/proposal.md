## Why

Con `add-domain-permission-checks`, quien tiene `projects.view` ve todos los proyectos. Hace falta
compartir cada proyecto solo con sus miembros. Depende de `add-roles-permissions-system` y de
`add-domain-permission-checks`.

## What Changes

- Miembros de proyecto en `projects13`, con rol `owner`, `editor` o `viewer`. Quien crea un proyecto
  queda como `owner`. Al migrar, el creador (`created_by`) de cada proyecto existente pasa a ser su
  `owner`.
- Sin `projects.manage`, un usuario solo ve los proyectos de los que es miembro (y sus epics), y la
  edición o el borrado dependen también de su rol en el proyecto. Con `projects.manage` lo ve y lo
  gestiona todo.
- Gestionan los miembros el `owner` del proyecto o quien tiene `projects.manage`. Un proyecto MUST
  conservar al menos un `owner`.
- El bridge limita las opciones de cliente de quien no tiene `entities.view` a los clientes de sus
  proyectos.
- Drawer de miembros en el formulario de proyecto.

## Capabilities

### New Capabilities

Ninguna.

### Modified Capabilities

- `projects-domain`: pertenencia a proyectos y su efecto en visibilidad y autorización.
- `entity-projects-bridge`: opciones de cliente acotadas a los proyectos del usuario.

## Impact

- `projects13`: migración de `PRO_project_members` (FK a proyectos y a `users`, igual que las columnas
  de auditoría) con la migración de datos de los owners, modelo y factory, `ProjectPolicy`,
  `EpicPolicy`, `ProjectListQuery`, `EpicListQuery`, planning, controller y request de miembros, drawer
  y traducciones es/eu/fr/en.
- `entity-projects13`: query de opciones de cliente y validación del proyecto.
- Sin dependencias nuevas.
