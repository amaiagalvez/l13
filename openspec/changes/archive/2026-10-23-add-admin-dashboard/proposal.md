## Why

El grupo Administración solo ofrece listados: para saber cuántas cuentas hay, si están fallando
accesos o cuánta gente entra cada día hay que filtrar el registro de sesiones a mano. Un panel de
resumen da esa foto de un vistazo y hace visible un pico de fallos sin esperar al aviso por correo.

## What Changes

- Página nueva «Panel de administración» en `basics13`, de solo lectura, con:
  - KPIs de cuentas (no borradas): total, activas, nuevas, inactivas, pendientes de verificar el
    email y tasa de crecimiento respecto al inicio del periodo;
  - número de intentos de acceso fallidos en el periodo;
  - gráfica por día de los inicios de sesión correctos: total y usuarios únicos.
- Periodo seleccionable de 7, 30 o 90 días (por defecto 7), hoy incluido; un valor desconocido
  usa el defecto.
- Permiso: reutiliza `sessions.view`, sin permisos nuevos.
- El componente compartido `x-basics13::chart` admite varias series con un color por serie.
- `projects/demo13`: entrada «Panel de administración» al principio del grupo Administración.

## Capabilities

### New Capabilities

Ninguna.

### Modified Capabilities

- `basics-login-audit` (de `add-session-log-and-settings`, aún sin archivar): requirement nuevo del
  resumen de accesos. Los datos salen solo de `BAS_sessions` y del modelo de usuario, ya dentro de
  esa capability; no justifica una capability propia.

## Impact

- `basics13`: ruta `sessions.dashboard`, controller, request, query en `src/Queries/Sessions`,
  vista, ampliación de `components/chart.blade.php` y del componente Alpine `chart`, traducciones
  es/eu/fr/en y tests.
- `projects/demo13`: sidebar y su test.
- Sin dependencias nuevas (ApexCharts ya está). Sin migraciones.
- Orden: implementar y archivar después de `add-session-log-and-settings`.
