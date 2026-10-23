# Design

## Context

`LoginAudit` escribe las filas de `BAS_sessions` y `SessionLog` las hace inmutables (solo
`logged_out_at` puede fijarse). El panel usa `SessionDashboardQuery` y el componente `chart`, que hoy
solo pinta series por categoría (área/línea).

## Goals / Non-Goals

**Goals:** clasificar una vez al escribir para poder agrupar con SQL.

**Non-Goals:** detectar el dispositivo de forma segura (el agente de usuario es falseable), bots o
modelos concretos; distinguir un iPad con iPadOS 13+ (se presenta como Mac y queda `desktop`).

## Decisions

- **Columna frente a cálculo al leer**: se guarda en `device_type` (string, como todos los enums de
  BAS). Sin índice nuevo: el panel ya filtra por `(event, created_at)` y agrupa pocas filas.
- **Clasificación**: enum `DeviceType` con `fromUserAgent(string)`, expresiones regulares propias
  (tablet: `iPad|Tablet|PlayBook|Silk|Kindle` o `Android` sin `Mobile`; mobile: `Mobi|iPhone|iPod|
  Android|Windows Phone|Opera Mini`; resto desktop). Alternativa descartada: `mobiledetect` o
  `jenssegers/agent`, una dependencia nueva para tres categorías.
- **Relleno**: la migración añade la columna con valor por defecto `desktop` y reclasifica por lotes
  con el query builder (`chunkById`), sin pasar por el modelo, cuya inmutabilidad lo rechazaría. El
  modelo sigue rechazando cambios de `device_type`.
- **Gráfico**: el componente `chart` acepta `type="donut"`; en ese caso `series` es una lista de
  números y `labels` sus nombres (ApexCharts `labels`, no `xaxis.categories`). Total en el centro y
  leyenda con % se pintan en Blade, accesibles sin JS.

## Risks / Trade-offs

- [Agente de usuario falseable o iPad como Mac] → solo estadística, documentado en el modelo.
- [Relleno lento con muchas filas] → por lotes; la tabla es de auditoría y moderada.
