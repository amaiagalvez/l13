## ADDED Requirements

### Requirement: Listado de entidades soporta búsqueda avanzada con select multiple
El listado de entidades (`EntityListQuery`) MUST exponer campos filtrables: `name` (text), `notes` (text, visible=false), `active` (boolean), `created_at` (date), `entity_type_ids` (select, multiple=true, options=callable returning active entity types per scope). La vista `entities/list.blade.php` MUST incluir el modal de búsqueda avanzada y pasar la configuración desde el query.

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

### Requirement: Listado de tipos de entidad soporta búsqueda avanzada multilingüe
El listado de tipos de entidad (`EntityTypeListQuery`) MUST exponer campos filtrables: `name_eu` (text), `name_es` (text), `name_fr` (text), `name_en` (text) agrupados en group=`names`, `notes` (text, visible=false), `active` (boolean), `created_at` (date). La vista `entity-types/list.blade.php` MUST incluir el modal.

#### Scenario: Usuario filtra tipos por nombre en euskera
- **WHEN** en el modal se selecciona "Nombre (eu)", operador "empieza por", valor "Enpr"
- **THEN** la lista muestra tipos con `name_eu LIKE 'Enpr%'`

#### Scenario: Usuario filtra tipos por estado archivado
- **WHEN** en el modal se selecciona "Activo", operador "es falso"
- **THEN** la lista muestra solo tipos archivados (active=false)

#### Scenario: Usuario filtra tipos por nombre en cualquier idioma
- **WHEN** en el modal se usa grupo "Nombres" con campo "Nombre (fr)", operador "contiene", valor "Entre"
- **THEN** se encuentran tipos con `name_fr LIKE '%Entre%'`
