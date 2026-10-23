# Code Review — change `add-session-log-and-settings`

Objeto: artefactos de planificación de `openspec/changes/add-session-log-and-settings/` (proposal, design, tasks y 3 specs delta), contrastados con el código real de `packages/basics13` y `projects/demo13`. Aún no hay código del change. Rutas de artefactos relativas a `openspec/changes/add-session-log-and-settings/`.

## Executive Summary

El plan cubre toda la petición y es proporcional (sin capas especulativas), pero **no está listo para aplicar**. Tres defectos rompen el comportamiento o la suite si se implementa tal cual:

1. El bloqueo no dura `lockout_minutes`: `RateLimiter` fija el temporizador en el primer fallo (BUG-001).
2. `Limit::none()` elimina todo límite por IP en `POST /login` y rompe un test existente (SEC-001).
3. La migración de permisos concede `settings.manage` sin sembrarlo y fallaría (BUG-002).

Le siguen ocho problemas MEDIUM: caché de settings incompatible con `serializable_classes=false`, aviso de pico que no ve los intentos bloqueados, correo en cola sin worker, test de arquitectura que exige FormRequests, límites de settings, logout doble con 500, filas espurias por remember-me e impersonación, y PII en el aviso.

Revisores: Laravel, Security, Database, Performance, Business Logic, Architecture, Testing, Frontend, Maintainability y DevOps. API omitido: el change no expone API. Después, Devil's Advocate. Informes brutos en este directorio.

## Detected Stack

Laravel 13 · PHP 8.5 · Fortify · Livewire 4 · Flux free (sin `flux:tab` ni date-picker) · Tailwind 4 · PHPUnit + Dusk · MariaDB (host) / SQLite en memoria (tests de paquete, `packages/basics13/phpunit.xml:35-36`) · `CACHE_STORE=database` con `serializable_classes=false` · `QUEUE_CONNECTION=database` sin worker en docker-compose · `MAIL_MAILER=log` · sin Sentry · sin `trustProxies`.

## Checks Executed

| Check | Resultado |
|---|---|
| `openspec validate add-session-log-and-settings --strict` | PASS. INFO: el archivado rechazaría el delta `basics-roles-permissions` hasta que exista su spec principal (archivar antes `add-roles-permissions-system`). |
| Verificación manual de `RateLimiter.php:128-167`, `AuthenticationTest.php:43-60`, `config/cache.php:134`, seed `000006:16-63`, `sidebar.blade.php:59`, `LoginRequest.php:28`, `ChecksResourceArchitecture.php:129-155` | Confirmados |
| PHPUnit / PHPStan / Pint / Dusk | NOT RUN: no hay código del change que verificar |

## Critical Findings

Ninguno.

## High Findings

### BUG-001 — El bloqueo dura lo que quede de la ventana del primer fallo, no `lockout_minutes`

Severity: HIGH · Category: Bug · File: design.md · Line: 52-57 (spec basics-login-audit:48-55, tasks.md:24) · Confidence: HIGH

Problem: el plan llama a `hit($key, lockout_minutes*60)` en cada fallo y considera bloqueado el login mientras exista el temporizador.

Evidence: `vendor/laravel/framework/src/Illuminate/Cache/RateLimiter.php:165-167` crea `:timer` con `cache->add` solo en el primer hit. `tooManyAttempts` (128-132) depende de ese temporizador.

Impact: si los fallos llegan en t=0, t=1 y t=14 min, el bloqueo dura unos 1 min, pero la fila `lockout` registra 15. El mensaje de "minutos restantes" también es erróneo. Si `max_login_attempts` se baja, o hay concurrencia, `== max` puede saltarse.

Recommendation:
- Al llegar a `hit() >= max`, hacer `Cache::add('login-locked|'.$key, …, $m*60)` y `clear()` del contador.
- Escribir la fila `lockout` solo si `add` gana; `authenticateUsing` comprueba esa clave.
- Declarar en la spec la ventana de recuento y añadir un test con fallos espaciados (`travel()`).

Fuentes: LAR-001, BUS-001, DB-002, SEC-002, BUS-010.

### SEC-001 — `Limit::none()` deja `POST /login` sin límite por IP

