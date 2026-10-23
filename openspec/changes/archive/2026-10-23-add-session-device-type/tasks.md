# Tasks

## 1. Datos

- [x] 1.1 Enum `DeviceType` (`desktop`, `mobile`, `tablet`) con `fromUserAgent()` y test unitario con agentes reales (iPhone, iPad, Android móvil y tablet, escritorio, vacío)
- [x] 1.2 Migración en basics13: columna `device_type` (default `desktop`) y relleno por lotes de las filas existentes
- [x] 1.3 `SessionLog`: fillable, cast al enum y PHPDoc; factory con `device_type`
- [x] 1.4 `LoginAudit` guarda `device_type` en login, fallo y bloqueo; tests Feature

## 2. Panel

- [x] 2.1 `SessionDashboardQuery::summary()` devuelve `devices` (recuento por tipo de los logins correctos del periodo, los tres siempre)
- [x] 2.2 Componente `chart` y su JS admiten `donut` (labels propios, sin gradiente ni ejes)
- [x] 2.3 Tarjeta «Sesiones por dispositivo» en `sessions/dashboard.blade.php`: anillo, total en el centro, leyenda con %, estado vacío
- [x] 2.4 Textos nuevos en es, eu, fr y en
- [x] 2.5 Tests Feature del escenario de reparto y del estado vacío en `SessionDashboardTest`

## 3. Impersonación

- [x] 3.1 Columna `impersonator_id` en `BAS_sessions`; relación `impersonator` en `SessionLog`
- [x] 3.2 `LoginAudit` registra la impersonación marcada y fija su logout al salir o al cerrar sesión
- [x] 3.3 Distintivo «Impersonado por» en el listado; textos en es, eu, fr y en; tests Feature

## 4. Verificación

- [x] 4.1 Pint, PHPStan nivel 7 y suites de basics13 y del resto de paquetes y demo13
- [x] 4.2 Pedir al usuario el Dusk del panel
