# Design

## Context

Ver `proposal.md` - Why. El paquete `basics13` ya provee gestión de usuarios (listar, crear, editar, archivar, papelera) con políticas, transformadores y vistas compartidas. El modelo `User` tiene campo `is_super_admin` y el trait `HasRoles` expone `isSuperAdmin()`. Las traducciones para "impersonate" ya existen en los 4 locales. La arquitectura usa `ListTransformer` base para construir acciones de fila.

## Goals / Non-Goals

**Goals:**
- Añadir endpoint `POST users/{user}/impersonate` que inicie impersonación guardando `impersonator_id` en sesión
- Añadir endpoint `POST impersonation/leave` que restaure el usuario original
- Añadir habilidad `impersonate` en `UserPolicy` (solo superadmin, no sobre sí mismo ni otro superadmin)
- Añadir acción `impersonateAction` en `UserListTransformer` visible solo para superadmin en lista activa
- Registrar permiso `users.impersonate` en seeder/migración y asignarlo al superadmin
- Tests unitarios, feature y browser; i18n en 4 locales

**Non-Goals:**
- Impersonación de superadmin a superadmin
- Impersonación desde lista archivados/papelera
- Historial de impersonaciones (auditoría) — fuera de alcance inicial
- UI de "banner" indicando impersonación activa — se deja para follow-up

## Decisions

### 1. Controlador: métodos en `UserController` vs controlador dedicado
**Decisión**: Añadir métodos `impersonate(Model $user)` y `leaveImpersonation()` en `UserController`.
**Racional**: La impersonación es una acción sobre el recurso `User`; mantenerla en el mismo controlador evita fragmentación. `leaveImpersonation` no recibe modelo, pero encaja en el mismo archivo.
**Alternativa**: `UserImpersonationController` dedicado — más limpio para responsabilidad única, pero añade archivo y ruta extra sin beneficio claro.

### 2. Guardar identidad original: sesión vs cookie firmada vs JWT
**Decisión**: Guardar `impersonator_id` en `session()` (Laravel session).
**Racional**: La sesión ya está autenticada, cifrada y firmada; `Auth::onceUsingId()` o `Auth::login()` restauran limpiamente. No requiere infraestructura extra.
**Alternativa**: Cookie firmada separada — útil si la sesión se pierde, pero añade complejidad y superficie de ataque.

### 3. Cambio de usuario: `Auth::login()` vs `session()->put('login_web_...')`
**Decisión**: Usar `Auth::login($targetUser)` tras guardar `auth()->id()` en `session('impersonator_id')`.
**Racional**: `Auth::login()` regenera sesión, setea guard, dispara eventos `Login` — comportamiento estándar. Al salir, `Auth::login($originalUser)` restaura.

### 4. Permiso `users.impersonate`: seeder vs migración vs config
**Decisión**: Añadir permiso en `database/seeders/PermissionSeeder.php` (o equivalente) y asignarlo al rol "superadmin" si existe, o documentar que solo se concede a cuentas con `is_super_admin = true` vía política.
**Racional**: El permiso vive en BD; el seeder es idempotente. La política ya verifica `isSuperAdmin()` antes del permiso, así que el permiso es capa extra, no sustituto.

### 5. Acción en transformador: método nuevo vs sobrescribir `active()`
**Decisión**: Añadir `protected function impersonateAction(Model $record): array` en `UserListTransformer` y llamarlo desde `active()` sobrescrito localmente.
**Racional**: `ListTransformer::active()` es `final protected` en la base — no se puede sobrescribir. Solución: `UserListTransformer` define su propio `active()` copiando la base y añadiendo la acción, o mejor: añadir `integrationRow()` que devuelve acciones extra y que `ListTransformer::active()` incluya. **Pero** `integrationRow()` devuelve `array<string, mixed>` para columnas, no acciones.
**Decisión final**: Sobrescribir `active()` en `UserListTransformer` copiando la implementación base y añadiendo la acción condicional. Es la única forma sin tocar la clase base compartida.

### 6. Rutas: nombres y middleware
**Decisión**:
- `POST users/{user}/impersonate` → `users.impersonate` (middleware: `web`, `auth`, `verified`)
- `POST impersonation/leave` → `impersonation.leave` (middleware: `web`, `auth`)
**Racional**: Seguir convención `users.{action}` para acciones sobre recurso. `leave` es global, no sobre usuario.

### 7. Validación en Form Request vs Policy
**Decisión**: Validaciones de autorización en `UserPolicy::impersonate()`; validación de existencia del usuario en route model binding. No se necesita Form Request dedicado (sin cuerpo de petición).

## Risks / Trade-offs

- [Risk] Sobrescribir `active()` en `UserListTransformer` duplica código de la base → Mitigación: mantenerlo simple, solo añadir la acción; si la base cambia, actualizar en follow-up
- [Risk] Sesión perdida durante impersonación deja al usuario "atrapado" → Mitigación: `leaveImpersonation` verifica `session('impersonator_id')` y falla seguro; añadir middleware que limpie `impersonator_id` si el usuario original ya no existe
- [Risk] Eventos `Login` disparados en impersonación pueden activar listeners no deseados (ej. notificaciones) → Mitigación: pasar `remember: false` y considerar `Auth::setUser()` sin eventos, pero `Auth::login()` es más seguro para regenerar sesión
- [Risk] Superadmin impersonado crea registros → `created_by` queda como usuario objetivo, `updated_by` como superadmin original (según spec) → Implementar en `TracksAuditColumns` leyendo `session('impersonator_id')`

## Migration Plan

1. Ejecutar seeder de permisos para crear `users.impersonate`
2. Desplegar código (controlador, política, transformador, rutas)
3. Verificar en staging: superadmin ve botón, puede impersonar y volver
4. Rollback: revertir código + eliminar permiso de BD si necesario

## Open Questions

- ¿Banner visual indicando impersonación activa? (Dejado para follow-up)
- ¿Auditoría de sesiones de impersonación? (Fuera de alcance)
- ¿Límite de tiempo de impersonación? (No en v1)