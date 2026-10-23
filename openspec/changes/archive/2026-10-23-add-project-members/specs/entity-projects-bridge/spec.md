## ADDED Requirements

### Requirement: Sin entities.view, al editar el selector de cliente se limita a los de tus proyectos
Quien no tiene `entities.view` MUST recibir, al crear un proyecto, todos los clientes activos, para que
pueda crear su primer proyecto. Al editar un proyecto MUST recibir solo los clientes activos de los
proyectos de los que es miembro, más el cliente actual del proyecto, y la validación MUST rechazar
cualquier otro. Con `entities.view`, el selector MUST ofrecer todos los clientes activos.

#### Scenario: Alta sin proyectos
- **WHEN** un usuario con `projects.create`, sin `entities.view` y sin proyectos abre el alta de proyecto
- **THEN** el selector ofrece todos los clientes activos y puede guardar el proyecto

#### Scenario: Opciones acotadas al editar
- **WHEN** un usuario sin `entities.view` edita uno de sus proyectos
- **THEN** solo ve los clientes de sus proyectos

#### Scenario: Cliente ajeno enviado al editar
- **WHEN** ese usuario envía en la edición un cliente que no está en sus proyectos
- **THEN** la validación lo rechaza en el campo cliente
