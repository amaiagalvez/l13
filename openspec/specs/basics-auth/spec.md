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
creación de usuarios de Fortify.

#### Scenario: El host usa el modelo predeterminado
- **WHEN** el proveedor de usuarios no configura un modelo específico
- **THEN** el stack usa el modelo `User` provisto por `basics13`

#### Scenario: El host configura un modelo propio
- **WHEN** el host configura otro modelo compatible con Fortify en `auth.providers.users.model`
- **THEN** las acciones y los flujos de autenticación usan ese modelo sin requerir `App\Models\User`

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
