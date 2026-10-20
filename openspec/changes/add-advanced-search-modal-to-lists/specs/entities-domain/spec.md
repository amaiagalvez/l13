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

## MODIFIED Requirements

### Requirement: El tipo de cliente tiene nombre en 4 idiomas
El tipo de cliente MUST almacenar su nombre en euskera (`name_eu`), español (`name_es`), francés (`name_fr`) e inglés (`name_en`). El euskera es obligatorio; los demás son opcionales pero se validan igual si se envían. **Cada campo de nombre es filtrable independientemente en la búsqueda avanzada, agrupados bajo grupo "Nombres".**

#### Scenario: Se filtra por nombre en francés
- **WHEN** en búsqueda avanzada se usa campo "Nombre (fr)", operador "contiene", valor "Entre"
- **THEN** se encuentran tipos con `name_fr LIKE '%Entre%'`

### Requirement: CRUD canónico con papelera, archivado y selectores
El tipo de cliente sigue el patrón canónico: Controller + TrashController + ArchivedController + ListQuery + SelectOptionsQuery + Transformer + FormRequest + Policy + Factory + Vistas Flux + Tests. Los selectores solo devuelven activos. **El listado (activos, archivados, papelera) soporta búsqueda avanzada además de la simple, y las opciones del select `entity_type_ids` en entidades respetan el scope (active/archived/trashed).**