# Proposal

## Why

Los proyectos y los epics ya tienen fechas (planning y timeline las agregan en listas), pero no existe una vista de calendario que muestre cuándo vence cada uno, ni un sitio para los eventos propios del usuario. El calendario es la vista natural sobre esas fechas y necesita además un calendario personal que solo ese usuario controla.

## What Changes

- Pantalla nueva de calendario en `projects13`, con vistas de mes, semana, día y año.
- Tres calendarios en la misma pantalla:
  - Calendario de proyectos: derivado de las fechas de los proyectos activos, sin registros propios (como planning/timeline, lectura pura).
  - Calendario de epics: derivado de las fechas de los epics activos dentro de proyectos activos, sin registros propios.
  - Calendario personal: eventos propios del usuario (`basics13`), el ÚNICO calendario donde se pueden crear, editar y eliminar eventos.
- El botón de crear evento, el formulario y las acciones solo existen para el calendario personal (modelo nuevo en `basics13`, con `user_id` obligatorio).
- Visibilidad por calendario: los tres se muestran juntos pero cada uno se puede ocultar de forma individual (el estado en la URL, como las vistas de planning).
- Ítem de menú sembrado por `projects13` para la ruta del calendario.
- Sin registros nuevos para proyectos ni epics; sus datos y políticas actuales no cambian.

## Capabilities

### New Capabilities

- `calendars`: la pantalla de calendario, el calendario personal y sus invariantes propias (dueño, visibilidad, vistas). Capability nueva porque ninguna existente cubre un recurso de calendario; lo compartido con planning/timeline queda en esa capability (ver Modified).

### Modified Capabilities

- `planning-timeline`: sus requirements de "solo lectura" y "una sola fecha es un hito" se amplían para cubrir el calendario como pantalla agregada más.

## Impact

- `basics13`: modelo + fábrica + migración de la tabla de eventos personales, policy y request del formulario (repo de `basics13`).
- `projects13`: controller de calendario, query que deriva proyectos/epics en eventos, vistas del calendario (mes/semana/día/año) y siembra de menú nueva; el calendario lee los eventos personales de `basics13` (basics es infraestructura neutral, los dominios ya dependen de ella).
- `projects/demo13`: nada (usa los paquetes existentes).
- Tests: Feature en `basics13` (CRUD personal) y en `projects13` (pantalla agregada); PHPStan nivel 7 y cobertura ≥ 80 % por paquete.
