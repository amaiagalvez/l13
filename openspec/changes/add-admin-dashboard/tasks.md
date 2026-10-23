# Tasks

Rutas relativas a `packages/basics13/` salvo indicación. Requiere `add-session-log-and-settings` implementado.

## 1. Datos

- [ ] 1.1 `SessionDashboardRequest::period()` (7/30/90, defecto 30 ante ausente o desconocido); test Unit.
- [ ] 1.2 `Queries/Sessions/SessionDashboardQuery`: usuarios no borrados, fallos `failed` del periodo, serie diaria de `login` (total y usuarios distintos) con días a cero. Tests Feature con `travel()`: escenario «Recuentos del periodo» del spec, borde del primer día incluido y del día anterior excluido, usuarios borrados no cuentan y archivados sí, `lockout` no cuenta.

## 2. HTTP y vista

- [ ] 2.1 `SessionDashboardController` y ruta `sessions.dashboard` en `routes/sessions.php`; tests Feature: 200 con `sessions.view`, 403 sin él, `period=45` muestra 30 días, `ArchitectureTest` en verde.
- [ ] 2.2 `x-basics13::chart` con prop `colors` opcional y lectura de colores en el componente Alpine `chart`; los usos sin `colors` no cambian.
- [ ] 2.3 Vista `resources/views/sessions/dashboard.blade.php` (skill `ux-design`): selector de periodo, tarjetas «Usuarios» y «Accesos fallidos», gráfica de dos series con etiqueta accesible que resume el periodo, enlace al listado filtrado por fallidos; `data-test` por cifra. Test Feature de las cifras en el HTML.
- [ ] 2.4 Claves nuevas en `resources/lang/{es,eu,fr,en}.json` («Administration dashboard», «Failed sign-ins», «Unique users», «Sign-ins», «Last :days days»…); test de traducciones en verde.

## 3. Host

- [ ] 3.1 `projects/demo13`: entrada «Panel de administración» (`sessions.view`, icono `chart-bar`, `data-test` `nav-admin-dashboard`) al principio del grupo Administración y «Sesiones» sin marcarse en el panel; test Feature del sidebar.

## 4. Verificación

- [ ] 4.1 Pint, PHPStan nivel 7 y suites de `basics13` y `demo13` en verde. Dusk de la gráfica: dar el comando al usuario, no ejecutarlo.
