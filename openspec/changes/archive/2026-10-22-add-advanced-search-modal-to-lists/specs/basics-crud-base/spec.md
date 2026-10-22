## ADDED Requirements

### Requirement: Los campos filtrables se declaran en la query y dependen del scope
Cada query de listado MUST declarar sus campos filtrables en `searchableFields($scope)`, y el scope
(activos, archivados o papelera) MUST llegar a la condición de visibilidad y a las opciones de cada
campo. Un campo `select` MUST ser múltiple salvo que declare lo contrario. El campo `active` MUST
ofrecerse solo en la papelera, porque las otras dos listas ya contienen un único estado.

#### Scenario: Se abren los filtros en la lista de archivados
- **WHEN** se piden los campos filtrables de un recurso con flag `active` para el scope de archivados
- **THEN** no se incluye `active`, y en la papelera sí

### Requirement: Búsqueda simple y filtros avanzados se combinan con AND
Los filtros avanzados MUST aplicarse sobre la consulta ya acotada al scope y antes de paginar, unidos
con AND entre sí y con la búsqueda simple. Un `select` múltiple MUST filtrar por pertenencia: `equals`
encuentra cualquiera de los valores elegidos y `not_equals` ninguno de ellos.

#### Scenario: Se busca un término y se filtra por estado en la papelera
- **WHEN** la búsqueda simple es "Acme" y el filtro avanzado pide inactivos
- **THEN** solo salen los registros que cumplen las dos condiciones

## MODIFIED Requirements

### Requirement: Búsqueda y selectores tratan la entrada como no confiable
Un término de búsqueda MUST llegar escapado con un carácter de escape declarado, para que sus
comodines se busquen literalmente; los valores de los filtros avanzados MUST tratarse igual. Un
endpoint de opciones MUST devolver solo activos, con orden estable y el mismo tope que la paginación.

#### Scenario: Se busca un nombre con un guion bajo
- **WHEN** el término contiene el comodín `_`
- **THEN** se busca como carácter literal y no como comodín

#### Scenario: Un filtro avanzado de texto contiene comodines
- **WHEN** el valor del filtro contiene `%` o `_`
- **THEN** se busca como carácter literal y no como comodín
