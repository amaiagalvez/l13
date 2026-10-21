## ADDED Requirements

### Requirement: Selectores cruzados sin permiso explícito (en bridge)
Para formularios que referencian recursos de otro dominio (ej: `projects.entity_id` → `entities`),
el sistema MUST NOT requerir permiso `*.select` explícito. En su lugar, las queries de opciones
(`EntitySelectOptionsQuery`) MUST vivir en el **bridge `entity-projects13`** y filtrar contextualmente:

#### Scenario: Selector de entidad en formulario de proyecto
- **WHEN** usuario con `projects.create` pero SIN `entities.view` abre formulario de proyecto
- **WHEN** `EntitySelectOptionsQuery` (en `entity-projects13`) resuelve opciones
- **THEN** retorna solo entidades de proyectos donde el usuario es owner o member
- **THEN** usuario NO ve entidades ajenas en el select
- **WHEN** usuario envía formulario con `entity_id` válido de sus proyectos
- **THEN** validación pasa (ProjectFormRequest verifica acceso via policy/query)
- **WHEN** usuario intenta `entity_id` de proyecto ajeno
- **THEN** validación falla: "La entidad seleccionada no es accesible"
