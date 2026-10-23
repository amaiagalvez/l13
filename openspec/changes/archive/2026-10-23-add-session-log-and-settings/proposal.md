## Why

Hoy no queda constancia de quién entra, quién falla ni desde dónde, y el único freno a la fuerza
bruta es el limitador de Fortify (5 intentos por minuto y por login + IP), fijo en código. Hace falta
un registro de accesos inalterable, un bloqueo configurable por cuenta y un aviso cuando hay un
ataque, con los parámetros editables desde la interfaz.

## What Changes

- Tabla `BAS_sessions` en `basics13`: una fila por inicio de sesión correcto, intento fallido o
  bloqueo, con usuario (si se conoce), login introducido, IP, agente de usuario («desde dónde»;
  sin geolocalización), motivo del fallo y momento. El logout se apunta en la fila de su login
  (`logged_out_at`). Ninguna otra escritura posterior desde la aplicación: sin edición, borrado ni
  rutas de modificación.
- La contraseña de un intento fallido **no** se guarda nunca, ni en claro ni cifrada (decisión del
  usuario frente a la petición inicial: suele ser la contraseña real con una errata).
- Listado de sesiones de solo lectura con filtros por rango de fechas, usuario, login y resultado
  (correcto, fallido, bloqueo). Permiso `sessions.view`.
- Tabla `BAS_settings` de configuración por paquete: `package`, `class`, `variable`, `type`, `value`,
  `name_{eu,es,fr,en}` y `description_{eu,es,fr,en}`. Cada paquete siembra sus filas por migración.
  `basics13` siembra `max_login_attempts` (3), `lockout_minutes` (15) y `alert_email` (vacío).
- Pantalla «Configuración de la aplicación» en Administración, con una pestaña por paquete; solo se
  edita `value`, validado según `type` y con límites. Permisos `settings.view` y `settings.edit`.
- Bloqueo por login normalizado, desde cualquier IP y aunque la cuenta no exista: el fallo número
  `max_login_attempts` (el 3.º por defecto, decisión del usuario) bloquea `lockout_minutes` minutos
  completos y apunta una fila de bloqueo. Durante el bloqueo se rechaza incluso la contraseña
  correcta. Un login correcto reinicia el contador.
- Más de 10 intentos fallidos en total en menos de 5 minutos, incluidos los rechazados por bloqueo:
  `report()` de una incidencia (llega a Sentry si el host lo tiene instalado) y correo inmediato a
  `alert_email` si está configurado, con recuentos y enlace al listado, sin logins ni IPs. Como
  mucho un aviso cada 5 minutos.
- **BREAKING:** el limitador `login` de Fortify pasa de 5/min por login + IP a un tope por IP; el
  freno por cuenta lo hace el bloqueo configurable.

## Capabilities

### New Capabilities

- `basics-login-audit`: registro inalterable de accesos, bloqueo por intentos fallidos y aviso de
  ataque. Va aparte de `basics-auth` porque es auditoría y defensa, no el stack de autenticación, y
  `basics-auth` ya está cerca de su presupuesto.
- `basics-settings`: configuración por paquete en base de datos, traducida y editable desde la
  interfaz. Ninguna capability existente cubre configuración en tiempo de ejecución.

### Modified Capabilities

- `basics-roles-permissions` (de `add-roles-permissions-system`, aún sin archivar): `basics13`
  siembra también los módulos `sessions` y `settings`, y la lista de acciones recoge `impersonate`,
  que la siembra actual ya usa.
- `basics-user-impersonation`: la marca `impersonator_id` se guarda antes del login del usuario
  objetivo (y se borra después del login de vuelta), para que el registro de accesos detecte la
  impersonación en el evento `Login`.

## Impact

- `basics13`:
  - Migraciones de `BAS_sessions`, `BAS_settings`, sus filas iniciales y los permisos nuevos
    (concedidos al rol `admin`).
  - Modelos `SessionLog` y `Setting`, una clase de auditoría de login en `src/Auth`, mailable de
    aviso, controllers, requests, queries, transformer, policies, vistas, rutas y traducciones
    es/eu/fr/en.
  - Cambia `FortifyServiceProvider` (limitador `login`, registro de eventos) y se reescribe el test
    del 429 de `tests/Feature/Auth/AuthenticationTest.php`.
- `projects/demo13`: entradas «Sesiones» y «Configuración de la aplicación» en el grupo
  Administración del sidebar.
- Sin dependencias nuevas: Sentry no se instala; el aviso usa el `report()` de Laravel. Carpeta
  nueva `src/Mail` en `basics13` (aprobada).
- Orden: archivar después de `add-roles-permissions-system`.
