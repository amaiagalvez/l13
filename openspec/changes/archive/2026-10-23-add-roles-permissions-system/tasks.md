# Tasks

## 1. Esquema y datos iniciales (basics13)

- [x] 1.1 Migración de `roles`, `permissions`, `permission_role` y `role_user` según D1. Verificación: test de esquema en MariaDB que confirma que el slug y el nombre son únicos entre los no eliminados (incluidos los archivados) y reutilizables tras la papelera, y que un rol con usuarios no se puede borrar por FK.
- [x] 1.2 Migración de `users.is_super_admin` (boolean, por defecto false). Verificación: test de esquema y `UserSchemaParityTest` actualizado.
- [x] 1.3 Migración de datos de D4: permisos `users.*` y `roles.*`, roles `admin` (sistema), `manager` y `viewer`, y asignación de `admin` a los usuarios no eliminados. Verificación: Feature test que migra con usuarios existentes (uno en la papelera) y comprueba sus roles, y que repetir la siembra no duplica nada.
- [x] 1.4 Modelos `Role` y `Permission` con factories. `is_system` y `active` fuera de `Fillable`. Verificación: Unit test de que esos atributos no son asignables en masa.

## 2. Resolución de permisos (basics13)

- [x] 2.1 Concern de roles en el `User` predeterminado según D3: `roles()`, `hasPermissionTo()`, `forgetPermissions()`, y asignar, quitar y sincronizar guardando `assigned_by`. Verificación: Feature tests de `manage`, unión de roles, rol archivado y rol eliminado sin efecto, super-admin sin consultas, y una sola consulta para N comprobaciones (`DB::getQueryLog`).
- [x] 2.2 README de `basics13`: los hosts con modelo propio tienen que añadir la concern (**BREAKING**). Verificación: revisión del README.

## 3. Super-admin (basics13)

- [x] 3.1 Comando `basics13:make-super-admin` interactivo. Falla si ya hay un super-admin, si el correo está en uso o si la contraseña está vacía. Verificación: Feature tests de cada caso con `expectsQuestion`.
- [x] 3.2 Excluir al super-admin en `UserListQuery` (activos, archivados, papelera) y en `UserSelectOptionsQuery`. Verificación: Feature tests de los tres listados y del endpoint de opciones.
- [x] 3.3 `UserPolicy` con permisos `users.*`. Deniega a otros usuarios cualquier acción sobre el super-admin. Un estado de factory da `admin` al actor, y se actualizan los tests y flujos Dusk de usuarios que existen. Verificación: tests de 403 sin permiso y sobre el super-admin, y la suite de usuarios en verde.
- [x] 3.4 El super-admin funciona con recordarme, cambio de contraseña y 2FA. Verificación: Feature tests de login con remember, logout, cambio de contraseña y confirmación de 2FA.

## 4. Registro y asignación de roles (basics13)

- [x] 4.1 Config `basics13.roles.registration_role` (por defecto `null`) y asignación en `CreateNewUser`. Verificación: Feature tests con y sin rol configurado. El primer registrado no es super-admin.
- [x] 4.2 Selector de roles en el formulario de usuario, solo con `roles.manage` y nunca en la propia cuenta. Los roles de sistema solo los ofrece y acepta para el super-admin. Validación en `UserRequest`. Verificación: Feature tests de autoasignación rechazada, de rol de sistema rechazado para quien no es super-admin y de asignación válida con `assigned_by`.

## 5. CRUD de roles (basics13)

- [x] 5.1 `RolePolicy`: permisos `roles.*`. Un rol de sistema solo lo edita el super-admin y nunca se archiva ni se borra. Un rol con usuarios no va a la papelera. Verificación: Unit tests de la policy.
- [x] 5.2 `RoleController`, `RoleTrashController`, `RoleArchivedController`, requests, `RoleListQuery`, `RoleListTransformer` y rutas siguiendo el recurso canónico. Verificación: Feature tests de CRUD, slug duplicado con archivado, restauración con conflicto de slug, flags del payload ignorados y 403.
- [x] 5.3 Vistas `roles/list` y `roles/form`: slug autogenerado desde el nombre (editable, solo lectura en roles de sistema) y casillas de permisos agrupadas por módulo. Sin casilla de activo. Verificación: Dusk test de la autogeneración del slug y del guardado de permisos.

## 6. Permisos y resumen (basics13)

- [x] 6.1 Listado de permisos de solo lectura agrupado por módulo (`roles.view`). Verificación: Feature test de listado y de que no hay rutas de escritura.
- [x] 6.2 Resumen de permisos efectivos (`roles.view`): query, transformer y vista con tarjetas en móvil y badges de Flux con texto. CSV de las filas filtradas. Excluye al super-admin. Verificación: Feature tests de unión de roles, `manage`, filtros, CSV filtrado y super-admin oculto.
- [x] 6.3 Entradas de navegación de roles, permisos y resumen, visibles según su permiso. Verificación: Feature test del menú con y sin permiso.

## 7. Traducciones (basics13)

- [x] 7.1 Claves en `resources/lang/{es,eu,fr,en}.json` para roles, permisos (una etiqueta por slug), resumen, comando y mensajes de denegación. Verificación: los tests de `TranslationFilesTestCase` pasan.

## 8. Verificación final

- [x] 8.1 Suites de `basics13`, `entities13`, `projects13`, `entity-projects13` y `demo13`, Dusk, PHPStan nivel 7 y Pint, todo en Docker. La cobertura se mantiene ≥ 80 %.

## Workflow follow-up

- Al archivar: añadir la fila "Roles y permisos" (`basics-roles-permissions`, `basics-super-admin`) al mapa de specs de `AGENTS.md`, y actualizar la sección Authentication / Authorization de `.github/docs/architecture/ARCHITECTURE.md` (deja de valer "every authenticated verified user").
