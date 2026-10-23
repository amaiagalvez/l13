# Proposal

## Why

Los superadministradores necesitan poder iniciar sesión como otro usuario para diagnosticar problemas, verificar permisos o realizar tareas en nombre de ese usuario sin necesidad de conocer su contraseña. Esta funcionalidad de "impersonación" es estándar en sistemas de administración y mejora la capacidad de soporte y depuración.

## What Changes

- **Nueva capacidad**: `basics-user-impersonation` - Impersonación de usuarios por superadministradores
- **Nueva ruta**: `users.{user}.impersonate` (POST) para iniciar la impersonación
- **Nuevo controlador**: Método `impersonate` en `UserController` (o nuevo `UserImpersonationController`)
- **Nuevo permiso**: `users.impersonate` - concedido solo a superadministradores
- **Nueva política**: Habilidad `impersonate` en `UserPolicy` que verifica superadmin y que no se impersone a sí mismo ni a otro superadmin
- **Transformador**: Nueva acción `impersonateAction` en `UserListTransformer` visible solo para superadministradores en la lista de activos
- **Vista**: El botón "Impersonate" aparece en las acciones de fila de la tabla de usuarios
- **Middleware/servicio**: Lógica de cambio de usuario guardando el usuario original en sesión para poder volver
- **Ruta de retorno**: `impersonation.leave` para volver al usuario original
- **Tests**: Unitarios (política, transformador), Feature (endpoint, flujo completo), Browser (UI)
- **i18n**: Usar claves existentes `actions.impersonate` / `actions.impersonation` en los 4 locales

## Capabilities

### New Capabilities

- `basics-user-impersonation`: Impersonación de usuarios por superadministradores. Cubre el permiso, la política, el endpoint, la acción en la lista y el flujo de vuelta al usuario original.

### Modified Capabilities

Ninguna. Queda para un change posterior invalidar la impersonación si el superadministrador original
se elimina o se desactiva durante ella.

## Impact

**Código afectado (basics13)**:
- `src/Http/Controllers/UserController.php` - nuevo método `impersonate`
- `src/Http/Controllers/UserController.php` - nuevo método `leaveImpersonation` (o controlador aparte)
- `src/Policies/UserPolicy.php` - nueva habilidad `impersonate`
- `src/Transformers/UserListTransformer.php` - nueva acción `impersonateAction`
- `routes/users.php` - nuevas rutas `users.{user}.impersonate` e `impersonation.leave`
- `database/seeders/PermissionSeeder.php` - seeder para añadir permiso `users.impersonate` y asignarlo al rol superadmin (si existe) o documentarlo
- `resources/views/users/list.blade.php` - sin cambios (usa componente compartido)
- `resources/lang/{en,es,eu,fr}/actions.php` - claves ya existen
- `tests/Feature/Users/UserImpersonationTest.php` - nuevos tests
- `tests/Unit/Policies/UserPolicyImpersonateTest.php` - tests de política
- `tests/Unit/Transformers/UserListTransformerImpersonateTest.php` - tests de transformador
- `tests/Browser/Users/UserImpersonationTest.php` - tests de navegador

**Paquetes afectados**: Solo `basics13` (infraestructura neutra de autenticación y usuarios).

**Sin cambios en**: `entities13`, `projects13`, `entity-projects13`, `projects/demo13` (salvo que hereden la funcionalidad vía basics13).