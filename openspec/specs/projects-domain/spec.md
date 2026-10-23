# Projects

## Purpose
Garantiza los invariantes del recurso proyecto que no se deducen de su tabla: qué lo hace único,
cómo se comporta sin cliente cuando el bridge no está instalado y qué se exige a sus fechas.

## Requirements

### Requirement: El nombre es único entre los no eliminados
El nombre de un proyecto MUST ser único entre los registros no eliminados, incluidos los archivados,
y MUST poder reutilizarse cuando el registro que lo usaba está en la papelera.

#### Scenario: Un proyecto archivado conserva su nombre frente a uno nuevo
- **WHEN** se crea un proyecto cuyo nombre coincide con el de un proyecto archivado
- **THEN** la creación se rechaza por conflicto de nombre

### Requirement: El borrado se bloquea en cuanto hay epics
Un proyecto con epics, incluidos los que están en la papelera, MUST NOT poder pasar a la papelera ni
ser eliminado de forma definitiva. La guarda MUST contar los epics eliminados. Archivar un proyecto
MUST NOT archivar sus epics.

#### Scenario: Se archiva un proyecto que tiene epics activos
- **WHEN** un proyecto con epics se archiva
- **THEN** sus epics permanecen en el listado de epics activos

### Requirement: Las fechas se validan entre sí
Un proyecto MUST tener fecha de inicio; su fecha de fin es opcional y, cuando existe, MUST NOT preceder
al inicio. El nombre MUST tener una longitud mínima.

#### Scenario: Un proyecto termina antes de empezar
- **WHEN** se envía un proyecto con fecha de fin anterior a su fecha de inicio
- **THEN** la validación rechaza la fecha de fin

### Requirement: Sin cliente, el proyecto sigue siendo válido
Todo proyecto MUST tener cliente cuando el bridge está instalado, y la regla que lo exige MUST aportar
el bridge, no el paquete base. El base MUST NOT declarar ni asumir esa columna: un proyecto guardado
sin ella MUST seguir siendo legible. El selector MUST exigir un cliente activo salvo al editar el
proyecto que ya lo tiene.

#### Scenario: Se lee un proyecto sin cliente
- **WHEN** una fila existente no tiene cliente porque el bridge se instaló después
- **THEN** aparece en su listado sin cliente en lugar de desaparecer del listado

### Requirement: Los filtros de epics y proyectos no repiten el contexto
El listado de epics MUST permitir filtrar por el nombre de su proyecto y MUST NOT ofrecer ese campo
cuando la lista ya está acotada a un proyecto. Las fechas de fin opcionales MUST poder filtrarse por
vacías.

#### Scenario: Se abre la búsqueda avanzada de los epics de un proyecto
- **WHEN** la lista de epics se abre desde un proyecto
- **THEN** el modal no ofrece el campo de proyecto

### Requirement: Los proyectos se autorizan por permisos de su módulo
Cada acción sobre proyectos MUST exigir `projects.{acción}`, y planning y timeline MUST exigir
`projects.view`. Al instalarse, el paquete MUST añadir `projects.manage` al rol de sistema `admin`.

#### Scenario: Sin permiso de vista
- **WHEN** un usuario sin `projects.view` abre el listado de proyectos o el planning
- **THEN** recibe 403

#### Scenario: Borrado definitivo sin permiso
- **WHEN** un usuario con `projects.delete` pero sin `projects.force_delete` vacía un proyecto de la papelera
- **THEN** recibe 403 y el proyecto sigue en la papelera

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
