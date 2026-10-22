## ADDED Requirements

### Requirement: Los filtros de epics y proyectos no repiten el contexto
El listado de epics MUST permitir filtrar por el nombre de su proyecto y MUST NOT ofrecer ese campo
cuando la lista ya está acotada a un proyecto. Las fechas de fin opcionales MUST poder filtrarse por
vacías.

#### Scenario: Se abre la búsqueda avanzada de los epics de un proyecto
- **WHEN** la lista de epics se abre desde un proyecto
- **THEN** el modal no ofrece el campo de proyecto
