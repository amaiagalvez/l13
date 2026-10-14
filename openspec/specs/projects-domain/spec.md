# Projects

## Purpose
Garantiza los invariantes del recurso proyecto que no se deducen de su tabla: qué lo hace único,
cómo se comporta sin cliente cuando el bridge no está instalado y qué se exige a sus fechas.

## Requirements

### Requirement: El nombre es único entre los no eliminados
El nombre de un proyecto MUST ser único entre los registros no eliminados, incluidos los archivados,
y MUST poder reutilizarse cuando el registro que lo usaba está en la papelera.

#### Scenario: Un proyecto archivado conserva su nombre frente a uno nuevo
- **WHEN** se crea un proyecto cuyo nombre coincide con el de un proyecto archivado
- **THEN** la creación se rechaza por conflicto de nombre

### Requirement: El borrado se bloquea en cuanto hay epics
Un proyecto con epics, incluidos los que están en la papelera, MUST NOT poder pasar a la papelera ni
ser eliminado de forma definitiva. La guarda MUST contar los epics eliminados. Archivar un proyecto
MUST NOT archivar sus epics.

#### Scenario: Se archiva un proyecto que tiene epics activos
- **WHEN** un proyecto con epics se archiva
- **THEN** sus epics permanecen en el listado de epics activos

### Requirement: Las fechas se validan entre sí
Un proyecto MUST tener fecha de inicio; su fecha de fin es opcional y, cuando existe, MUST NOT preceder
al inicio. El nombre MUST tener una longitud mínima.

#### Scenario: Un proyecto termina antes de empezar
- **WHEN** se envía un proyecto con fecha de fin anterior a su fecha de inicio
- **THEN** la validación rechaza la fecha de fin

### Requirement: Sin cliente, el proyecto sigue siendo válido
Todo proyecto MUST tener cliente cuando el bridge está instalado, y la regla que lo exige MUST aportar
el bridge, no el paquete base. El base MUST NOT declarar ni asumir esa columna: un proyecto guardado
sin ella MUST seguir siendo legible. El selector MUST exigir un cliente activo salvo al editar el
proyecto que ya lo tiene.

#### Scenario: Se lee un proyecto sin cliente
- **WHEN** una fila existente no tiene cliente porque el bridge se instaló después
- **THEN** aparece en su listado sin cliente en lugar de desaparecer del listado