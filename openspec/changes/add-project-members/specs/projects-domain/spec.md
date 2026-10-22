## ADDED Requirements

### Requirement: Sin projects.manage solo se ven los proyectos de los que se es miembro
Quien crea un proyecto MUST quedar como su miembro `owner`, y un usuario MUST NOT aparecer dos veces en
el mismo proyecto. Sin `projects.manage`, un usuario MUST ver solo los proyectos de los que es miembro,
junto con sus epics, en los listados, el planning y el timeline.

#### Scenario: Proyecto ajeno
- **WHEN** un usuario con `projects.view`, sin `projects.manage` y sin ser miembro abre un proyecto
- **THEN** recibe 404 y el proyecto no aparece en su listado

#### Scenario: Proyectos existentes
- **WHEN** se migra una instalación con proyectos
- **THEN** el creador de cada proyecto es su `owner`

### Requirement: El rol en el proyecto limita la edición y la gestión de miembros
Cada miembro MUST tener el rol `owner`, `editor` o `viewer`. Sin `projects.manage`, editar MUST exigir
ser `owner` o `editor`, y borrar o gestionar los miembros, ser `owner`. Un proyecto MUST NOT quedarse
sin `owner`.

#### Scenario: Miembro viewer
- **WHEN** un miembro `viewer` con `projects.edit` envía una edición del proyecto
- **THEN** recibe 403

#### Scenario: Último owner
- **WHEN** se quita o se degrada al único `owner` de un proyecto
- **THEN** se rechaza y el proyecto conserva su `owner`
