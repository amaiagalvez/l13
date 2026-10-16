## Context
`demo13` contiene hoy el stack completo de usuarios y autenticación: integración con Fortify,
modelo `User`, acciones, middleware, ajustes de cuenta, vistas, rutas, migraciones y pruebas.
`basics13` ya es el paquete de infraestructura compartida, pero todavía no ofrece autenticación.
La arquitectura documentada asigna actualmente auth y settings al host, de modo que este cambio
altera explícitamente esa propiedad.

## Goals / Non-Goals
- Goals:
  - Hacer reutilizable en otros hosts el comportamiento de autenticación y gestión de cuenta que hoy
    ofrece `demo13`.
  - Mantener el comportamiento existente para login de usuarios activos, Fortify, 2FA, passkeys y
    settings.
  - Permitir que un host personalice el modelo autenticable y sobrescriba vistas.
  - Mantener los datos y la compatibilidad con instalaciones existentes.
- Non-Goals:
  - Trasladar el shell, el dashboard o el arranque propio de una aplicación.
  - Añadir funciones de autenticación que no existen actualmente.
  - Introducir otra capa de integración o abstracciones específicas del host.

## Decisions

### Propiedad del stack
El paquete `basics13` será dueño de la implementación reutilizable de usuarios, autenticación y
settings de cuenta. El host seguirá configurando el modelo, el middleware y las integraciones
específicas de su aplicación, y conservará el shell y el dashboard.

### Modelo configurable
`basics13` expondrá un modelo `User` predeterminado. El host podrá usar su propio modelo mediante la
configuración existente de `auth.providers.users.model`; los controladores, acciones y relaciones del
stack no deberán asumir que el modelo concreto es `App\Models\User`.

Cuando el registro esté habilitado, el modelo configurable deberá extender
`Illuminate\Foundation\Auth\User`, tal como exige el contrato `CreatesNewUsers` de Fortify. Los demás
flujos seguirán resolviendo el modelo desde `auth.providers.users.model`.

### Configuración de funciones en lugar de marcadores Chisel
Los bloques `@chisel-*` no son editables por Chisel una vez que residen en un paquete Composer. Las
funciones actualmente conmutables mediante esos bloques se habilitarán desde configuración y se
integrarán con la configuración Fortify del host. No se mantendrán marcadores que aparenten activar
funciones pero que el instalador no pueda modificar.

Las claves serán booleanas bajo `basics13.auth.features`: `registration`,
`email_verification`, `two_factor_authentication`, `passkeys` y `password_confirmation`. Estos
nombres corresponden a los marcadores existentes (`registration`, `email-verification`, `2fa`,
`passkeys` y `password-confirmation`). Todas tendrán valor `true` por defecto para conservar la
configuración actual de `demo13`; Fortify seguirá siendo responsable de sus opciones propias.

### Recursos del paquete y sobrescrituras del host
El proveedor de servicio del paquete registrará las migraciones, vistas, traducciones y rutas de
settings. Las vistas del paquete serán publicables para que un host pueda personalizarlas sin
mantener una copia del stack completo. El namespace Livewire usado por las páginas trasladadas se
registrará desde el paquete.

### Dependencias
`basics13` declarará Fortify y Livewire como dependencias de ejecución, puesto que el paquete será
responsable de ejecutar ese stack. Las versiones deberán respetar las versiones mayores compatibles
ya usadas por el workspace.

### Migraciones y datos existentes
La implementación deberá conservar las tablas y los datos de usuario existentes y evitar que una
migración ya aplicada en `demo13` vuelva a ejecutarse con efectos destructivos. Antes de mover o
renombrar migraciones se deberá comprobar cómo se registra su historial en instalaciones existentes
y definir una transición compatible; no se permite resolverlo borrando o recreando datos.

Laravel guarda en `migrations.migration` el nombre base de cada archivo, sin su ruta ni extensión.
Por ello el paquete conservará exactamente los nombres `0001_01_01_000000_create_users_table`,
`2025_08_14_170933_add_two_factor_columns_to_users_table` y
`2024_01_01_000000_create_passkeys_table`. El proveedor cargará esas migraciones desde el paquete;
las instalaciones con esos nombres registrados las omitirán, mientras que las instalaciones nuevas
las ejecutarán. Una prueba de integración simulará el historial previo y comprobará que los datos se
conservan y las migraciones no se duplican.

### Verificación
Las pruebas del paquete cubrirán su funcionamiento autónomo con Testbench. Las pruebas de integración
del host cubrirán la configuración del modelo, el middleware y la continuidad de los flujos que
dependen del arranque de la aplicación.

## Risks / Trade-offs
- Una transición incorrecta de las migraciones puede causar duplicación de esquema o pérdida de
  datos → conservar el historial compatible y probar actualización desde una instalación existente.
- Un cambio de configuración puede alterar qué funciones Fortify están activas → cubrir cada opción
  que hoy habilitan los marcadores y mantener la configuración actual de `demo13`.
- Livewire y Fortify pasan a ser dependencias de `basics13` → comprobar compatibilidad de versiones y
  que el paquete siga siendo instalable en los hosts previstos.
- Los hosts pueden necesitar personalizar vistas o el modelo → exponer los puntos de configuración
  existentes y probar una sobrescritura desde el host.

## Migration Plan
1. Definir la configuración que reemplaza a los marcadores Chisel y la estrategia de compatibilidad
   de migraciones.
2. Añadir las dependencias y trasladar la implementación al paquete sin eliminar todavía el soporte
   del host.
3. Configurar `demo13` para consumir el paquete y verificar los flujos y datos existentes.
4. Retirar la implementación duplicada del host una vez que las pruebas de integración pasen.
5. Actualizar la documentación de arquitectura y las specs vigentes.

## Resolved Questions
- Las claves de activación son `basics13.auth.features.registration`,
  `basics13.auth.features.email_verification`,
  `basics13.auth.features.two_factor_authentication`, `basics13.auth.features.passkeys` y
  `basics13.auth.features.password_confirmation`.
- Se conservan los nombres base originales de las migraciones. Laravel compara esos nombres con la
  columna `migration` de su repositorio, independientemente de la ruta donde encuentre los archivos.
