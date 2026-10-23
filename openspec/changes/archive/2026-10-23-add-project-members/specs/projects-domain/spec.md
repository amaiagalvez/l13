## ADDED Requirements

### Requirement: Sin projects.manage solo se ven los proyectos de los que se es miembro
Quien crea un proyecto MUST quedar como su miembro `owner`, y un usuario MUST NOT aparecer dos veces en
el mismo proyecto. Sin `projects.manage`, un usuario MUST ver solo los proyectos de los que es miembro,
junto con sus epics, en los listados, el planning, el timeline y las cifras del panel de inicio.

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

### Requirement: El rol en el proyecto limita también sus epics
Sin `projects.manage`, editar o comentar un epic MUST exigir `epics.edit` y ser `owner` o `editor` de su
proyecto, y borrarlo MUST exigir `epics.delete` y ser `owner`. Un `viewer` MUST solo leer los epics.

#### Scenario: Viewer comenta
- **WHEN** un miembro `viewer` con `epics.edit` añade un comentario a un epic del proyecto
- **THEN** recibe 403 y no se crea el comentario

### Requirement: El owner elige miembros sin users.view
Quien puede gestionar los miembros de un proyecto MUST poder buscar usuarios activos para añadirlos,
aunque no tenga `users.view`. La búsqueda MUST devolver solo el identificador y el nombre, y MUST NOT
incluir al super-admin.

#### Scenario: Owner sin users.view
- **WHEN** el `owner` de un proyecto sin `users.view` busca usuarios para añadir
- **THEN** recibe los usuarios activos, sin el super-admin