Severity: HIGH · Category: Security · File: design.md · Line: 57-58 (proposal.md:26,30; tasks.md:24) · Confidence: HIGH

Problem: el bloqueo nuevo solo cuenta por login. Al anular el limitador `login` desaparece todo freno por petición.

Evidence: `FortifyServiceProvider.php:229-235` define hoy 5/min por login + IP. Fortify usa ese limitador nombrado en lugar del suyo (`AuthenticatedSessionController.php:86`, `config/fortify.php:118`). Además, `packages/basics13/tests/Feature/Auth/AuthenticationTest.php:43-60` espera un 429 tras 5 fallos y ninguna tarea lo sustituye.

Impact:
- Una IP puede hacer *password spraying* ilimitado (2 intentos por cuenta, sin bloqueo).
- Cada intento suma bcrypt, un INSERT y un COUNT.
- La suite de basics13 queda en rojo.

Recommendation:
- Mantener el limitador `login` solo por IP (p. ej. `Limit::perMinute(10)->by($request->ip())`) junto al bloqueo por login.
- Añadir a 4.1 la reescritura del test del 429.

Fuentes: OPS-001, SEC-003, LAR-003, ARC-003, BUS-004, TST-001.

### BUG-002 — La migración de permisos concede `settings.manage` sin sembrarlo

Severity: HIGH · Category: Bug · File: design.md · Line: 84-85 (tasks.md:13) · Confidence: HIGH

Problem: solo se siembran `sessions.view`, `settings.view` y `settings.edit`, pero se concede `settings.manage`.

Evidence: el patrón de `2026_10_22_000006_seed_user_and_role_permissions.php:61` indexa `$permissionIds[$permission]` sin comprobar que exista.

Impact: la migración falla, y con ella toda suite con `RefreshDatabase`.

Recommendation: sembrar `MODULES × ACTIONS` para `sessions` y `settings`, igual que en 000006.

Fuentes: DB-001.

## Medium Findings

### BUG-003 — Cachear modelos `Setting` se rompe con `serializable_classes=false`

Severity: MEDIUM · Category: Bug · File: design.md · Line: 72-75 · Confidence: HIGH

Problem: el plan cachea "para siempre" la configuración por paquete.

Evidence: `projects/demo13/config/cache.php:134` (`serializable_classes => false`) con el store `database`. Los tests usan el store `array`, que no serializa y oculta el fallo. Las siembras con `insertOrIgnore` no disparan `saved` y no invalidan la caché.

Impact: en producción `Setting::get` devolvería objetos incompletos o el valor por defecto, y el bloqueo configurable dejaría de funcionar.

Recommendation: eliminar la caché. Una consulta indexada por intento es despreciable frente a bcrypt.

Fuentes: LAR-002, OPS-004, DB-008, MNT-007, ARC-004, LAR-006.

### BUG-004 — El aviso de pico no ve los rechazos durante un bloqueo

Severity: MEDIUM · Category: Bug · File: design.md · Line: 49-50, 62 (spec basics-login-audit:61-69) · Confidence: HIGH

Problem: los rechazos por bloqueo se escriben en `authenticateUsing`, que lanza la excepción antes de que Fortify dispare `Failed`. La comprobación del pico solo corre en el listener de `Failed`. No está definido el `event` de esas filas ni si cuentan para el pico.

Impact: la fuerza bruta contra una sola cuenta (3 fallos y después N rechazos) nunca dispara el aviso. Los tests de 4.2 con un solo login no llegan a 11 fallos.

Recommendation:
- Un único método privado escribe la fila de fallo y comprueba el pico, llamado desde ambos caminos.
- Las filas `locked` son `event=failed`, cuentan para el pico y no hacen `hit()`.
- Los tests usan varios logins.

Fuentes: BUS-003, TST-003.

### OPS-001 — El correo de aviso va en cola y no hay worker

Severity: MEDIUM · Category: Production · File: design.md · Line: 65 (tasks.md:25) · Confidence: HIGH

Evidence: `projects/demo13/docker-compose.yml` no tiene servicio `queue:work` (solo `composer dev` usa `queue:listen`). `.env.example:43` fija `QUEUE_CONNECTION=database`.

Impact: el aviso se queda en `jobs` y nunca sale.

