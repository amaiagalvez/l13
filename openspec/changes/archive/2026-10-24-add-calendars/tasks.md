# Tasks

Aplicar siguiendo AGENTS.md y la regla de zona de cada ruta. Commit en el repo del paquete dueño.

## 1. `basics13` — recurso de eventos personales

- [x] 1.1 Migración + modelo `PersonalEvent` (fillable, cast de fechas, FK a `User` con borrado en cascada — sin papelera ni `active`) en `packages/basics13/src`, con `user_id`, título, nota nullable y fechas; verificar con `DX php artisan test packages/basics13/tests/Unit --compact`
- [x] 1.2 Factory con fechas variadas (un día, un rango, cruzando mes) y verificar generándola en un test
- [x] 1.3 Policy: cada usuario ve y manipula solo sus eventos; otros reciben 403; verificar con `DX php artisan test packages/basics13/tests/Feature --filter=PersonalEvent --compact`
- [x] 1.4 Form request: título y fecha de inicio obligatorios, fecha de fin opcional y coherente (fin ≥ inicio), error por campo; verificar con test Feature de store/update inválidos
- [x] 1.5 `PersonalEventController` + rutas (store/update/destroy) en `basics13`; el código y los tests del grupo 1 no referencian `projects13` (borde de paquete); ejecutar la suite de `basics13` al cerrar el grupo y comprobar que pasa

## 2. `projects13` — pantalla de calendario

- [x] 2.1 Query derivada: proyectos y epics activos (papelera y archivados fuera, policies de `projects.view`/`epics.view`) en eventos en memoria + eventos personales del usuario autenticado, con el matiz de una sola fecha → hito de un día; verificar con test del query (archivado con epics activos no aparece, sin fecha no genera evento, una fecha → un día)
- [x] 2.2 Ruta + `CalendarController` con estado en la URL (`?view=`, `?date=`, `?calendars=`) y atajo de hoy; verificar con test Feature que recarga manteniendo vista, periodo y calendarios ocultos
- [x] 2.3 Vistas Blade de mes, semana, día y año bajo `resources/views/calendars/`, mismo conjunto de eventos en las cuatro, cada una acotada a su periodo; verificar con test parametrizado: el mismo evento visible en las cuatro vistas de su periodo y ausente fuera
- [x] 2.4 Modal del evento personal (crear/editar/eliminar) solo para el calendario personal, y sin acciones sobre proyectos ni epics; verificar con test Feature: crear como autor, editar/borrar ajeno → 403, y 404/403 al pedir acción de evento de dominio
- [x] 2.5 Siembra idempotente del ítem de menú sin permisos; verificar con test Feature de la siembra repetida y de visibilidad del ítem para un usuario autenticado sin permisos de dominio
- [x] 2.6 i18n del nuevo texto en `resources/lang/{es,eu,fr,en}.json` (calendarios, vistas, formularios, "Fuera de..." si aplica); verificar con test de traducciones existentes

- [x] 2.7 Aspecto de la maqueta: panel lateral (crear, mini mes, casillas con recuento por calendario), selector de vista segmentado, días de relleno en la rejilla del mes y celdas acotadas con "+N más" hacia la vista de día (mes del año → vista de mes); verificar con test Feature del límite y del enlace, y de los recuentos con un calendario oculto

## 3. Verificación final

- [x] 3.1 Suite completa de todos los paquetes (`basics13`, `entities13`, `projects13`, `entity-projects13`) y de `demo13` con `PHPUNIT_ARGS="--no-progress"` en Docker, más Pint y PHPStan nivel 7 (`--no-progress --error-format=raw`); cobertura ≥ 80 % en `basics13` y `projects13`
- [x] 3.2 Prueba manual en Docker (Playwright) de las cuatro vistas, crear/editar/borrar un evento personal y ocultar cada calendario (pantallazos del resultado)
- [x] 3.3 Dar al usuario el comando de Dusk si el modal añade comportamiento JS (no ejecutarlo yo); esperar resultado pasado

## Workflow follow-up

- Archivar el change tras el merge (PR propio) y comprobar el presupuesto (~60 líneas) de la spec `calendars`.
- Actualizar el mapa de specs de AGENTS.md y las invariantes de `.github/instructions/*.instructions.md` si el archivado crea la capability nueva.
