## ADDED Requirements

### Requirement: Listado de entidades soporta búsqueda avanzada con select multiple
El listado de entidades (`EntityListQuery`) MUST exponer campos filtrables: `name` (text), `notes` (text, visible=false), `active` (boolean, solo en el scope `trashed`), `created_at` (date), `entity_type_ids` (select, multiple=true, options=callable returning active entity types per scope). La vista `entities/list.blade.php` MUST incluir el modal de búsqueda avanzada y pasar la configuración desde el query.

#### Scenario: Usuario filtra entidades por nombre que contiene "Acme"
- **WHEN** en el modal de búsqueda avanzada se selecciona campo "Nombre", operador "contiene", valor "Acme"
- **THEN** la lista muestra solo entidades con `name LIKE '%Acme%'`

#### Scenario: Usuario filtra entidades por múltiples tipos
- **WHEN** en el modal se selecciona campo "Tipos", operador "es igual a", valores [3, 5, 7] (IDs de tipos)
- **THEN** la lista muestra solo entidades que tienen CUALQUIERA de los tipos 3, 5, 7 (IN query)

#### Scenario: Usuario filtra entidades excluyendo tipos
- **WHEN** en el modal se selecciona campo "Tipos", operador "no es igual a", valores [2, 4]
- **THEN** la lista muestra entidades que NO tienen los tipos 2, 4 (NOT IN query)

#### Scenario: Usuario combina búsqueda simple y filtro avanzado
- **WHEN** búsqueda simple = "Tech" Y filtro avanzado = active=false
- **THEN** la lista muestra entidades con nombre que contiene "Tech" Y active=false

#### Scenario: Las opciones de tipos respetan el scope del listado
- **WHEN** se abre el modal desde el listado de activos, archivados o papelera
- **THEN** el select "Tipos" ofrece solo los tipos de entidad activos de ese scope

### Requirement: Listado de tipos de entidad mantiene solo la búsqueda simple
El listado de tipos de entidad MUST NOT ofrecer búsqueda avanzada: es una tabla corta de consulta y
la búsqueda simple sobre sus nombres basta.

#### Scenario: El listado de tipos de entidad no muestra la búsqueda avanzada
- **WHEN** se abre el listado de tipos de entidad
- **THEN** no aparece el botón "Búsqueda avanzada" y la búsqueda simple sigue funcionando
