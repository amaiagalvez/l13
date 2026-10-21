## ADDED Requirements

### Requirement: Listado de proyectos soporta búsqueda avanzada con select condicional multiple
El listado de proyectos (`ProjectListQuery`) MUST exponer campos filtrables: `name` (text), `notes` (text, visible=false), `start_date` (date), `end_date` (date, nullable), `active` (boolean), `created_at` (date), `entity_id` (select, nullable, **multiple=true**, options=callable returning active entities per scope, condition=hide when scoped to entity). La vista `projects/list.blade.php` MUST incluir el modal de búsqueda avanzada y pasar la configuración desde el query.

#### Scenario: Usuario filtra proyectos por rango de fechas
- **WHEN** en el modal se selecciona "Fecha inicio", operador "entre", valores "2024-01-01" y "2024-06-30"
- **THEN** la lista muestra proyectos con `start_date BETWEEN '2024-01-01' AND '2024-06-30'`

#### Scenario: Usuario filtra proyectos sin entidad asociada
- **WHEN** en el modal se selecciona "Entidad", operador "está vacío"
- **THEN** la lista muestra proyectos con `entity_id IS NULL`

#### Scenario: Se filtra por entidad asociada
- **WHEN** en búsqueda avanzada se usa campo "Entidad", operador "es igual a", valor "7"
- **THEN** se encuentran proyectos con `entity_id = 7`

#### Scenario: Campo entidad oculto cuando lista está scopeada por entidad
- **WHEN** se accede a lista de proyectos desde una entidad (route tiene parámetro entity)
- **THEN** el campo "Entidad" no aparece en el modal (condition retorna false)

#### Scenario: Usuario filtra proyectos por nombre y estado
- **WHEN** búsqueda simple = "Alpha" Y filtro avanzado = active=true
- **THEN** la lista muestra proyectos activos con nombre que contiene "Alpha"

### Requirement: Listado de épicas soporta búsqueda avanzada con select condicional multiple
El listado de épicas (`EpicListQuery`) MUST exponer campos filtrables: `name` (text), `notes` (text, visible=false), `start_date` (date, nullable), `end_date` (date, nullable), `active` (boolean), `created_at` (date), `project_id` (select, **multiple=true**, options=callable returning active projects for current entity/scope, condition=hide when scoped to project). La vista `epics/list.blade.php` MUST incluir el modal.

#### Scenario: Usuario filtra épicas por proyecto específico
- **WHEN** en el modal se selecciona "Proyecto", operador "es igual a", valor "3"
- **THEN** la lista muestra solo épicas del proyecto con ID 3

#### Scenario: Campo proyecto oculto cuando lista está scopeada por proyecto
- **WHEN** se accede a lista de épicas desde un proyecto (route tiene parámetro project)
- **THEN** el campo "Proyecto" no aparece en el modal (condition retorna false)

#### Scenario: Usuario filtra épicas sin fecha de fin
- **WHEN** en el modal se selecciona "Fecha fin", operador "está vacío"
- **THEN** la lista muestra épicas con `end_date IS NULL`

#### Scenario: Usuario filtra épicas por rango de fechas de inicio
- **WHEN** en el modal se selecciona "Fecha inicio", operador "después de", valor "2024-07-01"
- **THEN** la lista muestra épicas con `start_date > '2024-07-01'`
