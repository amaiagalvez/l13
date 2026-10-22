# Tasks

## 1. Datos (projects13)

- [ ] 1.1 Migración de `PRO_project_members` (`project_id`, `user_id`, `role`, `added_by`, timestamps), único (`project_id`, `user_id`), FK a proyectos y a `users`. Migración de datos: el `created_by` de cada proyecto pasa a `owner`. Verificación: test de esquema (duplicado rechazado) y Feature test de la migración de owners.
- [ ] 1.2 Modelo `ProjectMember` con factory y relación `members()` en `Project`. Crear un proyecto añade a su autor como `owner`. Verificación: Feature test del alta.

## 2. Autorización y visibilidad (projects13)

- [ ] 2.1 `ProjectPolicy` y `EpicPolicy`: con `projects.manage` todo. Si no, la pertenencia y el rol según la spec. Verificación: Unit tests de las policies por rol de miembro.
- [ ] 2.2 `ProjectListQuery`, `EpicListQuery`, planning y timeline acotados a los proyectos del usuario sin `projects.manage`. Un proyecto ajeno por URL devuelve 404. Verificación: Feature tests de listados, planning y acceso directo.

## 3. Gestión de miembros (projects13)

- [ ] 3.1 Controller, request y rutas de miembros (añadir, cambiar rol, quitar), con la guarda del último owner bloqueando el proyecto (`lockForUpdate()` en transacción). Verificación: Feature tests de CRUD de miembros, del último owner y de 403 para quien no es owner.
- [ ] 3.2 Drawer de miembros en el formulario de proyecto. Verificación: Dusk test de añadir, cambiar rol y quitar.
- [ ] 3.3 Traducciones es/eu/fr/en. Verificación: tests de traducciones en verde.

## 4. Bridge (entity-projects13)

- [ ] 4.1 Opciones de cliente y validación acotadas a los proyectos del usuario sin `entities.view`. Verificación: Feature tests de opciones acotadas y de cliente ajeno rechazado.

## 5. Verificación final

- [ ] 5.1 Suites de `projects13`, `entity-projects13` y `demo13`, Dusk, PHPStan nivel 7 y Pint en Docker. La cobertura se mantiene ≥ 80 %.