Recommendation: `Mail::send` síncrono, que ya está deduplicado a uno cada 5 min, envuelto en `try`/`report()`.

### TST-001 — El test de arquitectura exige FormRequest en los `index` nuevos

Severity: MEDIUM · Category: Testing · File: tasks.md · Line: 30, 34 · Confidence: HIGH

Evidence: `packages/basics13/tests/ChecksResourceArchitecture.php:129-155` exige FormRequest en `index`, `store`, `update` y `restore`, y prohíbe `$request->input()`. `SessionLogController@index` lee filtros y `SettingController@index` lee `?package=`.

Impact: `ArchitectureTest` en rojo. Además, la vista de correo no puede usar `{!! !!}` (`ArchitectureTest.php:38-60`).

Recommendation: añadir `SessionLogListRequest` y la request de `index` de settings a 5.1 y 6.1.

### SEC-002 — Valores de settings sin límites

Severity: MEDIUM · Category: Security · File: design.md · Line: 69-79 (spec basics-settings:24-26) · Confidence: HIGH

Problem: validar solo por tipo deja pasar valores que rompen el login.

Impact:
- `lockout_minutes=0` desactiva el bloqueo.
- `max_login_attempts` a 0 o nulo (casteado a 0) bloquea a todo el mundo, incluso con la contraseña correcta.
- Un valor enorme bloquea logins de forma prácticamente permanente.
- `alert_email` sembrado vacío debe seguir siendo válido.

Recommendation:
- `integer` → `required|integer|min:1|max:1440`; `email` → `nullable|email`.
- Al leer, usar el valor por defecto si el guardado es nulo, vacío o menor que 1.

Fuentes: SEC-006, BUS-008, DB-009, FE-003.

### BUG-005 — Un logout doble da 500

Severity: MEDIUM · Category: Bug · File: design.md · Line: 37-44 · Confidence: HIGH

Problem: con buscar-y-guardar, un segundo logout de la misma sesión (dos pestañas o doble envío) intenta volver a fijar `logged_out_at` y el guard `updating` lanza excepción.

Recommendation: en el listener de logout, `SessionLog::where('session_key', $k)->whereNull('logged_out_at')->update([...])`, que es atómico e idempotente. Índice único en `session_key`.

Fuentes: DB-003.

### BUG-006 — `Login` también se dispara con remember-me y con la impersonación

Severity: MEDIUM · Category: Bug · File: design.md · Line: 41-46 · Confidence: HIGH

Evidence:
- `vendor/laravel/framework/src/Illuminate/Auth/SessionGuard.php:196-202` dispara `Login` al restaurar desde la cookie remember-me.
- `add-superadmin-user-impersonation` (design.md:37, tasks.md:20,28) usa `Auth::login()`.

Impact: filas `login` que no vienen del formulario, atribuidas al usuario impersonado y desde la IP del admin. Se sobrescribe `session_key`, y el admin puede desbloquear sin querer la cuenta que impersona.

Recommendation:
- Ignorar `viaRemember()` y la impersonación en el listener (no registrar fila ni hacer `clear()`).
- Añadir escenario y test, y coordinar con el change de impersonación.

Fuentes: LAR-004, BUS-006, ARC-001.

### SEC-003 — El aviso expone logins e IPs a un destino que cualquier editor de settings puede cambiar

Severity: MEDIUM · Category: Security · File: design.md · Line: 62-67 (spec basics-login-audit:62-65) · Confidence: HIGH

Problem: el correo y el contexto de `report()` llevan logins e IPs. `alert_email` lo cambia quien tenga `settings.edit`, aunque no tenga `sessions.view`. Sin Sentry, `report()` acaba en el log diario.

Recommendation: enviar solo recuentos, la ventana y un enlace a `sessions.index`, y no meter logins ni IPs en el contexto de la excepción.

Fuentes: SEC-005, OPS-003.

### DB-001 — Un login demasiado largo da 500 y la fila se pierde

Severity: MEDIUM · Category: Database · File: design.md · Line: 32-33 · Confidence: MEDIUM

