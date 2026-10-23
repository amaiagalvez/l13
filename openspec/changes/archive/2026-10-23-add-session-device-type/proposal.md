# Proposal

## Why

El panel de administración no dice desde qué tipo de dispositivo se entra. El registro de sesiones
solo guarda el agente de usuario en bruto, que no se puede agrupar ni contar en una consulta.

## What Changes

- `BAS_sessions` gana la columna `device_type` (`desktop`, `mobile` o `tablet`), clasificada a partir
  del agente de usuario al escribir cada fila (login, fallo y bloqueo).
- La migración rellena las filas existentes con la misma clasificación.
- El panel de administración añade un gráfico de anillo «Sesiones por dispositivo» con el total de
  inicios de sesión correctos del periodo en el centro y el % de cada tipo en la leyenda.
- El componente de gráfico admite el tipo `donut`.

## Capabilities

### New Capabilities

### Modified Capabilities
- `basics-login-audit`: cada fila registra el tipo de dispositivo y el panel lo resume en el periodo.

## Impact

- `packages/basics13`: migración, `SessionLog`, nuevo enum `DeviceType`, `LoginAudit`,
  `SessionDashboardQuery`, vista `sessions/dashboard`, componente `chart` y su JS, factory, i18n y tests.
- Sin dependencias nuevas: la clasificación usa expresiones regulares propias.
