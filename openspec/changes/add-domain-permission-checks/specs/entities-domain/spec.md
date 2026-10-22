## ADDED Requirements

### Requirement: Clientes y tipos se autorizan por permisos de su módulo
Cada acción sobre clientes MUST exigir `entities.{acción}` y cada acción sobre tipos de entidad
`entity_types.{acción}`. Al instalarse, el paquete MUST añadir `entities.manage` y
`entity_types.manage` al rol de sistema `admin`.

#### Scenario: Sin permiso de vista
- **WHEN** un usuario sin `entities.view` abre el listado de clientes
- **THEN** recibe 403

#### Scenario: Solo lectura
- **WHEN** un usuario con `entities.view` pero sin `entities.edit` envía una edición de cliente
- **THEN** recibe 403 y el cliente no cambia

#### Scenario: Instalación existente
- **WHEN** se migra una instalación cuyos usuarios tienen el rol `admin`
- **THEN** siguen gestionando clientes y tipos como antes
