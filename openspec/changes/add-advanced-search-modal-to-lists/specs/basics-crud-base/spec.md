## ADDED Requirements

### Requirement: ListQueryBase define campos filtrables con metadatos completos y scope-aware
Cada `ListQueryBase` concreto MUST implementar `searchableFields(string $scope): array`, que devuelve los campos filtrables con su columna, etiqueta, tipo, operadores, visibilidad, orden, grupo y nulabilidad. El `$scope` (`active`|`archived`|`trashed`) MUST llegar a los callables `condition` y `options`, para que visibilidad y opciones respeten el estado de la lista. Un campo `select` MUST ser de selección múltiple salvo que declare `multiple=false`.

#### Scenario: EntityListQuery define sus campos filtrables con configuración completa
- **WHEN** se llama `searchableFields('active')` en `EntityListQuery`
- **THEN** devuelve array con:
  - `name`: column=`ENT_entities.name`, type=`text`, operators=[contains, equals, not_equals, starts_with, ends_with], default_operator=`contains`, group=`basic`, order=10
  - `notes`: column=`ENT_entities.notes`, type=`text`, operators=[contains, equals, not_equals], default_operator=`contains`, group=`basic`, order=20, visible=`false`
  - `active`: column=`ENT_entities.active`, type=`boolean`, operators=[is_true, is_false], default_operator=`is_true`, group=`status`, order=10
  - `created_at`: column=`ENT_entities.created_at`, type=`date`, operators=[equals, before, after, between], default_operator=`between`, group=`dates`, order=10
  - `entity_type_ids`: column=`ENT_entity_entity_type.entity_type_id`, type=`select`, operators=[equals, not_equals], default_operator=`equals`, group=`relations`, order=10, options=callable returning active entity types for scope, multiple=`true`

#### Scenario: ProjectListQuery define sus campos filtrables con select scope-aware
- **WHEN** se llama `searchableFields('active')` en `ProjectListQuery`
- **THEN** devuelve array con:
  - `name`: column=`PRO_projects.name`, type=`text`, operators=[contains, equals, not_equals, starts_with, ends_with], default_operator=`contains`, group=`basic`, order=10
  - `notes`: column=`PRO_projects.notes`, type=`text`, operators=[contains, equals, not_equals], default_operator=`contains`, group=`basic`, order=20, visible=`false`
  - `start_date`: column=`PRO_projects.start_date`, type=`date`, operators=[equals, before, after, between], default_operator=`between`, group=`dates`, order=10, nullable=`false`
  - `end_date`: column=`PRO_projects.end_date`, type=`date`, operators=[equals, before, after, between, is_empty], default_operator=`between`, group=`dates`, order=20, nullable=`true`
  - `active`: column=`PRO_projects.active`, type=`boolean`, operators=[is_true, is_false], default_operator=`is_true`, group=`status`, order=10
  - `created_at`: column=`PRO_projects.created_at`, type=`date`, operators=[equals, before, after, between], default_operator=`between`, group=`dates`, order=30
  - `entity_id`: column=`PRO_projects.entity_id`, type=`select`, operators=[equals, not_equals, is_empty], default_operator=`equals`, group=`relations`, order=10, nullable=`true`, options=callable returning active entities for scope, condition=fn($scope) => !request()->route('entity'), **multiple=`true`**

### Requirement: ListQueryBase aplica filtros avanzados tipados incluyendo select
El método `paginate()` MUST aceptar un parámetro opcional `array $filters = []` y delegar en `applyFilters()` que traduce cada filtro a cláusulas SQL seguras (parameterizadas) según el tipo y operador. Los filtros se aplican sobre el query ya scopeado (tras `scoped()`), antes de paginar. Para `type=select` con `multiple=true`, el operador `equals` se traduce a `IN (...)` y `not_equals` a `NOT IN (...)`.

#### Scenario: Filtro de texto con operador "contains"
- **WHEN** `$filters = ['name' => ['operator' => 'contains', 'value' => 'Acme']]`
- **THEN** el query incluye `ENT_entities.name LIKE '%Acme%'` (con escape de comodines)

#### Scenario: Filtro de fecha con operador "between"
- **WHEN** `$filters = ['start_date' => ['operator' => 'between', 'value' => ['2024-01-01', '2024-12-31']]]`
- **THEN** el query incluye `PRO_projects.start_date BETWEEN '2024-01-01' AND '2024-12-31'`

#### Scenario: Filtro booleano con operador "is_false"
- **WHEN** `$filters = ['active' => ['operator' => 'is_false', 'value' => true]]`
- **THEN** el query incluye `PRO_projects.active = 0`

#### Scenario: Filtro select con operador "equals" y multiple=true
- **WHEN** `$filters = ['entity_type_ids' => ['operator' => 'equals', 'value' => [3, 5, 7]]]` y campo tiene `multiple=true`
- **THEN** el query incluye `ENT_entity_entity_type.entity_type_id IN (3, 5, 7)`

#### Scenario: Filtro select con operador "not_equals" y multiple=true
- **WHEN** `$filters = ['entity_type_ids' => ['operator' => 'not_equals', 'value' => [3, 5]]]` y campo tiene `multiple=true`
- **THEN** el query incluye `ENT_entity_entity_type.entity_type_id NOT IN (3, 5)`

#### Scenario: Filtro select con operador "is_empty" en campo nullable
- **WHEN** `$filters = ['entity_id' => ['operator' => 'is_empty', 'value' => true]]` y campo `nullable=true`
- **THEN** el query incluye `PRO_projects.entity_id IS NULL`

#### Scenario: Múltiples filtros se combinan con AND
- **WHEN** `$filters` tiene varios campos
- **THEN** todas las condiciones se unen con `AND` en el query

### Requirement: Búsqueda simple y filtros avanzados coexisten
Cuando tanto `$search` como `$filters` tienen valores, AMBOS MUST aplicarse: la búsqueda simple sobre sus columnas (`searchColumns`) y los filtros avanzados sobre sus campos respectivos, todo combinado con AND.

#### Scenario: Búsqueda simple "Acme" + filtro avanzado active=false
- **WHEN** `$search = 'Acme'` y `$filters = ['active' => ['operator' => 'is_false', 'value' => true]]`
- **THEN** el query busca `name LIKE '%Acme%'` Y `active = 0`

## MODIFIED Requirements

### Requirement: Búsqueda y selectores tratan la entrada como no confiable
Un término de búsqueda MUST llegar escapado con un carácter de escape declarado, para que sus comodines se busquen literalmente. **Los valores de filtros avanzados también se escapan según su tipo antes de interpolar en SQL.** Un endpoint de opciones MUST devolver solo activos, con orden estable y el mismo tope que la paginación.

#### Scenario: Se busca un nombre con un guion bajo
- **WHEN** el término contiene el comodín `_`
- **THEN** se busca como carácter literal y no como comodín

#### Scenario: Filtro avanzado de texto contiene comodines
- **WHEN** el valor del filtro contiene `%` o `_`
- **THEN** se escapan con el carácter de escape declarado antes de construir el LIKE