Evidence: `vendor/laravel/fortify/src/Http/Requests/LoginRequest.php:28` valida el login solo como `required|string`, sin longitud máxima. MariaDB en modo estricto devuelve "Data too long"; SQLite no, así que los tests no lo verían.

Recommendation: truncar `login` a la longitud de la columna al escribir, como ya se hace con `user_agent`.

Fuentes: DB-005, SEC-004 (solo la parte de longitud).

### FE-001 — El grupo Administración no se muestra a quien solo tiene los permisos nuevos

Severity: MEDIUM · Category: Frontend · File: tasks.md · Line: 39 · Confidence: HIGH

Evidence: `projects/demo13/resources/views/layouts/app/sidebar.blade.php:59` renderiza el grupo con `@if ($seesUsers || $seesRoles)`.

Recommendation: añadir `$seesSessions || $seesSettings` a la condición, y un test con un usuario que solo tenga `sessions.view`.

## Low Findings

### SPEC-001 — La inmutabilidad está sobredimensionada en la spec

Severity: LOW · Category: Correctness · File: specs/basics-login-audit/spec.md · Line: 23 (design.md:32) · Confidence: HIGH

Problem: "por ninguna vía" no se cumple. El query builder se salta los eventos del modelo, y `nullOnDelete` pone `user_id` a nulo al forzar el borrado de un usuario (`UserTrashController.php:49-52`). "En la práctica se conserva" es falso.

Recommendation: redactarlo como "inmutable a través de la aplicación (modelo Eloquent), sin rutas de escritura", y documentar que tras un borrado forzado `user_id` queda nulo pero `login` se conserva.

Fuentes: DB-004, BUS-007, SEC-007, MNT-005.

### BUG-007 — El design dice mal cómo se registra un usuario inactivo

Severity: LOW · Category: Correctness · File: design.md · Line: 45-46 · Confidence: HIGH

Evidence: `FortifyServiceProvider.php:154-176` expulsa con `ValidationException` y nunca dispara `Failed`. Por la vía de contraseña, el inactivo ya lo filtra `authenticateUsing` (136-139), y ahí sí se dispara `Failed`.

Recommendation: corregir el texto. El listener de log ignora a los usuarios `! active` en vez de depender del orden de registro de listeners.

Fuentes: LAR-005, MNT-006.

### PERF-001 — El filtro de fechas no usa índices y el "hasta" es ambiguo

Severity: LOW · Category: Performance · File: tasks.md · Line: 30 · Confidence: HIGH

Evidence:
- `packages/basics13/src/Queries/ListQueryBase.php:253-267` usa `whereDate()`, que aplica `DATE(created_at)` y no puede usar el índice.
- Ordenar solo por `created_at` es inestable cuando muchas filas comparten segundo.
- Filtrar por usuario no alcanza las filas `unknown_user`.

Recommendation:
- Rango sargable inclusivo (`>= from`, `< to + 1 día`).
- `orderByDesc('created_at')->orderByDesc('id')`.
- `login` buscable en el campo de búsqueda.
- Eager load de `user` con `withTrashed()`.
- Selector de usuarios remoto y limitado.

Fuentes: PERF-001, PERF-003, PERF-004, DB-011, FE-007, BUS-012.

### SPEC-002 — Choque del delta de permisos con la impersonación

Severity: LOW · Category: Correctness · File: specs/basics-roles-permissions/spec.md · Line: 4-5 · Confidence: HIGH

Problem: la lista de acciones omite `impersonate`, que `000006:19` ya siembra. `add-superadmin-user-impersonation` edita el mismo requirement.

Recommendation: incluir `impersonate` en la lista y fijar el orden de archivado de los tres changes.

Fuentes: BUS-013, ARC-002, DB-013.

### FE-002 — Pestañas, textos y formulario de la pantalla de configuración

Severity: LOW · Category: Frontend · File: design.md · Line: 12, 78 (tasks.md:34-35) · Confidence: HIGH

Problems y fixes:
- **Pestañas:** reutilizar `x-list.tabs` (`resources/views/components/list/tabs.blade.php`) en lugar de `flux:navbar`, y definir qué pasa con un `?package=` desconocido.
- **Claves de traducción:** "Settings" (`es.json:241`), "Login" y "Logout" ya existen como menú de cuenta y como verbos; usar claves nuevas y específicas.
- **Idioma que falta:** `name_*` y `description_*` caen a `en` y después a `variable`.
- **Formulario:** `data-track-changes` en el formulario; ocultar "Guardar" sin `settings.edit`.

