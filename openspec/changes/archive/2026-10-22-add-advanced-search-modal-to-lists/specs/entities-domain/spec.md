## ADDED Requirements

### Requirement: El filtro por tipos mira todos los tipos del cliente
En el listado de clientes, filtrar por tipos con `equals` MUST devolver los clientes con al menos uno
de los tipos elegidos, y con `not_equals` los que no tienen ninguno. El listado de tipos de entidad
MUST NOT ofrecer búsqueda avanzada: es un catálogo corto y la búsqueda simple basta.

#### Scenario: Se excluye un tipo que el cliente comparte con otro
- **WHEN** se filtra por "no es igual a" un tipo y un cliente tiene ese tipo y otro distinto
- **THEN** el cliente no aparece en la lista
