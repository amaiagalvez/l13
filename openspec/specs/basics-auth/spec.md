# Autenticación y settings de cuenta

## Purpose
Define el stack de autenticación y gestión de cuenta reutilizable que ofrece `basics13`, sin
trasladar responsabilidades específicas del host.

## Requirements

### Requirement: El paquete base proporciona autenticación reutilizable
`basics13` MUST proporcionar el stack reutilizable de autenticación y gestión de cuenta, incluyendo
la integración Fortify, sus páginas, settings, middleware, migraciones y traducciones. El host MUST
conservar sus responsabilidades propias, como el shell y el dashboard.

#### Scenario: Un host instala el stack de autenticación
- **WHEN** una aplicación instala y registra `basics13`
- **THEN** puede usar los flujos de autenticación y gestión de cuenta provistos por el paquete sin copiar su implementación

#### Scenario: El host conserva el shell de aplicación
- **WHEN** el stack de autenticación del paquete se habilita en un host
- **THEN** el shell y el dashboard continúan perteneciendo al host y no son reemplazados por el paquete

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

### Requirement: Las migraciones de autenticación preservan las instalaciones
El paquete MUST proporcionar el esquema de usuarios, 2FA y passkeys, conservando los nombres base
históricos de sus migraciones. Su integración MUST permitir que una instalación existente adopte el
paquete sin perder datos ni volver a ejecutar destructivamente migraciones ya aplicadas.

#### Scenario: Una instalación existente adopta el paquete
- **WHEN** el host integra `basics13` después de haber creado sus tablas de autenticación
- **THEN** los datos existentes se conservan y Laravel omite las migraciones cuyos nombres ya están registrados

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

### Requirement: El correo del usuario es único global, no solo entre activos
El correo MUST ser único entre todas las cuentas, incluidas las de la papelera, y MUST NOT relajarse a
único entre activos como los nombres de dominio: una cuenta eliminada MUST seguir reservándolo.

#### Scenario: Alta con el correo de una cuenta en la papelera
- **WHEN** se crea una cuenta con el correo de otra que está en la papelera
- **THEN** se rechaza por conflicto de unicidad y no se crea la nueva

### Requirement: La interfaz usa el idioma preferido del usuario
Cada usuario MAY guardar un idioma preferido entre `eu`, `es`, `fr` y `en`. Cualquier otro valor MUST
rechazarse al guardar. En cada petición autenticada, la interfaz MUST mostrarse en el idioma del usuario
y, si no tiene ninguno o el guardado ya no es válido, en `config('app.locale')`. Los selectores MUST
mostrar cada idioma por su nombre nativo, no por una bandera.

#### Scenario: Idioma aplicado o por defecto
- **WHEN** un usuario con idioma `eu` abre cualquier página
- **THEN** la interfaz se muestra en euskera; sin idioma, o con uno no válido, en `config('app.locale')`

#### Scenario: El usuario elige su idioma
- **WHEN** un usuario sin permisos de gestión de cuentas cambia su idioma desde los ajustes de perfil
- **THEN** se guarda, y la siguiente página se muestra en ese idioma

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
