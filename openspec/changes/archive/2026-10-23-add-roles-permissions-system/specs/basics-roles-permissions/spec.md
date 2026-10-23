## Purpose

Roles dinámicos con permisos que siembra cada paquete, cómo se resuelve si un usuario tiene un permiso
y quién puede asignar roles.

## ADDED Requirements

### Requirement: Los roles son recursos canónicos con slug único entre los no eliminados
Los roles MUST gestionarse como el resto de recursos: listado, alta, edición, archivados y papelera.
Su `slug` MUST ser único entre los roles no eliminados, incluidos los archivados, y MUST estar
respaldado por un índice. Un rol con usuarios asignados MUST NOT pasar a la papelera ni borrarse
definitivamente. Un rol de sistema MUST NOT archivarse, ir a la papelera ni borrarse, y solo el
super-admin MAY editarlo. Ser de sistema MUST NOT poder fijarse desde ninguna petición.

#### Scenario: Slug de un rol archivado
- **WHEN** se crea un rol con el slug de otro que está archivado
- **THEN** se rechaza con error en el campo slug

#### Scenario: Rol con usuarios
- **WHEN** se intenta mover a la papelera un rol asignado a algún usuario
- **THEN** se rechaza y la fila ofrece archivar en su lugar

#### Scenario: Rol de sistema
- **WHEN** un usuario con `roles.manage` que no es el super-admin intenta editar o archivar el rol `admin`
- **THEN** se deniega

### Requirement: Los permisos se siembran por paquete y son de solo lectura
Cada permiso MUST tener el slug `{módulo}.{acción}`, con la acción en `view`, `create`, `edit`,
`delete`, `archive`, `restore`, `force_delete` o `manage`. Cada paquete MUST sembrar solo los permisos
de los módulos que comprueban sus policies; `basics13` MUST sembrar solo `users` y `roles`. Sembrar
MUST ser idempotente y MUST NOT quitar permisos ya asignados a roles. La interfaz MUST NOT crear,
editar ni borrar permisos: solo los lista agrupados por módulo y los asigna a roles.

#### Scenario: Siembra repetida
- **WHEN** la siembra de permisos se ejecuta dos veces
- **THEN** no hay permisos duplicados y los roles conservan los suyos

### Requirement: Un usuario tiene un permiso si un rol activo suyo lo concede
`{módulo}.{acción}` MUST concederse si algún rol activo y no eliminado del usuario tiene ese permiso o
`{módulo}.manage`. Los roles archivados o eliminados MUST NOT conceder nada. Los permisos de un usuario
MUST consultarse una sola vez por petición, y un cambio de roles o de permisos MUST verse en la
petición siguiente.

#### Scenario: manage concede todas las acciones del módulo
- **WHEN** el usuario tiene un rol con `roles.manage`
- **THEN** tiene `roles.edit` y `roles.force_delete`

#### Scenario: Rol archivado
- **WHEN** se archiva el único rol que le daba `users.edit`
- **THEN** en la petición siguiente ya no tiene `users.edit`

### Requirement: Asignar roles exige roles.manage y nunca sobre uno mismo
Asignar o quitar roles MUST exigir `roles.manage`. Un usuario MUST NOT cambiar sus propios roles. Los
roles de sistema MUST asignarlos y quitarlos solo el super-admin. Cada asignación MUST registrar quién
la hizo y cuándo.

#### Scenario: Autoasignación
- **WHEN** un usuario con `users.edit` y `roles.manage` añade un rol a su propia cuenta
- **THEN** se rechaza y sus roles no cambian

#### Scenario: Rol de sistema asignado por quien no es super-admin
- **WHEN** un usuario con `roles.manage` asigna el rol `admin` a otra cuenta
- **THEN** se rechaza

### Requirement: El resumen muestra los permisos efectivos de cada usuario
El resumen MUST exigir `roles.view` y MUST mostrar, por usuario, sus roles activos y sus permisos
efectivos agrupados por los módulos que existan en datos. `manage` MUST mostrarse como acceso total al
módulo. La exportación CSV MUST contener exactamente las filas filtradas.

#### Scenario: Exportación filtrada
- **WHEN** se filtra el resumen por un rol y se exporta
- **THEN** el CSV contiene solo los usuarios con ese rol
