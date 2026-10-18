# Base CRUD

## Purpose
Garantiza que todo recurso de los tres dominios tenga la misma forma de entrada, consulta y
presentación, y que la lógica de negocio viva en la capa que le corresponde.

## Requirements

### Requirement: El controller es delgado y delegable
Un controller de recurso MUST NOT contener SQL, `Schema::`, `where` a mano ni `paginate()` a pelo.
MUST delegar la consulta en su query, dar forma a la fila en su transformer, y limitarse a autorizar,
delegar en el Form Request y reportar el resultado.

#### Scenario: Una acción necesita filtrar u ordenar
- **WHEN** un endpoint tiene que acotar un conjunto de registros
- **THEN** el criterio va en la clase de query, no en el controller

### Requirement: La unicidad de nombre se declara en dos capas
La unicidad del nombre entre los no eliminados MUST estar en las reglas del request, para el error en
el campo correcto, y MUST estar respaldada por un índice, para que la validación no sea la única
barrera. La regla MUST ignorar los eliminados y el registro editado.

#### Scenario: Dos peticiones validan el mismo nombre antes de escribir
- **WHEN** dos peticiones pasan la validación con el mismo nombre
- **THEN** la segunda falla contra el índice y se reporta como error en `name`

### Requirement: Búsqueda y selectores tratan la entrada como no confiable
Un término de búsqueda MUST llegar escapado con un carácter de escape declarado, para que sus
comodines se busquen literalmente. Un endpoint de opciones MUST devolver solo activos, con orden
estable y el mismo tope que la paginación.

#### Scenario: Se busca un nombre con un guion bajo
- **WHEN** el término contiene el comodín `_`
- **THEN** se busca como carácter literal y no como comodín