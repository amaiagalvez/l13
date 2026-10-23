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
  `routes/sessions.php` (sigue sin rutas de escritura). `SessionDashboardController` invocable; la autorización
  (`viewAny` sobre `SessionLog`) va en `SessionDashboardRequest::authorize()`, como en
  `SessionLogListRequest`. En el sidebar la entrada «Sesiones» pasa a
  `routeIs('sessions.*') && ! routeIs('sessions.dashboard')`, como ya se hace con `users.*`.
- **Periodo.** `SessionDashboardRequest::period()` devuelve 7, 30 o 90; cualquier otro valor (texto, array, número no
  admitido) da 7 sin error de validación (es un selector, no un formulario). Días del calendario en la zona horaria
  de la app, de `hoy - (n-1)` a hoy incluido, con rango sargable sobre `created_at`.
- **Query.** `Queries/Sessions/SessionDashboardQuery` (como `DashboardSummaryQuery`): usuarios con
  el modelo de `auth.providers.users.model` sin borrados (activos y archivados); fallos =
  `event = failed` (incluye rechazos `locked`; `lockout` no cuenta, es consecuencia de un fallo);
  serie = una consulta agrupada por `DATE(created_at)` con `COUNT(*)` y
  `COUNT(DISTINCT user_id)` sobre `event = login`, rellenada en PHP con ceros. Agrupar por
  `DATE(created_at)` equivale a días de la app porque `app.timezone` es UTC; si cambia, hay que
  agrupar con la conversión de zona.
- **KPIs de cuentas, sin columnas nuevas.** Activas = usuarios distintos con fila `login` en el
  periodo (no hay otra señal de uso). Inactivas = cuentas con `active = true` sin fila `login` en el
  periodo (las archivadas no pueden entrar). Pendientes = `email_verified_at` nulo (no existe
  aprobación de cuentas). Crecimiento = nuevas / cuentas creadas antes del periodo × 100, redondeado;
  sin cuentas previas se muestra «—». Las cuentas borradas no cuentan en ninguna cifra, tampoco en la
  base del crecimiento (aproximación aceptada: no hay histórico de bajas).
- **Comparación con el periodo anterior** (consejo de diseño del usuario). Solo en cifras de flujo:
  activas, nuevas y fallos, contra la ventana inmediatamente anterior de igual duración. Total,
  inactivas y pendientes son fotos del momento y no tienen periodo anterior fiable. Variación
  `(actual − anterior) / anterior × 100` con un decimal y formato del idioma; sin valor si el
  anterior es 0. Color con signo y texto, nunca solo color: subir es bueno en activas y nuevas
  (`success`) y malo en fallos (`danger`).
- **Alertas.** Calculadas en la misma query, una `flux:callout` por alerta y ninguna sección sin
  alertas. Inactividad: 30 días fijos (no el periodo), cuentas `active = true`. Registros: umbrales
  en constantes (`SURGE_MIN_NEW = 5`, `SURGE_FACTOR = 2`) para no avisar por 1 frente a 0.
  «Suspendidas» = archivadas (`active = false`) con `updated_at` en el periodo: no hay fecha de
  archivado, así que otra edición de una cuenta ya archivada también cuenta (aproximación aceptada).
  Cada alerta enlaza a su listado si el visitante puede listar usuarios.
- **Gráfica de dos series.** `x-basics13::chart` acepta `colors` (lista de clases de color); el
  componente Alpine lee el color calculado de un `<span>` oculto por clase y, sin `colors`, sigue
  usando el del elemento. Sin cambios para los usos existentes. Tipo `bar` o `area`: `area`, como
  el resto.
- **Vista.** `resources/views/sessions/dashboard.blade.php`: cabecera, selector de periodo con
  `flux:radio.group variant="segmented"` que navega con `?period=`, dos tarjetas de cifra y la
  gráfica; la tarjeta de fallos enlaza al listado filtrado por resultado fallido y por el rango de fechas del
  periodo, para que cuadre con la cifra de la tarjeta.

## Risks / Trade-offs

- 90 días sin caché: una consulta agregada que usan el índice `(event, created_at)`; aceptable. → Revisar si el
  registro crece mucho.
- `COUNT(DISTINCT user_id)` ignora las filas sin usuario; los logins correctos siempre lo tienen.
