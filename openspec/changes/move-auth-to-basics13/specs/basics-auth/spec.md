## ADDED Requirements
### Requirement: El paquete base proporciona autenticación reutilizable
`basics13` MUST proporcionar el stack reutilizable de autenticación y gestión de cuenta que hoy vive
en `demo13`, incluyendo la integración Fortify, sus páginas, settings, middleware y traducciones. El
host MUST conservar sus responsabilidades propias, como el shell y el dashboard.

#### Scenario: Un host instala el stack de autenticación
- **WHEN** una aplicación instala y registra `basics13`
- **THEN** puede usar los flujos de autenticación y gestión de cuenta provistos por el paquete sin copiar su implementación

#### Scenario: El host conserva el shell de aplicación
- **WHEN** el stack de autenticación del paquete se habilita en un host
- **THEN** el shell y el dashboard continúan perteneciendo al host y no son reemplazados por el paquete

### Requirement: El modelo de usuario predeterminado puede reemplazarse
`basics13` MUST proveer un modelo `User` predeterminado compatible con el stack, y MUST permitir que
el host configure su propio modelo autenticable mediante la configuración estándar del proveedor de
usuarios de Laravel. Si el registro está habilitado, el modelo configurado MUST extender
`Illuminate\Foundation\Auth\User`, como requiere el contrato de creación de usuarios de Fortify.

#### Scenario: El host usa el modelo predeterminado
- **WHEN** el proveedor de usuarios no configura un modelo específico
- **THEN** el stack usa el modelo `User` provisto por `basics13`

#### Scenario: El host configura un modelo propio
- **WHEN** el host configura otro modelo en `auth.providers.users.model`
- **THEN** las acciones y los flujos de autenticación usan ese modelo sin requerir `App\Models\User`

### Requirement: El paquete configura las funciones de autenticación
Las funciones de autenticación que actualmente se activan mediante marcadores `@chisel-*` MUST poder
activarse mediante configuración del host. El stack MUST respetar esa configuración para registrar
sus vistas, acciones y rutas correspondientes.

#### Scenario: Una función está deshabilitada
- **WHEN** el host deshabilita una función de autenticación en la configuración
- **THEN** el paquete no expone su flujo de usuario correspondiente

#### Scenario: Una función está habilitada
- **WHEN** el host habilita una función de autenticación en la configuración
- **THEN** el paquete registra el flujo existente de esa función

### Requirement: Las migraciones de autenticación preservan las instalaciones
El paquete MUST proporcionar el esquema de usuarios, 2FA y passkeys que requiere el stack. Su
integración MUST permitir que una instalación existente adopte el paquete sin perder datos de
usuarios ni volver a ejecutar destructivamente migraciones ya aplicadas.

#### Scenario: Una instalación existente adopta el paquete
- **WHEN** el host integra `basics13` después de haber creado sus tablas de autenticación
- **THEN** los datos existentes se conservan y las migraciones ya aplicadas no se ejecutan otra vez de forma destructiva

### Requirement: Las vistas y rutas de cuenta son extensibles por el host
El paquete MUST registrar las vistas Livewire y las rutas de settings de cuenta que forman parte del
stack trasladado, y MUST permitir que el host publique sobrescrituras de las vistas.

#### Scenario: El host personaliza una vista de autenticación
- **WHEN** el host publica una sobrescritura de una vista provista por el paquete
- **THEN** la aplicación usa la vista sobrescrita sin modificar el paquete

#### Scenario: El host carga las rutas de settings
- **WHEN** se registra el proveedor de `basics13`
- **THEN** las rutas de cuenta trasladadas están disponibles y las rutas del shell y dashboard siguen en el host
