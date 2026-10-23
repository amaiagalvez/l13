# Tasks

Rutas relativas a `packages/basics13/` salvo indicación. Requiere `add-session-log-and-settings` implementado.

## 1. Datos

- [x] 1.1 `SessionDashboardRequest::period()` (7/30/90, defecto 7 ante ausente, desconocido, texto o array) y `authorize()` con `viewAny` de `SessionLog`; test Unit de `period()` con `45`, `abc` y `period[]=7`.
- [x] 1.2 `Queries/Sessions/SessionDashboardQuery`: usuarios no borrados, fallos `failed` del periodo, serie diaria de `login` (total y usuarios distintos) con días a cero. Tests Feature con `travel()`: escenario «Recuentos del periodo» del spec, borde del primer día incluido y del día anterior excluido, usuarios borrados no cuentan y archivados sí, rechazo por login bloqueado cuenta y `lockout` no.

- [x] 1.3 KPIs de cuentas en `SessionDashboardQuery` (activas, nuevas, inactivas, pendientes, crecimiento) según el design; test Feature con el escenario «KPIs de cuentas» y crecimiento sin cuentas previas; variación frente al periodo anterior de activas, nuevas y fallos (escenario «Comparación con el periodo anterior»).

## 2. HTTP y vista

- [x] 2.1 `SessionDashboardController` y ruta `sessions.dashboard` en `routes/sessions.php`; tests Feature: 200 con `sessions.view`, 403 sin él, sin periodo o con `period=45` muestra 7 días con su botón activo, `ArchitectureTest` en verde.
- [x] 2.2 `x-basics13::chart` con prop `colors` opcional y lectura de colores en el componente Alpine `chart`; los usos sin `colors` no cambian.
- [x] 2.3 Vista `resources/views/sessions/dashboard.blade.php` (skill `ux-design`): selector de periodo, tarjetas «Usuarios» y «Accesos fallidos», gráfica de dos series con etiqueta accesible que resume el periodo, la tarjeta de fallos enlaza al listado filtrado por fallidos y por el rango de fechas del periodo; `data-test` por cifra. Tests Feature de las cifras en el HTML y del enlace con ambos filtros.
- [x] 2.4 Claves nuevas en `resources/lang/{es,eu,fr,en}.json` («Administration dashboard», «Failed sign-ins», «Unique users», «Sign-ins», «Last :days days»…); test de traducciones en verde.

- [x] 2.5 Tarjetas de los KPIs en la vista (rejilla responsive, «—» sin base de crecimiento, variación con signo y texto «frente a los :days días anteriores», color según si subir es bueno) y sus claves en los 4 idiomas; test Feature de las cifras.

- [x] 2.6 Alertas en la query (constantes de umbral) y bloque en la vista, solo con alertas activas; claves en 4 idiomas; tests Feature de los escenarios «Solo las alertas justificadas» y «Sin alertas».

## 3. Host

- [x] 3.1 `projects/demo13`: entrada «Panel de administración» (`sessions.view`, icono `chart-bar`, `data-test` `nav-admin-dashboard`) al principio del grupo Administración y «Sesiones» sin marcarse en el panel; test Feature del sidebar.

## 4. Verificación

- [x] 4.1 Pint, PHPStan nivel 7 y suites de `basics13` y `demo13` en verde. Test Dusk en `basics13`: el panel pinta dos series con colores distintos (la gráfica de un color no cambia de camino en el JS y no lleva Dusk propio); dar el comando al usuario, no ejecutarlo.
