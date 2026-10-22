## ADDED Requirements

### Requirement: Epics y comentarios se autorizan por permisos de epics
Cada acción sobre epics MUST exigir `epics.{acción}`, y añadir un comentario MUST exigir `epics.edit`.
Al instalarse, el paquete MUST añadir `epics.manage` al rol de sistema `admin`.

#### Scenario: Comentario sin permiso de edición
- **WHEN** un usuario con `epics.view` pero sin `epics.edit` añade un comentario
- **THEN** recibe 403 y no se crea el comentario
