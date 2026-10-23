## Context

Ver proposal.md. Estado actual en `basics13` (revisado en `.github/reviews/2026-10-23/CODE-REVIEW.md`):

- `FortifyServiceProvider::configureActiveUsers()` usa `Fortify::authenticateUsing` (solo usuarios
  `active`; un inactivo devuelve null y dispara `Failed`) y un listener de `Login` que expulsa a un
  inactivo que entra por 2FA, passkey o «recordarme», lanzando `ValidationException` sin `Failed`.
  Con `authenticateUsing`, `Failed` llega siempre con `user = null` y con el password en las
  credenciales.
- `configureRateLimiting()` define el limitador `login` (5/min por login + IP). Fortify usa ese
  limitador nombrado porque `config/fortify.php` del host lo referencia;
  `tests/Feature/Auth/AuthenticationTest.php:43-60` espera el 429.
- `Livewire\Actions\Logout` y el `destroy` de Fortify llaman a `logout()` (que dispara `Logout`)
  antes de invalidar la sesión.
- `UserController::impersonate()` hace `Auth::login()` y después guarda `impersonator_id`;
  `leaveImpersonation()` lo borra antes de `Auth::login()`. Ambos disparan `Login`.
- `SessionGuard` también dispara `Login` al restaurar la sesión con la cookie «recordarme».
- `RateLimiter::increment` crea el temporizador solo en el primer hit (`Cache/RateLimiter.php:165-167`).
- Host: `CACHE_STORE=database` con `serializable_classes=false`; sin worker de cola en
  docker-compose; sin Sentry. Los tests del paquete usan SQLite en memoria y el host MariaDB.
- Flux free: no hay `flux:tab`; `basics13` ya tiene `x-list.tabs`.

## Goals / Non-Goals

**Goals:** registro inalterable a nivel de aplicación, bloqueo configurable, aviso deduplicado,
configuración por paquete sin acoplar `basics13` a los dominios.

**Non-Goals:**
- Registrar fallos del reto 2FA o de passkeys (no disparan `Failed`). El login por passkey no pasa
  por el bloqueo: tiene su propio limitador (10/min) y sirve de vía de recuperación.
- Apuntar el fin de sesiones que caducan, que se cierran desde «otros dispositivos» o al cambiar la
  contraseña.
- Purga o retención del registro (una purga futura usará el query builder, que no pasa por el modelo).
- Inmutabilidad a nivel de base de datos (triggers).
- Desbloqueo manual: el bloqueo caduca solo; un operador puede hacer `Cache::forget` de la clave.
- Exportar el listado (lo cubrirá `download-tables-to-excel` si se aplica a este listado).
- Geolocalizar la IP.

## Decisions

**Modelo `SessionLog` sobre `BAS_sessions`.** El nombre `Session` choca con la fachada y la tabla
`sessions` del driver. Columnas `string` con cast a enums PHP (como el resto de tablas BAS, sin enums
nativos): `event` (`login`, `failed`, `lockout`), `failure_reason` nullable (`unknown_user`,
`wrong_password`, `inactive`, `locked`); `user_id` nullable (FK `nullOnDelete`: tras un borrado
definitivo del usuario queda nulo y `login` conserva la identidad); `login` (truncado a la longitud
de la columna, porque Fortify no limita su longitud y MariaDB estricto fallaría), `ip` (45),
`user_agent` (truncado a 512), `lockout_minutes` nullable, `session_key` (uuid, nullable, único),
`created_at` y `logged_out_at` nullable. `UPDATED_AT = null`. Sin `TracksAuditColumns`. Índices
`(created_at)`, `(user_id, created_at)`, `(event, created_at)`.

**Inmutabilidad en el modelo.** `updating` lanza salvo que el único cambio sea `logged_out_at` de
null a un valor; `deleting` lanza siempre. Toda escritura de la aplicación pasa por el modelo, salvo
el logout (abajo), que es condicional y no puede reescribir. Sin rutas de escritura. Alternativa
descartada: triggers por motor (SQLite/MariaDB), demasiada superficie para el beneficio.

