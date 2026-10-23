## ADDED Requirements

### Requirement: Los proyectos se autorizan por permisos de su módulo
Cada acción sobre proyectos MUST exigir `projects.{acción}`, y planning y timeline MUST exigir
`projects.view`. Al instalarse, el paquete MUST añadir `projects.manage` al rol de sistema `admin`.

#### Scenario: Sin permiso de vista
- **WHEN** un usuario sin `projects.view` abre el listado de proyectos o el planning
- **THEN** recibe 403

#### Scenario: Borrado definitivo sin permiso
- **WHEN** un usuario con `projects.delete` pero sin `projects.force_delete` vacía un proyecto de la papelera
- **THEN** recibe 403 y el proyecto sigue en la papelera
