# Design

## Context

Ver `proposal.md` (Why). Restricciones del código actual que condicionan el diseño:

- MariaDB 11.7 no tiene índices parciales. La unicidad entre los no eliminados ya se resuelve con una
  columna generada (`addUniqueActiveColumnIndex`).
- `addAuditColumns()` ya referencia `users` con FK, así que `basics13` ya asume esa tabla.
- `EloquentUserProvider` construye sus consultas con `newQuery()`, que aplica los global scopes.
- `updateRememberToken()` guarda el modelo de usuario, y lo mismo hacen la confirmación de 2FA y el
  cambio de contraseña.
- El registro abierto está activo por defecto (`basics13.auth.features.registration`).

## Goals / Non-Goals

**Goals:**

- Un esquema construido con los helpers existentes y válido en MariaDB.
- Que ninguna cuenta gane privilegios por su id ni pierda acceso al desplegar.

**Non-Goals:**

- Permisos de los dominios, miembros de proyecto e idioma del usuario (ver los changes enlazados en
  la propuesta).
- Permisos por registro (ownership). Aquí solo hay permisos por módulo.

## Decisions

### D1. Esquema

- `roles`: `name`, `slug`, `description`, `is_system`, más `addCommonColumns()` y
  `addAuditColumns()`. `addUniqueActiveNameIndex('roles')` y
  `addUniqueActiveColumnIndex('roles', 'slug')`.
- `permissions`: `slug` (único global), `module`, timestamps. Sin soft deletes ni `active`: son de solo
  lectura y los mantiene el paquete que los declara. La etiqueta visible sale de las traducciones por
  slug.
- `permission_role`: PK (`permission_id`, `role_id`), con las dos FK en cascada.
- `role_user`: PK (`role_id`, `user_id`). `role_id` con FK restrict, porque un rol con usuarios no se
  borra. `user_id` con FK a `users` en cascada. `assigned_by` es una FK nullable a `users` con
  `nullOnDelete`, y `created_at` guarda la fecha de asignación.
- `users.is_super_admin`: boolean, por defecto false.

Alternativas descartadas:

- **Pivote polimórfico:** no evita depender de `users`, porque ya hay FK de auditoría, y deja huérfanos
  sin FK.
- **Índice parcial:** MariaDB no lo soporta.
- **CHECK sobre `module`:** impide a los dominios añadir los suyos.
- **Triggers:** un soft delete es un `UPDATE`, y crear triggers exige privilegios con binlog.

### D2. Super-admin

- **Identificación:** el flag `is_super_admin` no es fillable. Lo pone solo un comando interactivo,
  `basics13:make-super-admin`, que pide nombre, correo y contraseña y no lee `env()`.
- **Ocultación:** se filtra con una condición explícita en `UserListQuery` (los tres scopes),
  `UserSelectOptionsQuery` y la query del resumen. No se usa un global scope, porque impediría el login.
- **Protección:** `UserPolicy` deniega a otros usuarios cualquier acción sobre la cuenta y cualquier
  cambio de sus roles. No se usan model events, porque bloquearían recordarme, 2FA y el cambio de
  contraseña.

Alternativas descartadas:

- **`id = 1`:** el primer usuario que se registre sería super-admin.
- **Un rol `superadmin`:** sería visible para sus iguales, y además quedaría un segundo concepto de
  acceso total.

### D3. Resolución de permisos

- La concern carga en una propiedad de la instancia los slugs de los permisos de sus roles activos y no
  eliminados, con una sola consulta. Las comprobaciones posteriores se hacen en memoria: el slug exacto
  o `{módulo}.manage`.
- El super-admin devuelve true sin consultar.
- `forgetPermissions()` vacía la propiedad, para tests y para cambios dentro de la misma petición.
- Las policies llaman a `hasPermissionTo()`. Las reglas sobre el registro objetivo (super-admin, uno
  mismo, rol de sistema) se comprueban antes del permiso. No hay `Gate::before`, porque el super-admin
  tampoco puede recibir roles.

Alternativa descartada: `once()` keyed por instancia. Funciona, pero la única forma de invalidarlo es
`Once::flush()`, que vacía todo el proceso.

### D4. Transición y datos iniciales

- Una migración de datos de `basics13` crea:
  - los permisos `users.*` y `roles.*`;
  - el rol de sistema `admin` (`users.manage`, `roles.manage`);
  - los roles `manager` (`users.view`, `users.create`, `users.edit`, `roles.view`) y `viewer`
    (`users.view`, `roles.view`).
- La misma migración asigna `admin` a cada usuario no eliminado. Es una migración y no un seeder porque
  así se ejecuta en cada despliegue y en los tests con `RefreshDatabase`.
- `CreateNewUser` asigna el rol `config('basics13.roles.registration_role')`, que por defecto es `null`.
- Los dominios añaden sus permisos, y los suman a `admin`, con su propia migración de datos en sus
  changes.

### D5. Permisos de solo lectura

Las policies comprueban slugs fijos en el código, así que crear, renombrar o borrar un permiso desde la
interfaz o no tendría efecto o rompería esas comprobaciones. La pantalla de permisos solo lista. El
formulario del rol los muestra como casillas agrupadas por módulo.

### D6. Concern obligatoria en el modelo del host

Alternativa considerada: un `Gate` que resuelva los permisos para cualquier modelo. Se descarta porque
añade una indirección sin necesidad. Hosts afectados: los que configuran su propio modelo. Se documenta
en el README como **BREAKING**.

### Rutas

Sin prefijo, igual que `users/...`, bajo `web`, `auth` y `verified`:

- `roles`, `roles/archived`, `roles/trash`
- `permissions` (solo `index`)
- `users/permissions-summary`

## Risks / Trade-offs

- [Instalación nueva sin super-admin: nadie puede asignar roles de sistema] → El README y el seeder de
  desarrollo de `demo13` ejecutan el comando. En una instalación existente los usuarios conservan
  `admin`.
- [Los tests actuales de usuarios crean actores sin roles] → Un estado de factory que da `admin`, y se
  actualizan esos tests en la misma tarea.
- [Un rol de sistema se queda sin permisos por error] → Solo el super-admin puede editarlo.

## Migration Plan

1. Desplegar y ejecutar `php artisan migrate`. Esto crea las tablas, siembra los datos iniciales y
   asigna `admin` a los usuarios existentes.
2. Ejecutar una vez `php artisan basics13:make-super-admin`.
3. Rollback: los `down()` borran las tablas nuevas y la columna `is_super_admin`, y al revertir el
   código las policies vuelven a permitirlo todo.
