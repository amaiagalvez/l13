# Tasks

## 1. Permiso y seeder

- [x] 1.1 Añadir permiso `users.impersonate` en `database/seeders/PermissionSeeder.php` (o equivalente) y verificar que `php artisan db:seed --class=PermissionSeeder` lo crea en BD
- [x] 1.2 Si existe rol "superadmin", asignarle el permiso en el seeder; si no, documentar que la política usa `isSuperAdmin()` y verificar que el test de política pasa sin rol

## 2. Rutas

- [x] 2.1 Añadir en `routes/users.php`:
  - `Route::post('users/{user}/impersonate', [UserController::class, 'impersonate'])->whereNumber('user')->name('users.impersonate');`
  - `Route::post('impersonation/leave', [UserController::class, 'leaveImpersonation'])->name('impersonation.leave');`
- [x] 2.2 Verificar `php artisan route:list --name=users.impersonate` y `--name=impersonation.leave` muestran las rutas

## 3. Controlador

- [x] 3.1 Añadir método `impersonate(Model $user)` en `UserController`:
  - Autorizar con `$this->authorize('impersonate', $user);`
  - Guardar `$originalId = auth()->id();`
  - `Auth::login($user);`
  - `session(['impersonator_id' => $originalId]);`  // DESPUÉS del login (regenera sesión)
  - Redirigir a `route('dashboard')` o `route('home')` con mensaje de éxito
- [x] 3.2 Añadir método `leaveImpersonation()` en `UserController`:
  - Verificar `session()->has('impersonator_id')` si no, abortar 400
  - Recuperar `$originalId = session('impersonator_id');`
  - `$originalUser = $this->userModelClass()::findOrFail($originalId);`
  - `session()->forget('impersonator_id');`
  - `Auth::login($originalUser);`
  - Redirigir a `route('users.index')` con mensaje
- [x] 3.3 Verificar que los métodos compilan y responden (test feature básico)

## 4. Política

- [x] 4.1 Añadir método `impersonate(User $user, Model $record): bool` en `UserPolicy`:
  - `return $this->isSuperAdmin($user) && !$record->isSuperAdmin() && $record->getKey() !== $user->getAuthIdentifier();`
- [x] 4.2 Verificar test unitario `UserPolicyImpersonateTest` cubre: superadmin→usuario normal (true), superadmin→sí mismo (false), superadmin→otro superadmin (false), no superadmin (false)

## 5. Transformador

- [x] 5.1 Añadir método `protected function impersonateAction(Model $record): array` en `UserListTransformer` que devuelve acción tipo `link` o `confirm-modal` hacia `route('users.impersonate', $record)`
- [x] 5.2 Sobrescribir `active()` en `UserListTransformer` copiando implementación base y añadiendo la acción condicional:
  - Solo si `auth()->user()?->isSuperAdmin()`
  - Solo si `!$record->isSuperAdmin()`
  - Solo si `$record->getKey() !== auth()->id()`
- [x] 5.3 Verificar test unitario `UserListTransformerImpersonateTest` cubre presencia/ausencia de acción según usuario actual y objetivo

## 6. Tests Unitarios

- [x] 6.1 `tests/Unit/Policies/UserPolicyImpersonateTest.php` — 4 casos (ver 4.2)
- [x] 6.2 `tests/Unit/Transformers/UserListTransformerImpersonateTest.php` — acción aparece/solo superadmin/solo usuario normal/no propia/no superadmin

## 7. Tests Feature

- [x] 7.1 `tests/Feature/Users/UserImpersonationTest.php`:
  - Superadmin puede impersonar usuario normal → sesión cambia, `impersonator_id` guardado
  - Superadmin NO puede impersonar a sí mismo → 403
  - Superadmin NO puede impersonar a otro superadmin → 403
  - Usuario normal NO puede impersonar → 403
  - Impersonado puede salir → vuelve a superadmin original
  - Salir sin impersonación activa → 400/redirect
  - Impersonado crea registro → `created_by` = usuario objetivo, `updated_by` = superadmin original

## 8. Tests Browser (Dusk)

- [x] 8.1 `tests/Browser/Users/UserImpersonationTest.php`:
  - Superadmin logueado ve botón "Impersonate" en filas de usuarios normales
  - Superadmin NO ve botón en su propia fila ni en otros superadmins
  - Click en botón → confirma modal → redirige a dashboard como usuario objetivo
  - Usuario impersonado ve botón "Leave impersonation" (o similar) → click → vuelve a lista usuarios como superadmin

## 9. i18n (claves ya existen en actions.php)

- [x] 9.1 Verificar que `__('actions.impersonate')` y `__('actions.impersonation')` existen en `resources/lang/{en,es,eu,fr}/actions.php`
- [x] 9.2 Añadir claves de confirmación si faltan: `messages.impersonate_confirm_title`, `messages.impersonate_confirm_text`, `messages.impersonation_started`, `messages.impersonation_ended`, `messages.no_active_impersonation` en 4 locales (messages.php)

## 10. Auditoría (TracksAuditColumns)

- [x] 10.1 Revisar `TracksAuditColumns` concern: si usa `auth()->id()` para `updated_by`, cambiar a `session('impersonator_id') ?? auth()->id()` para que la auditoría refleje al superadmin original durante impersonación
- [x] 10.2 Verificar con test feature que `updated_by` = superadmin original al editar durante impersonación

## 11. Verificación final

- [x] 11.1 Ejecutar suite de tests de `basics13`: `DX php artisan test --filter=basics13` (o equivalente en Docker)
- [x] 11.2 Ejecutar PHPStan nivel 7: `DX ./vendor/bin/phpstan analyse packages/basics13/src` (errores pre-existentes en ExcelExportService, sin nuevos errores)
- [x] 11.3 Ejecutar Pint: `DX ./vendor/bin/pint --test packages/basics13`
- [x] 11.4 Verificar cobertura ≥ 80% en paquetes afectados

## Workflow follow-up

- Archivar el change después de merge en su propio PR
- Verificar presupuesto de specs afectadas tras archivar