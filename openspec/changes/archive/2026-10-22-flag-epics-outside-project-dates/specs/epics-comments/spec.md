## MODIFIED Requirements

### Requirement: El epic hereda de su proyecto la pertenencia y las fechas
Un epic MUST pertenecer a un proyecto no eliminado, y la validación MUST aceptar el proyecto que ya
tiene aunque esté archivado. Al guardar un epic, sus fechas MUST quedar dentro de las del proyecto.
Cambiar las fechas de un proyecto MUST NOT rechazarse por sus epics; el listado de epics MUST
marcar en rojo, con texto alternativo, las fechas de un epic que hayan quedado fuera de las del
proyecto.

#### Scenario: Se edita un epic cuyo proyecto fue archivado
- **WHEN** se guarda el formulario sin cambiar el proyecto asignado
- **THEN** la validación lo acepta, y un epic fuera de las fechas del proyecto se rechaza

#### Scenario: El proyecto cambia de fechas y deja un epic fuera
- **WHEN** se acortan las fechas de un proyecto y uno de sus epics queda fuera de ellas
- **THEN** el proyecto se guarda, y el listado de epics muestra en rojo las fechas de ese epic con el texto "Fuera de las fechas del proyecto"
