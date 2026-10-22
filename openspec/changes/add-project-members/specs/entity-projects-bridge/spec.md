## ADDED Requirements

### Requirement: Sin entities.view, el selector de cliente se limita a los de tus proyectos
Quien no tiene `entities.view` MUST recibir como opciones de cliente del formulario de proyecto solo los
clientes activos de los proyectos de los que es miembro, y la validación MUST rechazar cualquier otro.
Con `entities.view`, el selector MUST ofrecer todos los clientes activos.

#### Scenario: Opciones acotadas
- **WHEN** un usuario sin `entities.view` abre el formulario de proyecto
- **THEN** solo ve los clientes de sus proyectos

#### Scenario: Cliente ajeno enviado
- **WHEN** ese usuario envía un cliente que no está en sus proyectos
- **THEN** la validación lo rechaza en el campo cliente
