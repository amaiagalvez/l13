## 1. Policy y modelo
- [x] 1.1 Crear `Basics13\Policies\UserPolicy` (viewAny, create, update, delete, restore, forceDelete, archive, activate; sin roles)
- [x] 1.2 Registrar en el `ServiceProvider` `Gate::policy(config('auth.providers.users.model'), UserPolicy::class)`; sin `#[UsePolicy]` en el modelo (lo exige `tests/Unit/ArchitectureTest`, porque el host puede sustituirlo)

## 2. Form Requests
- [x] 2.1 `UserRequest` reutilizando `ProfileValidationRules` (nombre, correo, notas) y `PasswordValidationRules` (contraseña obligatoria en alta y opcional en edición, confirmada), con `authorize()` delegando en la policy
- [x] 2.2 `UserListRequest`, `UserSelectOptionsRequest`
- [x] 2.3 `UserRestoreRequest`, `UserTrashDestroyRequest` resolviendo el registro con el modelo de `auth.providers.users.model` (no con `Basics13\Models\User`)

## 3. Queries
- [x] 3.1 `Queries\Users\UserListQuery` (active/archived/trashed + búsqueda) sobre el modelo resuelto de `auth.providers.users.model`
- [x] 3.2 `Queries\Users\UserSelectOptionsQuery` (solo activos, mismo tope que la paginación)

## 4. Transformer
- [x] 4.1 `UserListTransformer` (columnas nombre/correo; `editPayload` sin contraseña; rutas `users.*`)

## 5. Controllers
- [x] 5.1 `UserController` (index, selectOptions, store, update, destroy) extendiendo `Basics13\Http\Controllers\Controller`, con `UniqueConstraintViolation::rethrowAsValidationError()`
- [x] 5.2 `UserTrashController` extendiendo `TrashController` (pre-check adaptado a una identidad por correo)
- [x] 5.3 `UserArchivedController` extendiendo `ArchivedController`

## 6. Vistas y rutas
- [x] 6.1 `resources/views/users/{list,form}.blade.php` reutilizando `x-basics13::*` (aplicar skill `ux-design`)
- [x] 6.2 `routes/users.php` (`users.index|store|update|destroy`, `users.options`, `users.archived.*`, `users.trash.*`) dentro del grupo `['web','auth','verified']`, como customers/projects, cargado desde `ServiceProvider::boot()`
- [x] 6.3 Enlazar `{user}` con `Route::model('user', config('auth.providers.users.model'))` (CRUD y archivado) y usar `{trashedUser}` en las rutas de papelera, leído por sus requests, porque el binding por defecto excluye los registros en la papelera

## 7. i18n
- [x] 7.1 Claves nuevas en `resources/lang/{en,es,eu,fr}.json` con paridad verificada

## 8. Tests
- [x] 8.1 `UserCrudTest`, `UserTrashTest`, `UserArchivedTest`, `UserSelectOptionsTest` extendiendo `Basics13\Tests\Feature\FeatureTestCase`
- [x] 8.2 Carrera de correo duplicado en alta y edición
- [x] 8.3 Caso de modelo configurado por el host (`auth.providers.users.model`): CRUD autorizado y binding resueltos

## 9. Documentación
- [x] 9.1 Corregir `openspec/specs/package-boundaries/design.md`: `User` pertenece a `basics13`, no a `projects/demo13`

## 10. Cierre
- [x] 10.1 Pint + PHPStan al nivel configurado del paquete (7, el mismo que el resto de paquetes y que `projects/demo13`) + suite del paquete
- [x] 10.2 `openspec validate add-user-crud --strict` en verde
