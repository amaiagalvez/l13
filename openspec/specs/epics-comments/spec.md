# Epics y comentarios

## Purpose
Garantiza los invariantes del epic y de sus comentarios: la relación con el proyecto que lo contiene,
el alcance de las fechas heredadas y que un comentario sea un agregado y no un recurso.

## Requirements

### Requirement: El nombre del epic es único dentro de su proyecto
El nombre MUST ser único entre los no eliminados de su mismo proyecto, incluidos los archivados, y el
mismo nombre MUST existir en otro proyecto. Sin id de proyecto utilizable, MUST seguir impidiendo el
duplicado.

#### Scenario: Dos proyectos distintos usan el mismo nombre de epic
- **WHEN** se crea un epic con el nombre de otro de otro proyecto
- **THEN** la creación se acepta porque la unicidad es por proyecto

#### Scenario: Se envía un epic sin proyecto válido
- **WHEN** se envía un epic cuyo proyecto no pasa la validación
- **THEN** el nombre sigue validándose y el error va a su campo

### Requirement: El epic hereda de su proyecto la pertenencia y las fechas
Un epic MUST pertenecer a un proyecto no eliminado, y la validación MUST aceptar el proyecto que ya
tiene aunque esté archivado. Sus fechas MUST quedar dentro de las del proyecto.

#### Scenario: Se edita un epic cuyo proyecto fue archivado
- **WHEN** se guarda el formulario sin cambiar el proyecto asignado
- **THEN** la validación lo acepta, y un epic fuera de las fechas del proyecto se rechaza

### Requirement: El borrado se bloquea en cuanto hay comentarios
Un epic con comentarios MUST NOT pasar a la papelera ni ser eliminado de forma definitiva, y su fila
MUST NOT ofrecer la acción de borrado mientras los tenga.

#### Scenario: Un epic con comentarios
- **WHEN** se intenta mandar a la papelera un epic que ya tiene comentarios
- **THEN** el borrado se rechaza y se informa de la dependencia

### Requirement: El comentario es un agregado del epic
Un comentario MUST crearse por la habilidad de comentar el epic y MUST llevar el autor tomado de la
petición, no del payload. No tiene request propio ni se edita ni se borra por separado, y MUST NOT
quedar huérfano si su autor desaparece.

#### Scenario: Un comentario falló la validación
- **WHEN** el cuerpo de un comentario es inválido
- **THEN** el error aparece junto al comentario, no junto a los campos del epic

### Requirement: El comentario es un "Half-Resource" sin ciclo de vida independiente
Un comentario NO es un recurso completo: NO usa soft deletes, NO tiene flag `active` ni archivo/restauración, NO tiene list query propia ni select options, y NO tiene rutas de lista/creación/edición/eliminación independientes. Existe SOLO como agregado del epic a través de `Epic::comments()` y se gestiona via `EpicCommentController::store()` y `EpicCommentController::show()` (para carga reciente en el epic). Su policy (`EpicCommentPolicy`) permite todas las acciones por consistencia, pero la autorización real se delega al epic (`$user->can('comment', $epic)` en `EpicCommentRequest`).

#### Scenario: El comentario no tiene papelera ni archivo
- **WHEN** se consulta la tabla `PRO_epic_comments`
- **THEN** no existe columna `deleted_at` ni `active`, y no hay controladores `EpicCommentTrashController` ni `EpicCommentInactiveController`

#### Scenario: El comentario no aparece en listados ni selects globales
- **WHEN** se buscan list queries o select options para comentarios
- **THEN** no existe `EpicCommentListQuery` ni `EpicCommentSelectOptionsQuery`; los comentarios solo se cargan via `Epic::comments()->with('user')->latest()->limit(...)`

#### Scenario: La policy existe pero la autorización real está en el epic
- **WHEN** se evalúa `EpicCommentPolicy::create()`
- **THEN** retorna `true`, pero `EpicCommentRequest::authorize()` verifica `$user->can('comment', $epic)` que sí aplica reglas de negocio