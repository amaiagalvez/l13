## ADDED Requirements

### Requirement: Los usuarios se clasifican en tipos
`basics13` MUST ofrecer el recurso canónico de tipos de usuario (listar, crear, editar, archivar y papelera) con nombre único entre activos. Un usuario MAY tener varios tipos y, en cada uno, MAY estar marcado como admin. Con `users.edit` se asignan tipos y se marca admin. Con solo `users.manage_own` el actor MUST añadir o quitar únicamente tipos que administra, MUST NOT tocar `is_admin` ni otros tipos, y el usuario MUST conservar al menos uno de esos tipos. Solo los tipos activos se ofrecen y dan alcance; un usuario sin tipos solo es alcanzable con `users.*` global.

#### Scenario: Usuario con varios tipos
- **WHEN** un gestor con `users.edit` asigna dos tipos a un usuario y lo marca admin en uno
- **THEN** el usuario queda unido a ambos y es admin solo del marcado

### Requirement: Un tipo en uso no se elimina
Un tipo de usuario unido a alguna cuenta, aunque esté en la papelera, MUST NOT poder enviarse a la papelera ni eliminarse definitivamente. Eliminar definitivamente un usuario MUST liberar sus uniones.

#### Scenario: Borrar un tipo con usuarios
- **WHEN** se intenta eliminar un tipo que tiene un usuario en la papelera
- **THEN** se rechaza y el tipo sigue activo

### Requirement: La gestión de usuarios se delega por tipo
Los "usuarios de" un actor MUST ser las cuentas, de cualquier estado, que comparten un tipo activo donde el actor es admin, excluidos el propio actor, los admins de un tipo compartido y quien tenga `users.manage` o `users.edit`. `users.view_own` MUST permitir solo verlos; `users.manage_own` MUST permitir sobre ellos las acciones de `users.*` salvo suplantar, y asignar solo roles cuyos permisos tiene el actor. El alcance MUST aplicarse a listados, selector y resumen de permisos. La regla del super-admin MUST prevalecer.

#### Scenario: Admin de tipo edita a un usuario de su tipo
- **WHEN** un actor con `users.manage_own`, admin del tipo A, edita a un usuario del tipo A
- **THEN** se permite, con roles dentro de sus propios permisos

#### Scenario: Escalada de privilegios
- **WHEN** ese actor asigna un rol con un permiso que él no tiene, o edita a otro admin del tipo A
- **THEN** se deniega

#### Scenario: Fuera de alcance
- **WHEN** el usuario solo tiene el tipo B, o el tipo A está archivado
- **THEN** no aparece en sus listados ni selector y la acción se deniega

#### Scenario: Solo lectura
- **WHEN** un actor con solo `users.view_own` intenta archivar a uno de sus usuarios
- **THEN** se deniega
