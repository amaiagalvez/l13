## Why
La autenticación y la gestión de usuarios están acopladas actualmente a `projects/demo13`, por lo
que otros proyectos no pueden reutilizar el stack sin copiar código. Trasladar su implementación
compartida a `basics13` permitirá instalar y configurar el mismo flujo de autenticación y gestión de
cuenta en distintos proyectos.

## What Changes
- Trasladar a `basics13` el stack de autenticación basado en Fortify y Livewire, incluido el modelo
  `User` predeterminado, sus acciones, validaciones, middleware, vistas, rutas de cuenta y
  traducciones.
- Proporcionar desde el paquete las migraciones, factory y seeder de usuarios, incluidos los campos
  de 2FA y las passkeys.
- Sustituir los marcadores `@chisel-*` del stack trasladado por configuración del paquete para
  activar las funciones de autenticación disponibles.
- Permitir que cada aplicación configure otro modelo autenticable y publique sobrescrituras de las
  vistas del paquete.
- Mantener en el host el shell, el dashboard y el arranque específico de cada aplicación; retirar de
  `demo13` la implementación duplicada que pase a ser propiedad del paquete.
- Actualizar las pruebas y la documentación de arquitectura para reflejar la nueva propiedad.

## Impact
- Affected specs: `basics-auth` (nuevo), `package-boundaries`.
- Affected code: `packages/basics13/`, `projects/demo13/`, `openspec/specs/package-boundaries/`,
  `.github/docs/architecture/ARCHITECTURE.md`.
- Añade dependencias de ejecución para Fortify y Livewire a `basics13`.
- El cambio de propiedad de las migraciones y del modelo requiere una transición compatible con las
  bases de datos existentes; no debe perderse ni recrearse destructivamente la tabla de usuarios.