**Una clase `Basics13\Auth\LoginAudit`.** Concentra los listeners de `Login`, `Failed` y `Logout`, la
comprobación de bloqueo y el aviso, con un único `lockoutKey()` (login con
`Str::transliterate(Str::lower())`, sin IP, a diferencia del limitador actual).
`FortifyServiceProvider` solo la registra. No es una capa nueva: una clase concreta en una carpeta
existente, para no repartir la regla en cuatro closures.

**Enlazar logout con su login mediante un uuid en sesión.** El listener de `Login` crea la fila con
un `session_key` y lo guarda en la sesión; la regeneración de Fortify migra los datos. El de
`Logout` hace `SessionLog::where('session_key', $k)->whereNull('logged_out_at')->update(...)`:
atómico e idempotente, así que un segundo logout (dos pestañas) no hace nada ni da 500. Alternativa
descartada: el id de sesión de Laravel, que cambia tras el login.

**Qué `Login` se registra.** Se ignoran `viaRemember()` (restauración por cookie), la impersonación
(sesión con `impersonator_id`) y los usuarios inactivos (los expulsa el listener existente; no
depender del orden de registro). Para que la impersonación sea detectable, `impersonate()` guarda
`impersonator_id` antes de `Auth::login()` y `leaveImpersonation()` lo borra después; la
regeneración del guard conserva los datos de sesión. Esto modifica `basics-user-impersonation`, que
pedía guardar la marca después del login. En los casos ignorados tampoco se hace `clear()` del
contador.

**Motivo del fallo.** El listener de `Failed` lee solo el campo de login (nunca `password`) y busca la
cuenta: inexistente → `unknown_user`; existe e inactiva → `inactive`; si no → `wrong_password`.

**Bloqueo con clave propia.** `RateLimiter::hit()` arranca su temporizador en el primer fallo, así que
no sirve como duración del bloqueo. En `Failed`: `hit(lockoutKey, lockout_minutes*60)`; si devuelve
`>= max_login_attempts`, `Cache::add('login-locked|'.$key, $hasta, lockout_minutes*60)` y `clear()`
del contador; la fila `lockout` solo se escribe si `add` gana (una sola fila con concurrencia o si se
baja el máximo). `authenticateUsing` comprueba primero `login-locked|` y, si existe, registra un fallo
`locked` y lanza `ValidationException` con los minutos restantes, sin `hit()` ni alargar el bloqueo.
En login correcto (no ignorado): `clear()`. Alternativa descartada: contar filas `failed`.

**Tope por IP.** El limitador `login` pasa a `Limit::perMinute(10)->by($request->ip())`: frena el
*password spraying* desde una IP, que el bloqueo por login no ve. El test del 429 se reescribe para
este tope.

**Aviso de pico.** Un único método privado registra cada fallo (los de `Failed` y los `locked`) y
después comprueba el pico: si no existe `basics13.login-alert` en caché y hay más de 10 fallos en los
últimos 5 minutos (`skip(10)->exists()` sobre `(event, created_at)`), y
`Cache::add('basics13.login-alert', true, 300)` gana, entonces `report(new RuntimeException(...))` con
recuentos en el mensaje y, si `alert_email` no está vacío, `Mail::to()->send(new LoginFailureSpikeMail(...))`
síncrono dentro de `try`/`report()`: es un correo cada 5 minutos como mucho y no hay worker. Ni el
mensaje ni el correo llevan logins o IPs: número de fallos, de logins y de IPs distintos, ventana y
enlace a `sessions.index`. Umbrales (10, 5 min) como constantes de `LoginAudit`; el TTL se deriva de
los minutos. Sin clase de excepción propia; el mailable va en la carpeta nueva `src/Mail` (aprobada).

