## ADDED Requirements

### Requirement: La gestión de cuentas de usuario es un recurso canónico del paquete base
La gestión de cuentas MUST vivir en `basics13` como recurso canónico (listar, crear, actualizar,
archivar y papelera) y MUST operar sobre el modelo de `auth.providers.users.model`, no sobre
`Basics13\Models\User`. Editar sin enviar `password` MUST conservar la contraseña actual.

#### Scenario: El host administra cuentas con su propio modelo
- **WHEN** el host configura `auth.providers.users.model` y gestiona usuarios
- **THEN** el listado, las altas y las bajas operan sobre ese modelo sin requerir el del paquete

### Requirement: El correo del usuario es único global, no solo entre activos
El correo MUST ser único entre todas las cuentas, incluidas las de la papelera, y MUST NOT relajarse a
único entre activos como los nombres de dominio: una cuenta eliminada MUST seguir reservándolo.

#### Scenario: Alta con el correo de una cuenta en la papelera
- **WHEN** se crea una cuenta con el correo de otra que está en la papelera
- **THEN** se rechaza por conflicto de unicidad y no se crea la nueva
