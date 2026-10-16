## 1. Contrato y compatibilidad
- [x] 1.1 Inventariar los flujos, las funciones Chisel y los archivos de auth/settings que deben trasladarse, distinguiendo lo compartido del shell y dashboard del host.
- [x] 1.2 Definir las claves de configuración que sustituyen a los marcadores Chisel y cómo se conectan con Fortify.
- [x] 1.3 Definir y probar una estrategia de migración compatible con instalaciones que ya ejecutaron las migraciones de usuarios, 2FA y passkeys.
- [x] 1.4 Actualizar el mapa de propiedad y las reglas de límites de paquete para el stack de autenticación compartido.

## 2. Modelo y persistencia en `basics13`
- [x] 2.1 Añadir Fortify y Livewire como dependencias de ejecución con versiones compatibles.
- [x] 2.2 Trasladar el modelo `User` predeterminado, conservando auditoría, usuarios activos, soft deletes, 2FA y passkeys.
- [x] 2.3 Registrar las migraciones de usuarios, 2FA y passkeys con una transición segura para instalaciones existentes.
- [x] 2.4 Trasladar la factory y el seeder de usuarios como fixtures reutilizables.
- [x] 2.5 Permitir que el host reemplace el modelo mediante `auth.providers.users.model`.

## 3. Lógica de autenticación y settings
- [x] 3.1 Trasladar el proveedor de Fortify, acciones de creación y restablecimiento de contraseña, reglas de validación y limitadores.
- [x] 3.2 Trasladar el middleware de usuario activo y el comportamiento de cierre de sesión asociado.
- [x] 3.3 Trasladar el middleware de confirmación de contraseña para Livewire y la acción Livewire de cierre de sesión.
- [x] 3.4 Registrar las funciones de autenticación desde configuración, sin depender de los marcadores `@chisel-*`.
- [x] 3.5 Trasladar las rutas de settings de cuenta, manteniendo en el host sus rutas de shell y dashboard.

## 4. Vistas y recursos del paquete
- [x] 4.1 Trasladar las vistas de autenticación, settings, layouts y componentes relacionados con passkeys.
- [x] 4.2 Registrar el namespace Livewire de las páginas trasladadas y permitir publicar sobrescrituras de vistas.
- [x] 4.3 Trasladar las traducciones necesarias y mantener la paridad de idiomas aplicable.
- [x] 4.4 Registrar rutas, vistas, traducciones y migraciones desde el proveedor de servicio de `basics13`.

## 5. Integración del host
- [x] 5.1 Configurar `demo13` para usar el modelo, proveedor, middleware, rutas y funciones provistas por `basics13`.
- [x] 5.2 Retirar de `demo13` los archivos trasladados, evitando eliminar configuración propia del host.
- [ ] 5.3 Confirmar que el shell y el dashboard siguen siendo responsabilidad del host y que la aplicación puede sobrescribir modelo y vistas.

## 6. Pruebas y validación
- [x] 6.1 Trasladar o adaptar las pruebas de auth, usuarios, settings y middleware a pruebas del paquete.
- [x] 6.2 Añadir pruebas de integración del host para el modelo configurable, las funciones habilitadas, rutas y middleware.
- [ ] 6.3 Probar los flujos existentes de registro, login de usuario activo/inactivo, restablecimiento de contraseña, 2FA, passkeys y settings.
- [x] 6.4 Verificar la actualización desde un esquema existente sin pérdida de datos ni ejecución duplicada de migraciones.
- [ ] 6.5 Ejecutar las suites de pruebas aplicables, Pint y PHPStan para el paquete y el host.
- [ ] 6.6 Actualizar y validar la documentación de arquitectura y las specs afectadas.
