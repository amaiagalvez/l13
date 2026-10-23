# Tasks

Rutas relativas a `packages/basics13/` salvo indicación. Comandos con `DX` en Docker (skill `docker-commands`).

## 0. Aprobación

- [ ] 0.1 Pedir aprobación para la carpeta nueva `src/Mail` (AGENTS.md); verificar la respuesta antes del grupo 4.

## 1. Configuración (`BAS_settings`)

- [ ] 1.1 Migración `BAS_settings` (columnas del design, `description_*` en `text`, unique `package`+`variable`) y enum `SettingType` (`integer`, `string`, `email`); verificar con `database-schema` tras `migrate`.
- [ ] 1.2 Modelo `Setting` con `get(package, variable, default)` sin caché, casteo según `type`, defecto si falta, está vacío o es < 1, etiqueta `name_{locale}` con caída a `name_en`, constantes `MIN_INTEGER`/`MAX_INTEGER`, y factory. Tests Unit del casteo y la etiqueta con `new Setting([...])`; Feature de `get()` (defecto y valor cambiado).
- [ ] 1.3 Migración de siembra idempotente de `max_login_attempts` (3), `lockout_minutes` (15) y `alert_email` (vacío) con textos en los 4 idiomas; test Feature: repetirla no duplica ni pisa un valor cambiado.

## 2. Permisos

- [ ] 2.1 Migración que siembra `sessions.view`, `sessions.manage`, `settings.view`, `settings.edit`, `settings.manage` y concede `sessions.manage` y `settings.manage` al rol `admin`; ampliar los módulos permitidos si `Permission` los valida; test Feature de siembra repetida y de los permisos del admin.
- [ ] 2.2 `SessionLogPolicy` y `SettingPolicy` con `ChecksPermissions`; tests Unit con `PermissionsActor` como `tests/Unit/Policies/RolePolicyTest.php`.

## 3. Registro de sesiones (`BAS_sessions`)

- [ ] 3.1 Migración `BAS_sessions` (columnas `string`, `ip` 45, `session_key` único, índices del design) y enums `SessionEvent` y `LoginFailureReason`; verificar con `database-schema`.
- [ ] 3.2 Modelo `SessionLog` inmutable (`UPDATED_AT = null`; solo `logged_out_at` de null a valor; nunca borrar), truncado de `login` y `user_agent`, y factory. Tests Feature: cambiar otro campo o borrar lanza y la fila no cambia; un login de 1000 caracteres se guarda truncado.
- [ ] 3.3 Clase `Auth\LoginAudit` con los listeners de `Login`, `Failed` y `Logout`, registrada en `FortifyServiceProvider`; `impersonate()`/`leaveImpersonation()` guardan `impersonator_id` antes y lo borran después de `Auth::login()`. Tests Feature en `tests/Feature/Auth` vía `POST login.store`: login correcto con `session_key` tras regenerar; contraseña errónea con `user_id`; login inexistente sin `user_id`; cuenta inactiva con motivo `inactive`; ninguna columna contiene la contraseña; logout fija solo su fila y un segundo logout no falla ni cambia nada; logout sin `session_key` (`actingAs`) no falla; restaurar por «recordarme» e impersonar o volver no crean filas ni reinician el contador.

## 4. Bloqueo, tope por IP y aviso