**Settings sin caché.** Modelo `Setting` sobre `BAS_settings` con `package`, `class` (informativa),
`variable`, `type` (`string` con cast a enum `integer`, `string`, `email`), `value` (text nullable),
`name_{eu,es,fr,en}` (string), `description_{eu,es,fr,en}` (text), timestamps y unique
`(package, variable)`. `type` hace falta para validar y castear. Lectura con
`Setting::get('basics13', 'max_login_attempts', 3)`: una consulta indexada, despreciable frente al
hash, sin caché (la caché del host no deserializa modelos y las siembras no la invalidarían). Si el
valor falta, está vacío o un entero queda fuera de `[MIN_INTEGER, MAX_INTEGER]`, devuelve el
defecto. Siembra por migración con
`insertOrIgnore` (no pisa valores cambiados). Las etiquetas usan `name_{locale}` y, si está vacío,
`name_en`.

**Pantalla de configuración.** No es un recurso canónico (sin alta, borrado, papelera ni archivado):
`SettingController@index` con `SettingIndexRequest` (`package` válido; uno desconocido da 404) y
pestañas `x-list.tabs` con el paquete traducido; `@update` por paquete con `UpdateSettingsRequest`,
que construye las reglas según `type`: entero `between` con las constantes `Setting::MIN_INTEGER` (1)
y `MAX_INTEGER` (1440), correo `nullable|email` y texto `nullable|string`, ambos con el `max` de
configuración. Guarda modelo a modelo.
Formulario con `data-track-changes`; sin `settings.edit`, campos deshabilitados y sin botón de
guardar. Ruta `/configuration` (`configuration.*`), porque `settings/*` ya son los ajustes de cuenta.

**Listado de sesiones.** `SessionLogController@index` con `SessionLogListRequest`, `ListQueryBase`,
transformer y la lista compartida, sin columna de acciones; ruta `/sessions`, `sessions.index`.
Filtro de fechas sargable en la query (`>= desde` y `< hasta + 1 día`, no `whereDate`), orden
`created_at desc, id desc`, `login` en la búsqueda y `with('user')` con `withTrashed()`. El usuario se
filtra con un `select` estático de la búsqueda avanzada, como `role_ids` del resumen de permisos:
todas las cuentas (archivadas, en papelera y el super-admin incluidos), por nombre. Alternativa
descartada: el selector remoto (`users.options`), que solo existe en el formulario de épicas, excluye
cuentas inactivas y el super-admin, y el modal de búsqueda avanzada no admite opciones remotas.

**Permisos.** Migración nueva que siembra `sessions.view`, `sessions.manage`, `settings.view`,
`settings.edit` y `settings.manage`, y concede `sessions.manage` y `settings.manage` al rol `admin`.
`SessionLogPolicy` y `SettingPolicy` usan `ChecksPermissions` como las existentes.

**Sidebar del host.** El grupo Administración se muestra también con `sessions.view` o
`settings.view`. Claves de traducción nuevas y específicas («Application configuration», «Sessions»,
eventos y motivos): «Settings», «Login» y «Logout» ya existen con otro sentido.

## Risks / Trade-offs

- [Bloquear por login permite a un atacante bloquear cuentas ajenas, también la del super-admin] →
  Es el criterio elegido; el bloqueo caduca, passkeys siguen entrando, el tope por IP lo encarece y
  el aviso de pico lo hace visible.
- [Escribir una fila por intento] → Coste despreciable frente al hash; el tope por IP limita el volumen.
- [La tabla crece sin límite] → Fuera de alcance; los índices por fecha permiten purgar luego.
- [Sin Sentry, `report()` solo escribe en el log diario] → Hasta que el host lo instale; el correo
  es el canal visible.
- [Detrás de un proxy, `ip()` sería la del proxy] → Hoy no hay proxy; si se añade, configurar
  `trustProxies` solo con sus IPs.
- [Un `settings.edit` puede redirigir `alert_email`] → El correo no lleva datos personales.

## Migration Plan

Migraciones aditivas en `basics13`; rollback con `migrate:rollback` en orden inverso (las
siembras de settings y permisos no deshacen filas; las tablas se borran con su migración). Al
desplegar, el límite de 5/min por login + IP pasa a 10/min por IP más 3 fallos → 15 minutos por
login (**BREAKING** de comportamiento).
