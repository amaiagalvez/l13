## Context

`basics13` ya tiene `BAS_sessions` (`SessionLog`, eventos `login`, `failed`, `lockout`), la policy
`SessionLogPolicy` con `sessions.view` y el componente `x-basics13::chart` (ApexCharts cargado bajo
demanda) que usa el dashboard de `projects13`. Ese componente toma un único color: el `color` del
elemento.

## Goals / Non-Goals

**Goals:** resumen de solo lectura con dos cifras y una gráfica diaria, periodo 7/30/90.

**Non-Goals:** filtros por usuario o IP (ya están en el listado), exportación, caché, tiempo real,
permiso propio.

## Decisions

- **Ubicación y ruta.** `GET sessions/dashboard`, nombre `sessions.dashboard`, en
  `routes/sessions.php` (sigue sin rutas de escritura). `SessionDashboardController` invocable con
  `authorize('viewAny', SessionLog::class)`. En el sidebar la entrada «Sesiones» pasa a
  `routeIs('sessions.*') && ! routeIs('sessions.dashboard')`, como ya se hace con `users.*`.
- **Periodo.** `SessionDashboardRequest::period()` devuelve 7, 30 o 90; cualquier otro valor da 30
  sin error de validación (es un selector, no un formulario). Días del calendario en la zona horaria
  de la app, de `hoy - (n-1)` a hoy incluido, con rango sargable sobre `created_at`.
- **Query.** `Queries/Sessions/SessionDashboardQuery` (como `DashboardSummaryQuery`): usuarios con
  el modelo de `auth.providers.users.model` sin borrados (activos y archivados); fallos =
  `event = failed` (incluye rechazos `locked`; `lockout` no cuenta, es consecuencia de un fallo);
  serie = una consulta agrupada por `DATE(created_at)` con `COUNT(*)` y
  `COUNT(DISTINCT user_id)` sobre `event = login`, rellenada en PHP con ceros. `DATE()` funciona en
  SQLite y MySQL.
- **Gráfica de dos series.** `x-basics13::chart` acepta `colors` (lista de clases de color); el
  componente Alpine lee el color calculado de un `<span>` oculto por clase y, sin `colors`, sigue
  usando el del elemento. Sin cambios para los usos existentes. Tipo `bar` o `area`: `area`, como
  el resto.
- **Vista.** `resources/views/sessions/dashboard.blade.php`: cabecera, selector de periodo con
  `flux:radio.group variant="segmented"` que navega con `?period=`, dos tarjetas de cifra y la
  gráfica; enlace «Ver sesiones» al listado filtrado por fallidos.

## Risks / Trade-offs

- 90 días sin caché: una consulta agregada con índice en `created_at`; aceptable. → Revisar si el
  registro crece mucho.
- `COUNT(DISTINCT user_id)` ignora las filas sin usuario; los logins correctos siempre lo tienen.
