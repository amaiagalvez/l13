# Design

## Context

La pantalla replica el patrón de planning/timeline (`PlanningController`, `PlanningQuery`): controller + Blade en `projects13`, lectura pura del dominio, fechas ya validadas (las de un epic van dentro de las del proyecto, matiz de año bisiesto para el 29/feb en el día). Los eventos personales son un recurso nuevo en `basics13`, que ya posee los modelos de ámbito de usuario (`SessionLog`, `Setting`). Ningún paquete de dominio depende de otro, así que el calendario vive donde están los proyectos y lee `basics13`.

## Goals / Non-Goals

**Goals:**
- Las tres fuentes comparten las cuatro vistas y su navegación, con visibilidad individual en la URL.
- Creación, edición y borrado de eventos personales restringidos a los propios.

**Non-Goals:**
- Eventos de dominio nuevos, recordatorios por correo, compartir calendarios, arrastrar eventos para moverlos.
- Vista de lista (la propuesta quedó en mes, semana, día y año).

## Decisions

1. **Pantalla y query en `projects13`, eventos personales en `basics13`.** El calendario es una vista sobre proyectos/epics, la pipeline de `PlanningController` sirve de molde; el recurso del usuario va a infra neutral. Alternativa descartadas: el recurso en `demo13` (partiría la ya existente en dos repos para una sola pantalla) o en `projects13` (el recurso no es del dominio de proyectos).
2. **Controller + Blade, no Livewire.** Las vistas y la navegación son GET con estado en la URL (`?view=week&date=2026-06-12&calendars=epics`), exactamente como las pestañas de planning. Sin componentes dinámicos: ni ciclo de JS para lograrlo. El `Create event` y la edición abren un modal simple (no panel nuevo): un solo modal reutilizable para el formulario, como en los formularios de los dominios.
3. **Derivar, no persistir.** El query del calendario recoge fechas de proyectos y epics activos y las convierte en eventos en memoria (rango inicio→fin, una sola fecha → un día); los personales salen de su tabla. La tabla de eventos personales guarda `user_id`, título, nota (`text`, nullable), fecha de inicio obligatoria y fecha de fin opcional (`date`, sin zona: se interpretan en la zona de la aplicación, lo que facilita el caso de borde del 29-feb), índice por usuario y fechas. Cada evento se dibuja según su origen: proyecto, epic o personal. Los eventos personales son CRUD plano: sin papelera ni `active`/archivados — desviarse del patrón trash del proyecto a propósito, porque un evento de usuario no es un recurso de dominio — y la FK a `User` debe borrarlos en cascada.
4. **Nada de permissions para el calendario.** El ítem del menú sin permisos (visible a cualquier autenticado), porque el calendario personal no depende de permisos de dominio y ocultar los de proyectos basta cuando una policy devuelve vacío; los eventos de proyecto/epic respetan las policies de sus listados (`projects.view`, `epics.view`).

## Risks / Trade-offs

- [Las cuatro vistas multiplican las pruebas de borde (meses de 28–31 días, semana a caballo de mes/año, 29-feb)] → Factoría de eventos personales con fechas variadas, test común por vista: los mismos eventos deben salir en las cuatro; navegación de extremo y vuelta a hoy por test Feature.
- [Un usuario puede intentar inyectar parámetro de calendario falsificado] → Los eventos personales se filtran siempre por `user_id` del usuario autenticado en el query; la policy solo autoriza si el dueño concuerda.
- [Manual: la tabla personal no puede ligarse al planificación existente] → Ninguna de las tablas de dominio cambia: la migration solo añade la tabla nueva; rollback trivial al drop de la tabla.
