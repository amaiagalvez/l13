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