- [ ] 4.1 Bloqueo en `LoginAudit` con `lockoutKey()` único, `hit` en `Failed`, clave `login-locked|` con TTL completo al alcanzar el máximo, fila `lockout` única, rechazo `locked` en `authenticateUsing` sin alargar, `clear` en login correcto. Limitador `login` a `Limit::perMinute(10)->by(ip)`. Reescribir `test_login_is_rate_limited_after_five_failed_attempts` para el tope por IP. Tests Feature: bloqueo al 3.er fallo con fallos en minutos 0, 1 y 14 desde IPs distintas y rechazo en el minuto 28 con contraseña correcta (`travel()`); desbloqueo al pasar el tiempo; login inexistente también se bloquea; cambiar el máximo a 5 por `configuration.update` se aplica; bajar el máximo con un contador en curso escribe una sola fila `lockout`.
- [ ] 4.2 Aviso de pico en `LoginAudit` (método único para fallos y rechazos `locked`) y mailable síncrono `Mail\LoginFailureSpikeMail` (vista Markdown sin `{!! !!}`, textos en 4 idiomas, solo recuentos y enlace). Tests Feature con `Exceptions::fake()` y `Mail::fake()` y varios logins: 11 fallos → un report y un `assertSent`; 15 → sigue siendo uno; 10 → nada; fallos de hace más de 5 minutos no cuentan (`travel()`); tras 5 minutos puede haber otro; con la clave `basics13.login-alert` ya puesta → nada (carrera); rechazos `locked` cuentan; sin `alert_email` → report sin correo; el correo no contiene logins ni IPs.
- [ ] 4.3 Traducciones del mensaje de bloqueo (con minutos) en es/eu/fr/en; test de traducciones del paquete en verde.

## 5. Listado de sesiones

- [ ] 5.1 `SessionLogController@index`, `SessionLogListRequest`, `SessionLogListQuery` (rango de fechas sargable con el último día incluido, usuario con el selector remoto limitado, `login` buscable, resultado; orden `created_at desc, id desc`; `with('user')` con `withTrashed()`), `SessionLogListTransformer` sin acciones, ruta `sessions.index` en `routes/sessions.php` y vista `resources/views/sessions/list.blade.php` con la lista compartida, sin columna de acciones (skill `ux-design`). Tests Feature: filtros combinados incluido el último día, orden, 403 sin permiso, no existe ninguna ruta de escritura `sessions.*`, el `user_agent` con `<script>` sale escapado; test Unit del transformer con `setRelation`; `ArchitectureTest` en verde.

## 6. Pantalla de configuración

- [ ] 6.1 `SettingController@index/update`, `SettingIndexRequest` (paquete desconocido → 404), `UpdateSettingsRequest` con reglas según `type` (entero `between` con las constantes de `Setting`, correo `nullable|email`), guardado modelo a modelo, rutas `configuration.*` en `routes/configuration.php` y vista con `x-list.tabs` por paquete, formulario con `data-track-changes`, campos deshabilitados y sin botón sin `settings.edit`. Tests Feature: guardar válido; 0 o no entero conserva el valor anterior; `alert_email` vacío es válido; cambiar `type`/`name_*` se ignora; solo `settings.view` → 200 sin botón y 403 al guardar; 403 sin permisos; `ArchitectureTest` en verde.
- [ ] 6.2 Claves nuevas y específicas («Application configuration», «Sessions», filtros, resultados, motivos) en `resources/lang/{es,eu,fr,en}.json`, sin reutilizar «Settings», «Login» ni «Logout»; test de traducciones en verde.

## 7. Host y verificación

- [ ] 7.1 `projects/demo13`: entradas «Sesiones» (`sessions.view`) y «Configuración de la aplicación» (`settings.view`) en el grupo Administración, que se muestra también con solo esos permisos; `data-test` `nav-sessions` y `nav-configuration`. Test Feature del host: un usuario con solo `sessions.view` ve el grupo y su entrada.
- [ ] 7.2 Pint, PHPStan nivel 7 y suites de `basics13`, `entities13`, `projects13`, `entity-projects13` y `demo13` en verde, con cobertura ≥ 80 %.

## Workflow follow-up

- Archivar después de `add-roles-permissions-system`; coordinar con `add-superadmin-user-impersonation`, que edita el mismo requirement de permisos.
- Al archivar: añadir `basics-login-audit` y `basics-settings` al mapa de specs de `AGENTS.md` e "Invariantes" de las zone rules.