Fuentes: FE-001, FE-002, FE-004, FE-006, FE-008.

### TST-002 — Huecos de tests y un Dusk innecesario

Severity: LOW · Category: Testing · File: tasks.md · Line: 20, 30, 34, 40 · Confidence: HIGH

Problem: la tarea 7.2 (Dusk) no prueba JS. Faltan estos casos:
- settings de solo lectura (200 en GET, 403 en PUT);
- login inexistente con `user_id` nulo;
- `user_id` relleno en contraseña errónea;
- logout que no toca otras filas;
- logout sin `session_key`;
- flujo real `POST login.store`;
- orden descendente y ausencia de rutas de escritura;
- bloqueo de un login inexistente desde varias IPs.

Recommendation: quitar 7.2 y añadir esos casos a 3.3, 5.1 y 6.1. La spec basics-login-audit ya pasa de 70 líneas (presupuesto ~60): fusionar escenarios para compensar.

Fuentes: TST-004, TST-005, TST-006, TST-008, BUS-014.

### DB-002 — Detalles de esquema

Severity: LOW · Category: Database · File: design.md · Line: 30-34, 70 · Confidence: HIGH

Problems y fixes:
- **Sin `updated_at`:** declarar `const UPDATED_AT = null`. Si no, el insert falla o el guard de inmutabilidad salta siempre.
- **Enums nativos:** usar columnas `string` con casts a enum PHP. Ninguna tabla BAS usa enums nativos, y añadir valores obliga a ALTER.
- **`description_*`:** tipo `text`, porque INSERT IGNORE trunca en silencio en MariaDB.

Fuentes: DB-006, DB-007, DB-010.

### MNT-001 — Dónde vive la lógica de bloqueo

Severity: LOW · Category: Maintainability · File: tasks.md · Line: 20, 24 (design.md:52-53) · Confidence: HIGH

Problem: tres listeners más la comprobación del bloqueo harían crecer `FortifyServiceProvider` (254 líneas) y repetirían la normalización de la clave en tres sitios. Además, "el mismo criterio que hoy" es falso: la clave actual incluye la IP.

Recommendation:
- Una clase concreta en `src/Auth` (carpeta existente) con un `lockoutKey()` privado; el provider solo registra los eventos.
- Pedir aprobación antes de crear `src/Mail` y `src/Exceptions`, que son carpetas base nuevas.

Fuentes: MNT-001, MNT-002, ARC-006.

### PERF-002 — El recuento del pico se repite tras haber avisado

Severity: LOW · Category: Performance · File: design.md · Line: 62-64 · Confidence: HIGH

Recommendation: comprobar `Cache::has('basics13.login-alert')` antes de contar, y contar con un límite (`skip(10)->exists()`).

## Informational Findings

- **INFO-001:**
  - **"Más de" frente a "al alcanzar":** la petición dice "más de ese valor" y el plan bloquea al alcanzarlo (3.º fallo, no 4.º). Confirmar con el usuario y anotarlo (BUS-002).
  - **Riesgos no documentados:** no hay desbloqueo manual (ni para el super-admin); los logins por passkey no pasan por el bloqueo; "cerrar otras sesiones" y el cambio de contraseña dejan filas abiertas; la deduplicación es un *cooldown* desde el primer aviso, no ventanas fijas; falta el orden de rollback. Anotarlos como riesgos o non-goals en design.md (BUS-005, BUS-009, BUS-011, OPS-008, SEC INFO).
- **INFO-002:** no hay `trustProxies`, así que `ip()` es hoy la conexión directa. Si se añade un proxy, configurar solo sus IPs explícitas.

## Security

SEC-001 (HIGH), SEC-002, SEC-003 (MEDIUM). Confirmado correcto:
- La comprobación del bloqueo en `authenticateUsing` también cubre el pre-check de 2FA (`RedirectIfTwoFactorAuthenticatable.php:81-82`).
- El mensaje de bloqueo no revela qué cuentas existen, porque los logins inexistentes también se bloquean.
- No se guarda la contraseña.

