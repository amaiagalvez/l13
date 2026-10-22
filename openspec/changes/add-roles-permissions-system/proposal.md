## Why

Hoy cualquier usuario verificado puede hacerlo todo: `UserPolicy` concede todas las habilidades y las
policies de dominio también. Hace falta limitar lo que hace cada cuenta mediante roles definidos
desde la interfaz, sin acoplar `basics13` a los dominios. Revisado según el code review del
2026-10-22 (`.github/reviews/2026-10-22/CODE-REVIEW.md`).

## What Changes

- Roles dinámicos en `basics13` como recurso canónico (listado, alta, edición, archivados y papelera),
  cada uno con su conjunto de permisos. Un usuario puede tener varios roles.
- Permisos sembrados y de solo lectura con slug `{módulo}.{acción}`. Cada paquete declara los de los
  módulos que comprueban sus policies; `basics13` solo declara `users` y `roles`. La interfaz los
  lista y los asigna desde el formulario del rol.
- Asignar roles exige `roles.manage`. Nadie cambia sus propios roles y los roles de sistema solo los
  asigna el super-admin.
- Cuenta super-admin identificada por un flag que no se puede asignar y que solo pone un comando de
  consola. Tiene todos los permisos, no aparece en listados ni en selectores, y otros usuarios no
  pueden modificarla.
- Resumen de los permisos efectivos de cada usuario, con exportación CSV de lo filtrado.
- **BREAKING:** la gestión de usuarios pasa a exigir permisos `users.*`. Una migración da el rol de
  sistema `admin` a los usuarios existentes, y los nuevos registros reciben el rol configurado
  (ninguno por defecto).
- **BREAKING:** el modelo de usuario configurado por un host MUST usar la concern de roles.

Fuera de este change, cada uno en su propio change:

- `add-user-language`: idioma preferido del usuario.
- `add-domain-permission-checks`: permisos de entidades, tipos de entidad, proyectos, epics y
  comentarios en sus policies.
- `add-project-members`: miembros de proyecto y selector contextual de entidades en el bridge.

## Capabilities

### New Capabilities

- `basics-roles-permissions`: roles, permisos sembrados, cómo se resuelve si un usuario tiene un
  permiso, quién asigna roles y el resumen de permisos. Ninguna capability existente cubre la
  autorización.
- `basics-super-admin`: la cuenta con acceso total, cómo se crea y su protección. Va separada porque
  afecta a la autenticación y a los listados de usuarios, no solo a los roles.

### Modified Capabilities

- `basics-auth`: el modelo de usuario configurable expone roles, la gestión de cuentas se autoriza con
  permisos y el registro asigna el rol configurado.

## Impact

- `basics13`:
  - Migraciones de `roles`, `permissions`, `permission_role`, `role_user` y `users.is_super_admin`, más la migración de datos inicial.
  - Modelos, concern, policies, controllers, requests, queries, transformers, vistas, rutas, comando y traducciones es/eu/fr/en.
- Cambian `UserPolicy`, `UserListQuery`, `UserSelectOptionsQuery`, `CreateNewUser` y los tests de
  usuarios existentes, porque el actor necesita permisos.
- Hosts con modelo de usuario propio: añadir la concern (**BREAKING**).
- Sin dependencias nuevas.
