## MODIFIED Requirements

### Requirement: Cada intento de acceso deja una fila sin contraseña
Cada login correcto, intento fallido y bloqueo MUST registrarse con resultado, momento, login
introducido, usuario si se conoce, IP, agente de usuario y tipo de dispositivo (`desktop`, `mobile`
o `tablet`) deducido del agente de usuario; uno vacío o no reconocido MUST contarse como `desktop`.
Un fallo MUST indicar su motivo: usuario inexistente, contraseña errónea, cuenta inactiva o login
bloqueado. Ninguna fila MUST contener la contraseña introducida. Restaurar la sesión con la cookie
«recordarme» MUST NOT registrarse como login. Iniciar una impersonación MUST registrarse como login
del usuario impersonado, marcado con quien impersona; salir de ella MUST fijar el logout de esa
fila sin cerrar la del superadmin, y cerrar sesión durante ella MUST fijarlo en ambas.

#### Scenario: Fallos con y sin cuenta
- **WHEN** alguien falla la contraseña de una cuenta existente y luego prueba un login inexistente
- **THEN** hay dos fallos con login, IP y motivo, el primero con usuario y el segundo sin él, y ninguno con la contraseña

#### Scenario: Tipo de dispositivo
- **WHEN** se entra desde un iPhone, un iPad, una tablet Android y un navegador de escritorio
- **THEN** las filas quedan como `mobile`, `tablet`, `tablet` y `desktop`

#### Scenario: Impersonación
- **WHEN** un superadmin impersona a un usuario y después sale de la impersonación
- **THEN** hay una fila de login del usuario con el superadmin como impersonador y con logout, y la fila del superadmin sigue abierta

## ADDED Requirements

### Requirement: El panel reparte los accesos del periodo por dispositivo
El panel MUST mostrar los inicios de sesión correctos del periodo, en total y en % por tipo de
dispositivo, con los tres tipos siempre presentes (0 % si no hay ninguno). Sin inicios de sesión
MUST indicarlo en lugar del gráfico.

#### Scenario: Reparto por dispositivo
- **WHEN** en el periodo hay 3 inicios desde escritorio, 1 desde móvil, ninguno desde tablet, un fallo desde móvil y un inicio de hace 8 días desde tablet
- **THEN** con periodo 7 el panel muestra 4 sesiones: escritorio 75 %, móvil 25 % y tablet 0 %
