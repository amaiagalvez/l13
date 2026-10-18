# Entities

## Purpose
Garantiza los invariantes del recurso cliente que no se deducen de su tabla: el alcance real de la
unicidad del nombre, quién bloquea su eliminación y qué ofrecen los selectores que lo eligen.

## Requirements

### Requirement: El nombre es único entre los no eliminados
El nombre MUST ser único entre los registros no eliminados, incluidos los archivados, y MUST poder
reutilizarse cuando el registro que lo usaba está en la papelera.

#### Scenario: Un cliente archivado conserva su nombre frente a uno nuevo
- **WHEN** se crea un cliente con el nombre de otro, ya archivado o en la papelera
- **THEN** el archivado se rechaza por conflicto y el de la papelera ofrece crear o restaurar

### Requirement: El borrado se bloquea en cuanto hay proyectos
Un cliente con proyectos, incluidos los de la papelera, MUST NOT pasar a la papelera ni ser eliminado de
forma definitiva. La guarda MUST contar los proyectos eliminados.

#### Scenario: Un cliente tiene un proyecto eliminado
- **WHEN** se intenta borrar un cliente cuyo único proyecto está en la papelera
- **THEN** el borrado se rechaza y se informa de la dependencia

### Requirement: El flag de archivado no se propaga
Archivar un cliente MUST NOT archivar sus proyectos ni sus epics: siguen siendo trabajo activo y MUST
seguir en los listados de activos y en los tableros.

#### Scenario: Se archiva un cliente que tiene proyectos activos
- **WHEN** un cliente con proyectos se archiva
- **THEN** sus proyectos permanecen en el listado de activos

### Requirement: Los selectores solo ofrecen clientes activos
El endpoint de opciones MUST devolver solo clientes activos, y la validación MUST exigir uno activo
salvo al editar el registro que ya lo tiene.

#### Scenario: Se edita un registro cuyo cliente fue archivado
- **WHEN** se abre el formulario con el cliente asignado archivado
- **THEN** la selección sigue siendo válida y no se ofrece para elegir otro

### Requirement: La fila resume lo que cuelga del cliente
La fila MUST ofrecer borrar mientras no haya proyectos y MUST cambiarla por archivar en cuanto haya
alguno.

#### Scenario: El cliente pasa a tener un proyecto
- **WHEN** se asigna un proyecto a un cliente que no tenía ninguno
- **THEN** su fila pasa de ofrecer borrar a ofrecer archivar