# Basics: soporte compartido

## Purpose
Garantiza que los helpers compartidos se comporten igual en todas partes y que su uso no dependa de
que quien llama recuerde una secuencia concreta.

## Requirements

### Requirement: Un borrado con guardas va en transacción
Un modelo con guardas MUST envolver su propio borrado en una transacción en su conexión, para que los
bloqueos de las guardas sigan vigentes hasta que el borrado se escribe.

#### Scenario: Un hijo aparece entre la comprobación y el borrado
- **WHEN** un hijo se crea tras la comprobación de la guarda
- **THEN** el borrado espera al bloqueo y falla, sin dejar filas huérfanas

### Requirement: Quien bloquea no vuelve a bloquear
La guarda MUST tomar ella misma el bloqueo de fila, y quien llama MUST NOT bloquearla antes, porque un
segundo bloqueo no añade garantía y hace divergir la ruta entre recursos.

#### Scenario: Un controller borra un registro protegido
- **WHEN** un endpoint borra un recurso con guardas
- **THEN** solo la guarda bloquea y el controller solo informa

### Requirement: La carrera de nombre único se reporta como validación
Una violación del índice llegada tras validar MUST convertirse en error de validación sobre el campo;
cualquier otro error MUST relanzarse sin traducir.

#### Scenario: Inserción duplicada que pasó la validación
- **WHEN** una escritura choca con el índice de unicidad
- **THEN** el usuario ve un error de validación y no se crea nada

### Requirement: La auditoría solo escribe cuando hay actor
Las columnas de autoría MUST quedar en blanco sin un actor resuelto, para que un comando no borre la
trazabilidad de quien creó la fila, y MUST NOT ser asignables en masivo.

#### Scenario: Un seeder crea un registro sin usuario
- **WHEN** se crea una fila fuera de una petición autenticada
- **THEN** sus columnas quedan nulas y no sobrescriben las preexistentes

### Requirement: Pistas y límites salen de las reglas
Las ayudas de un campo MUST derivarse de las reglas del request y los límites MUST ser el mismo valor
que valida el servidor y declara el `maxlength` del navegador.

#### Scenario: El navegador limita el campo más que el servidor
- **WHEN** el `maxlength` del formulario y la regla `max` divergen
- **THEN** la entrada se rechaza como inválida del cliente