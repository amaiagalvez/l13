# Entities

## Purpose
Garantiza los invariantes del recurso cliente que no se deducen de su tabla: el alcance real de la
unicidad del nombre, quién bloquea su eliminación y qué ofrecen los selectores que lo eligen.

## Requirements

### Requirement: El nombre es único entre los no eliminados
El nombre MUST ser único entre los registros no eliminados, incluidos los archivados, y MUST poder
reutilizarse cuando el registro que lo usaba está en la papelera.

#### Scenario: Un cliente archivado conserva su nombre frente a uno nuevo
- **WHEN** se crea un cliente con el nombre de otro, ya archivado o en la papelera
- **THEN** el archivado se rechaza por conflicto y el de la papelera ofrece crear o restaurar

### Requirement: El borrado se bloquea en cuanto hay proyectos
Un cliente con proyectos, incluidos los de la papelera, MUST NOT pasar a la papelera ni ser eliminado de
forma definitiva. La guarda MUST contar los proyectos eliminados.

#### Scenario: Un cliente tiene un proyecto eliminado
- **WHEN** se intenta borrar un cliente cuyo único proyecto está en la papelera
- **THEN** el borrado se rechaza y se informa de la dependencia

### Requirement: El flag de archivado no se propaga
Archivar un cliente MUST NOT archivar sus proyectos ni sus epics: siguen siendo trabajo activo y MUST
seguir en los listados de activos y en los tableros.

#### Scenario: Se archiva un cliente que tiene proyectos activos
- **WHEN** un cliente con proyectos se archiva
- **THEN** sus proyectos permanecen en el listado de activos

### Requirement: Los selectores solo ofrecen clientes activos
El endpoint de opciones MUST devolver solo clientes activos, y la validación MUST exigir uno activo
salvo al editar el registro que ya lo tiene.

#### Scenario: Se edita un registro cuyo cliente fue archivado
- **WHEN** se abre el formulario con el cliente asignado archivado
- **THEN** la selección sigue siendo válida y no se ofrece para elegir otro

### Requirement: La fila resume lo que cuelga del cliente
La fila MUST ofrecer borrar mientras no haya proyectos y MUST cambiarla por archivar en cuanto haya
alguno.

#### Scenario: El cliente pasa a tener un proyecto
- **WHEN** se asigna un proyecto a un cliente que no tenía ninguno
- **THEN** su fila pasa de ofrecer borrar a ofrecer archivar

### Requirement: El filtro por tipos mira todos los tipos del cliente
En el listado de clientes, filtrar por tipos con `equals` MUST devolver los clientes con al menos uno
de los tipos elegidos, y con `not_equals` los que no tienen ninguno. El listado de tipos de entidad
MUST NOT ofrecer búsqueda avanzada: es un catálogo corto y la búsqueda simple basta.

#### Scenario: Se excluye un tipo que el cliente comparte con otro
- **WHEN** se filtra por "no es igual a" un tipo y un cliente tiene ese tipo y otro distinto
- **THEN** el cliente no aparece en la lista

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
