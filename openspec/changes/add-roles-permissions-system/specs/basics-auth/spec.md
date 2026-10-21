## MODIFIED Requirements

### Requirement: El modelo de usuario predeterminado puede reemplazarse
`basics13` MUST proveer un modelo `User` predeterminado compatible con el stack y MUST permitir que
el host configure otro modelo mediante `auth.providers.users.model`. Si el registro está habilitado,
el modelo configurado MUST extender `Illuminate\Foundation\Auth\User`, como requiere el contrato de
creación de usuarios de Fortify. **El modelo de usuario MUST implementar la concern `HasRoles` para
soportar asignación dinámica de roles y verificación de permisos (usa `MorphToMany` con pivote polimórfico).**

#### Scenario: El host usa el modelo predeterminado
- **WHEN** el proveedor de usuarios no configura un modelo específico
- **THEN** el stack usa el modelo `User` provisto por `basics13` que incluye la concern `HasRoles`

#### Scenario: El host configura un modelo propio
- **WHEN** el host configura otro modelo compatible con Fortify en `auth.providers.users.model`
- **THEN** las acciones y los flujos de autenticación usan ese modelo sin requerir `App\Models\User`
- **THEN** el modelo configurado MUST usar la concern `HasRoles` (o implementar su interfaz) para
  mantener la funcionalidad de roles y permisos

### Requirement: La gestión de cuentas de usuario es un recurso canónico del paquete base
La gestión de cuentas MUST vivir en `basics13` como recurso canónico (listar, crear, actualizar,
archivar y papelera) y MUST operar sobre el modelo de `auth.providers.users.model`, no sobre
`Basics13\Models\User`. Editar sin enviar `password` MUST conservar la contraseña actual.
**El formulario de edición de usuario MAY incluir un selector para asignar roles (múltiples)
si la capability `basics-roles-permissions` está instalada.**
**El modelo de usuario MUST incluir un campo `lang` para almacenar el idioma preferido del usuario (eu, es, fr, en).**
**El sistema MUST incluir un usuario super-admin fijo con `id = 1` que tiene acceso total implícito, está oculto en listados, y es inmutable.**

#### Scenario: El host administra cuentas con su propio modelo
- **WHEN** el host configura `auth.providers.users.model` y gestiona usuarios
- **THEN** el listado, las altas y las bajas operan sobre ese modelo sin requerir el del paquete

#### Scenario: El host administra cuentas y asigna roles (con roles instalado)
- **WHEN** el host edita una cuenta de usuario y `basics-roles-permissions` está instalado
- **THEN** puede asignar uno o varios roles desde los roles activos del sistema
- **THEN** los roles asignados determinan los permisos efectivos del usuario

#### Scenario: El host administra cuentas sin roles (standalone)
- **WHEN** el host edita una cuenta de usuario sin `basics-roles-permissions` instalado
- **THEN** el formulario no muestra selector de roles
- **THEN** la funcionalidad de usuario funciona normalmente

#### Scenario: Usuario selecciona idioma preferido
- **WHEN** se crea/edita un usuario
- **THEN** el formulario incluye selector de idioma desde `Language::options()` (eu: Euskara, es: Español, fr: Français, en: English)
- **THEN** el valor se guarda en `users.lang`
- **WHEN** el usuario accede a la aplicación
- **THEN** `SetUserLocale` middleware lee `auth()->user()->lang`, valida con `Language::isValid()` y hace `app()->setLocale()`
- **THEN** la interfaz se muestra en su idioma preferido (fallback: `Language::default()` → `config('app.locale')`)

#### Scenario: Usuario super-admin (ID=1) inmutable y oculto
- **WHEN** se ejecuta seeder de usuarios
- **THEN** existe usuario con `id = 1`, email/password desde variables de entorno
- **WHEN** admin lista usuarios (activos, archivados, papelera)
- **THEN** usuario ID=1 NO aparece en ningún listado
- **WHEN** se intenta editar/borrar/archivar usuario ID=1
- **THEN** operación rechazada: "Usuario del sistema protegido"
- **WHEN** usuario ID=1 accede a la aplicación
- **THEN** tiene todos los permisos implícitos sin consultar BD

## ADDED Requirements

### Requirement: El modelo de usuario expone API de roles y permisos
El modelo de usuario, el predeterminado o el del host, MUST exponer mediante la concern `HasRoles` la
relación polimórfica `roles()`, la consulta de roles y permisos (`hasRole`, `hasAllRoles`,
`hasPermission`, `hasPermissionTo`, `hasModuleAccess`, `getAllPermissions`) y la asignación de roles
(`assignRole`, `removeRole`, `syncRoles`, que admite un array vacío). Solo cuentan los roles y
permisos activos.

#### Scenario: Verificación de permiso en policy de dominio
- **WHEN** una policy de `entities13` verifica `$user->hasPermissionTo('entities.edit')`
- **THEN** retorna true si el usuario tiene un rol con ese permiso (activo)
- **THEN** retorna false si no tiene ningún rol con ese permiso

#### Scenario: Asignación múltiple de roles
- **WHEN** se llama `$user->assignRole(['admin', 'manager'])`
- **THEN** el usuario tiene ambos roles asignados
- **THEN** `$user->hasRole('admin')` y `$user->hasRole('manager')` retornan true

#### Scenario: Sincronización de roles (empty array permitido)
- **WHEN** se llama `$user->syncRoles([])`
- **THEN** roles previos removidos, usuario queda sin roles
- **THEN** operación permitida (UI debe confirmar antes de ejecutar)

#### Scenario: Wildcard manage implícito
- **WHEN** usuario tiene rol con `projects.manage`
- **WHEN** se verifica `$user->hasPermissionTo('projects.edit')`
- **THEN** retorna true (module.manage implica todas las acciones)