## ADDED Requirements

### Requirement: La cabecera de listado ofrece búsqueda avanzada condicional
El componente de cabecera de listado (`page-header`) MUST incluir un botón "Búsqueda avanzada" que abra un modal con filtros por campo. El botón debe estar disponible en todos los listados (activos, archivados, papelera) y solo cuando el recurso tiene campos filtrables **visibles** (tras evaluar `visible` y `condition($scope)`).

#### Scenario: Se muestra el botón de búsqueda avanzada en un listado
- **WHEN** se renderiza la cabecera de un listado que define `searchableFields` con al menos un campo visible para el scope actual
- **THEN** aparece un botón "Búsqueda avanzada" junto al campo de búsqueda simple

#### Scenario: El botón no aparece si no hay campos filtrables visibles
- **WHEN** un recurso no define `searchableFields` o todos los campos tienen `visible=false` o `condition($scope)` que retorna false
- **THEN** el botón de búsqueda avanzada NO se renderiza

### Requirement: Modal de búsqueda avanzada reutilizable con grupos, orden y select type
El componente `advanced-search-modal` MUST ser un modal Flux que recibe `searchableFields` y emite `filters-applied` con los filtros elegidos. MUST soportar los tipos `text`, `date`, `boolean`, `number` y `select` con sus operadores, mostrarlos en una sola lista ordenada por `group` y por `order`, sin secciones plegables, y descartar en el servidor los que no cumplen `condition($scope)`; el toggle "Mostrar campos avanzados" (estado del componente Livewire, no de la URL) revela los `visible=false`.

#### Scenario: Se abre el modal y se muestran los campos configurados en orden
- **WHEN** el usuario clica "Búsqueda avanzada"
- **THEN** se abre un modal con los campos en una sola lista, ordenados por grupo (`basic`, `dates`, `status`, `relations`) y por `order`

#### Scenario: Operadores por tipo de campo
- **WHEN** el campo es de tipo `text`
- **THEN** los operadores disponibles son: contains, equals, not_equals, starts_with, ends_with
- **WHEN** el campo es de tipo `date`
- **THEN** los operadores disponibles son: equals, before, after, between, is_empty
- **WHEN** el campo es de tipo `boolean`
- **THEN** los operadores disponibles son: is_true, is_false
- **WHEN** el campo es de tipo `number`
- **THEN** los operadores disponibles son: equals, not_equals, greater_than, greater_than_or_equal, less_than, less_than_or_equal, between
- **WHEN** el campo es de tipo `select`
- **THEN** los operadores disponibles son: equals, not_equals, is_empty (si nullable), y el input es un select con `options` (multiple si `multiple=true`)

#### Scenario: Campo condicional no se muestra cuando condition retorna false
- **WHEN** un campo tiene `condition($scope)` que retorna false (ej. `entity_id` cuando la lista ya está filtrada por entidad)
- **THEN** el campo no aparece en el modal, tampoco al mostrar los campos avanzados

#### Scenario: Campo con visible=false no se muestra por defecto
- **WHEN** un campo tiene `visible=false` (ej. `notes`)
- **THEN** el campo no aparece en el modal salvo que el usuario active "Mostrar campos avanzados"

#### Scenario: Toggle "Mostrar campos avanzados" revela campos ocultos
- **WHEN** el usuario clica "Mostrar campos avanzados" en el modal
- **THEN** aparecen los campos con `visible=false` y el botón cambia a "Ocultar campos avanzados"

#### Scenario: Campo select renderiza options del callable con scope actual
- **WHEN** un campo `type=select` tiene `options` callable
- **THEN** el select se popula con las opciones retornadas por `options($scope)` donde scope es active/archived/trashed

#### Scenario: Se aplican los filtros y se cierra el modal
- **WHEN** el usuario configura filtros y clica "Aplicar"
- **THEN** el modal emite `filters-applied` con el array de filtros y se cierra

#### Scenario: Se limpian los filtros
- **WHEN** el usuario clica "Limpiar" en el modal
- **THEN** se resetean todos los campos y se emite `filters-applied` con array vacío

### Requirement: Los filtros avanzados persisten en la URL y badges en cabecera
Los filtros activos MUST serializarse en parámetros de query string (`filters[field][operator]=value`) para que el estado sea compartible y persistente al recargar. La búsqueda simple (`search`) y los filtros avanzados (`filters`) deben coexistir. **La cabecera muestra badges de filtros activos; el botón de eliminación (x) de cada uno es un enlace a la lista sin ese filtro, de vuelta a la primera página.**

#### Scenario: Se aplican filtros y la URL se actualiza
- **WHEN** se emite `filters-applied` con filtros
- **THEN** la URL incluye `filters[name][contains]=Acme&filters[start_date][after]=2024-01-01`

#### Scenario: Se recarga la página con filtros en la URL
- **WHEN** se carga una URL con parámetros `filters`
- **THEN** el modal se inicializa con esos filtros y la lista muestra los resultados filtrados

#### Scenario: Badge de filtro activo aparece en cabecera con botón eliminar
- **WHEN** hay filtros activos en la URL
- **THEN** la cabecera muestra badges tipo "Nombre: contiene Acme" con botón (x) que enlaza a la lista sin ese filtro

#### Scenario: Toggle "Mostrar campos avanzados" es estado local, no URL
- **WHEN** el usuario activa "Mostrar campos avanzados" y recarga la página
- **THEN** el modal se abre con campos avanzados ocultos (estado no persistido)

## MODIFIED Requirements

### Requirement: El esqueleto de página y de tabla no se duplica
Todo el esqueleto de un listado MUST venir de los componentes compartidos: encabezado, búsqueda, pestañas, tabla, paginación, mensajes de vacío y acciones de fila. **El encabezado ahora incluye opcionalmente el botón de búsqueda avanzada y badges de filtros activos.**

#### Scenario: Se añade un recurso nuevo
- **WHEN** un dominio incorpora un listado más
- **THEN** reutiliza los componentes y solo declara columnas y payload

#### Scenario: Se añade un recurso nuevo con búsqueda avanzada
- **WHEN** un dominio incorpora un listado más con `searchableFields`
- **THEN** reutiliza los componentes compartidos y solo declara columnas, payload y campos filtrables