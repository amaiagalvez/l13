## ADDED Requirements

### Requirement: El selector de cliente del proyecto se autoriza por el permiso del proyecto
Las opciones de cliente del formulario de proyecto MUST ofrecerse a quien tiene `projects.create` o
`projects.edit`, aunque no tenga `entities.view`, y MUST seguir limitadas a clientes activos. Sin
ninguno de esos permisos, MUST denegarse.

#### Scenario: Crear proyecto sin ver clientes
- **WHEN** un usuario con `projects.create` y sin `entities.view` abre el formulario de proyecto
- **THEN** el selector ofrece los clientes activos y puede guardar el proyecto

#### Scenario: Sin permisos de proyecto
- **WHEN** un usuario sin `projects.create` ni `projects.edit` pide las opciones de cliente
- **THEN** recibe 403
