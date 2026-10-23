## Context
Las cuentas se autorizan hoy con `UserPolicy` + `users.{action}` (y `users.manage` concede todo el módulo). Este change añade autorización acotada por datos, patrón nuevo en el proyecto.

## Decisions
- **Tablas en `basics13`**: `BAS_user_types` (como `ENT_entity_types`) y pivote `BAS_user_user_type(user_id, user_type_id, is_admin, timestamps)`: PK `(user_id, user_type_id)`, FK `user_id` `cascadeOnDelete`, FK `user_type_id` `restrictOnDelete`, índice `(user_type_id, user_id)`. Guardia de borrado en `UserType::booted()` con `lockForUpdate`, igual que `EntityType`.
- **Relación**: `userTypes()` (con `withPivot('is_admin')`) en `src/Concerns/HasRoles.php`, porque el host puede reemplazar el modelo; `UserType::users()` contra `auth.providers.users.model`.
- **Slugs `users.view_own` y `users.manage_own`** en el módulo `users`: `users.manage` los implica. La policy comprueba primero el permiso global; el `_own` solo se evalúa si falta.
- **Alcance**: scope `managedBy($actor)`: usuarios con algún tipo activo (no archivado ni eliminado) donde el actor es admin, excluidos el actor, los admins de un tipo compartido y quien tenga `users.manage`/`users.edit`. Lo usan `UserPolicy` (pertenencia, `withTrashed`) y `UserListQuery` en activos, archivados y papelera, el selector `users/options` y el resumen de permisos con su export. Sin servicio ni capa nueva.
- **Tipos con `manage_own`**: solo puede añadir o quitar los tipos que administra, nunca `is_admin`, y debe quedar al menos uno. Alta y unión en un único `DB::transaction` junto con los roles.
- **Roles**: `assignsRoles()` acepta `users.manage_own` sobre un usuario del alcance manteniendo `! editsOwnAccount()`; `assignableRoleRule()` añade, para ese caso, que todos los permisos del rol los tenga el actor.
- **Suplantación**: sin cambios; `manage_own` no la concede.

## Risks / Trade-offs
- El filtro de roles por permisos del actor añade una consulta por validación; aceptable en un formulario.
- El scope añade un `EXISTS` a los listados, cubierto por la PK y el índice del pivote.
