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