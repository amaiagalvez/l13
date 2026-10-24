# basics-navigation-menus Specification

## Purpose

Menús de navegación que siembra cada paquete en la base de datos y cómo se decide qué ítems ve cada usuario según sus permisos.

## Requirements

### Requirement: Cada paquete siembra sus menús de forma idempotente
Cada ítem MUST tener una clave única, un paquete dueño, una etiqueta como clave de traducción y un orden, y MAY tener un grupo padre (dos niveles como máximo), icono, ruta con nombre y permisos. Cada paquete MUST sembrar solo su grupo y los ítems de sus propias rutas. Sembrar MUST ser idempotente y MUST dejar en cada ítem exactamente los permisos sembrados. Un permiso inexistente MUST hacer fallar la siembra. La interfaz MUST NOT gestionar menús.

#### Scenario: Siembra repetida
- **WHEN** las migraciones de menús se ejecutan dos veces
- **THEN** no hay ítems ni permisos de ítem duplicados

#### Scenario: Permiso retirado en una versión nueva
- **WHEN** una siembra posterior quita un permiso a un ítem
- **THEN** el ítem deja de tener ese permiso

#### Scenario: Permiso inexistente
- **WHEN** una siembra asigna a un ítem un slug que no existe
- **THEN** la migración falla y no deja el ítem sin permisos

### Requirement: Un usuario ve un ítem si tiene alguno de sus permisos
Un ítem sin permisos MUST verse por cualquier usuario autenticado. Un ítem con permisos MUST verse solo si el usuario tiene al menos uno, resuelto como en `basics-roles-permissions`, incluido `{módulo}.manage`. El super-admin MUST ver todos los ítems. Un ítem cuya ruta no existe MUST NOT mostrarse, y un grupo sin hijos visibles tampoco. Los ítems MUST ordenarse por su orden dentro de cada nivel.

#### Scenario: Gestión delegada de usuarios
- **WHEN** un usuario solo tiene `users.view_own`
- **THEN** ve el ítem "Users" y no ve "Roles"

#### Scenario: manage concede el ítem
- **WHEN** un usuario solo tiene `roles.manage`
- **THEN** ve el ítem "Roles"

#### Scenario: Grupo vacío
- **WHEN** un usuario no tiene ningún permiso de administración, o ninguna ruta de un grupo está registrada
- **THEN** ese grupo no aparece

#### Scenario: Permiso retirado
- **WHEN** se archiva el único rol que le daba `roles.view`
- **THEN** en la petición siguiente ya no ve el ítem "Roles"

### Requirement: Pintar el menú no añade consultas
Pintar el menú MUST NOT ejecutar más consultas que las de los permisos del usuario, una vez sembrados los menús. Las etiquetas MUST mostrarse en el idioma activo.

#### Scenario: Recarga
- **WHEN** un usuario cuyos permisos ya están cargados recarga una página
- **THEN** pintar el menú no ejecuta consultas
