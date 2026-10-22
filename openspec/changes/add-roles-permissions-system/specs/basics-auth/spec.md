## MODIFIED Requirements

### Requirement: El modelo de usuario predeterminado puede reemplazarse
`basics13` MUST proveer un modelo `User` predeterminado compatible con el stack y MUST permitir que
el host configure otro modelo mediante `auth.providers.users.model`. Si el registro está habilitado,
el modelo configurado MUST extender `Illuminate\Foundation\Auth\User`, como requiere el contrato de
creación de usuarios de Fortify. El modelo configurado MUST usar la concern de roles de `basics13`,
porque la autorización de cuentas y de dominios consulta sus permisos.

#### Scenario: El host usa el modelo predeterminado
- **WHEN** el proveedor de usuarios no configura un modelo específico
- **THEN** el stack usa el modelo `User` provisto por `basics13`

#### Scenario: El host configura un modelo propio
- **WHEN** el host configura otro modelo compatible con Fortify en `auth.providers.users.model`
- **THEN** las acciones y los flujos de autenticación usan ese modelo sin requerir `App\Models\User`

#### Scenario: Modelo propio con roles
- **WHEN** el modelo configurado por el host usa la concern de roles
- **THEN** sus usuarios reciben roles y se autorizan igual que con el modelo predeterminado

### Requirement: La gestión de cuentas de usuario es un recurso canónico del paquete base
La gestión de cuentas MUST vivir en `basics13` como recurso canónico (listar, crear, actualizar,
archivar y papelera) y MUST operar sobre el modelo de `auth.providers.users.model`, no sobre
`Basics13\Models\User`. Editar sin enviar `password` MUST conservar la contraseña actual. Cada acción
MUST exigir su permiso `users.{acción}`. El formulario MUST mostrar el selector de roles solo a quien
tiene `roles.manage` y nunca sobre su propia cuenta.

#### Scenario: El host administra cuentas con su propio modelo
- **WHEN** el host configura `auth.providers.users.model` y gestiona usuarios
- **THEN** el listado, las altas y las bajas operan sobre ese modelo sin requerir el del paquete

#### Scenario: Usuario sin permiso
- **WHEN** un usuario sin `users.view` abre el listado de usuarios
- **THEN** recibe 403

## ADDED Requirements

### Requirement: Las cuentas existentes y las nuevas reciben un rol
Al introducir los permisos, cada cuenta existente no eliminada MUST recibir el rol de sistema `admin`,
para que nadie pierda acceso. Una cuenta creada por registro MUST recibir el rol de
`basics13.roles.registration_role`; si no hay ninguno configurado, MUST quedar sin roles.

#### Scenario: Instalación existente
- **WHEN** se migra una instalación con usuarios
- **THEN** cada usuario no eliminado tiene el rol `admin` y conserva el acceso a la gestión de cuentas

#### Scenario: Registro sin rol configurado
- **WHEN** alguien se registra y no hay rol de registro configurado
- **THEN** la cuenta se crea sin roles y la gestión de cuentas le responde 403