## Bugs / Correctness

BUG-001, BUG-002 (HIGH); BUG-003 a BUG-006 (MEDIUM); BUG-007, SPEC-001, SPEC-002 (LOW). Confirmado correcto:
- `session_key` sobrevive a la regeneración de la sesión.
- `Logout` se dispara antes de `Session::invalidate()`.
- `Failed` llega con `user = null` cuando se usa `authenticateUsing`. Sus credenciales incluyen el password, así que el listener debe leer solo el campo de login.

## Database

BUG-002, BUG-005, DB-001, DB-002, SPEC-001. Los índices propuestos cubren los filtros y la consulta del pico. El unique `(package, variable)` es correcto.

## Performance

PERF-001, PERF-002. Un INSERT y un lookup por intento son despreciables frente a bcrypt.

## Architecture

Respeta los límites entre paquetes: los dominios siembran sus propias filas en una tabla de `basics13`, y las pestañas salen de los datos. Está justificado que la pantalla de configuración se salga del recurso canónico, y las dos capabilities nuevas están justificadas. Pendientes: MNT-001 (carpetas nuevas) y SPEC-002 (orden de archivado).

## Testing

TST-001 (MEDIUM), TST-002, y la reescritura del test del 429 incluida en SEC-001.

## Production

OPS-001, SEC-003 (sin Sentry, `report()` solo escribe en el log diario).

## Maintainability

MNT-001.

## Rejected Findings

| ID | Motivo |
|---|---|
| SEC-001 (bloqueo por login + IP) | Contradice la decisión del usuario; el riesgo ya figura en design.md y SEC-001 (consolidado) lo mitiga |
| SEC-004 (enmascarar el login) | El usuario pidió guardar el login; solo se acepta la parte de longitud (DB-001) |
| SEC-008 (timing en `authenticateUsing`) | Preexistente, fuera de alcance |
| ARC-005 (traducciones en ficheros en vez de columnas) | Contradice la petición explícita de columnas `name_*` y `description_*` |
| ARC-008 (prefijo `BAS_`) | Sin soporte: `BAS_roles` y `BAS_permissions` ya usan ese prefijo |
| ARC-007, MNT-004 (nombres `settings` / `configuration`) | Preferencia de nombres |
| MNT-003 (constantes) | El design ya las declara como constantes |
| MNT-007 (`class` como texto libre) | Columna pedida por el usuario |
| OPS-005 (purga de la tabla `cache`) | Infraestructura general, no de este change |
| OPS-006 (`trustProxies`) | No hay proxy hoy y la clave de bloqueo no usa la IP (INFO-002) |
| OPS-007 (retención) | Non-goal explícito; una purga futura puede usar el query builder |
| TST-007 (colocación de tests unitarios) | Preferencia; las reglas de la suite ya lo imponen |
| TST-009, DB-012 | Informativos |

## Action Plan

1. **BUG-001:** bloqueo con clave propia y TTL completo; spec con ventana de recuento; test con `travel()`.
2. **SEC-001:** limitador `login` por IP en lugar de `Limit::none()`; reescribir el test del 429.
3. **BUG-002:** sembrar `sessions.*` y `settings.*` completos.
4. **BUG-003:** quitar la caché de `Setting`.
5. **BUG-004:** un único punto que registra el fallo y comprueba el pico; definir la semántica de `locked`.
6. **TST-001:** FormRequests de `index` en 5.1 y 6.1.
7. **OPS-001 + SEC-003:** correo síncrono, solo con recuentos y enlace.
8. **SEC-002:** reglas `min`/`max` y valor por defecto al leer.
9. **BUG-005, BUG-006, DB-001:** logout idempotente, ignorar remember-me e impersonación, truncar `login`.
10. **FE-001:** condición del grupo Administración en el sidebar.
11. **SPEC-001, BUG-007, SPEC-002, INFO-001:** corregir el texto de spec y design; preguntar "más de" frente a "alcanzar".
12. **LOW restantes:** PERF-001, FE-002, TST-002, DB-002, MNT-001, PERF-002.

Todo se corrige en los artefactos del change (`/openspec-update-change`) antes de `/openspec-apply-change`.
