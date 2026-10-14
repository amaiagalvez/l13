# Basics: papelera y estado archivado

## Purpose
Garantiza que la papelera, la restauración con conflicto de nombre y el estado archivado se
comporten igual en los tres dominios y no se puedan eludir desde un endpoint sin validar.

## Requirements

### Requirement: Restaurar exige su propio Form Request
Cada verb de la papelera MUST pasar por un Form Request propio que resuelva el registro eliminado y
autorice su habilidad, sin delegar el conflicto en el controller.

#### Scenario: Se llama a restaurar sin la habilidad
- **WHEN** un usuario sin `restore` llama al endpoint
- **THEN** se rechaza antes de tocar el registro, que sigue eliminado

### Requirement: La precomprobación no sustituye al índice
La restauración MUST tratar la comprobación previa del nombre como vía rápida y MUST seguir atrapando
la violación de unicidad, que es lo único que ve un nombre insertado a mitad.

#### Scenario: El nombre aparece entre la precomprobación y el restore
- **WHEN** un nombre activo aparece mientras se restaura otro con el mismo
- **THEN** la restauración se rechaza y el registro sigue eliminado

### Requirement: Restaurar nunca reporta un éxito que no ocurrió
La restauración MUST verificar que el registro sigue presente y fuera de la papelera antes de informar
de éxito, porque una escritura puede no afectar a ninguna fila.

#### Scenario: Restore concurrente con borrado definitivo
- **WHEN** una petición borra el registro mientras otra lo restaura
- **THEN** se informa de que ya no está en la papelera

### Requirement: El borrado definitivo se bloquea con relaciones
El borrado definitivo MUST ir en transacción con el bloqueo de fila y MUST rechazarse si hay filas que
lo referencian. Ese bloqueo no dice nada sobre la unicidad de nombres.

#### Scenario: Borrar definitivamente un registro con hijos
- **WHEN** se intenta borrar un registro que tiene referencias
- **THEN** la operación se rechaza y se informa de la dependencia

### Requirement: El flag de estado es reversible y auditado
El flag MUST alternar entre archivado y activo sin pasar por la papelera, ambos extremos MUST quedar
auditados, y MUST NOT ser asignable en masivo.

#### Scenario: Se restaura un registro que estaba archivado
- **WHEN** un registro archivado pasa por la papelera y se restaura
- **THEN** reaparece en el listado de archivados, auditado