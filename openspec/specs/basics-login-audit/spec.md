# basics-login-audit Specification

## Purpose
Registro inalterable de inicios y fines de sesión, bloqueo de un login tras varios fallos seguidos y
aviso cuando los fallos se disparan, para auditar accesos y frenar la fuerza bruta.

## Requirements

### Requirement: Cada intento de acceso deja una fila sin contraseña
Cada login correcto, intento fallido y bloqueo MUST registrarse con resultado, momento, login
introducido, usuario si se conoce, IP y agente de usuario. Un fallo MUST indicar su motivo: usuario
inexistente, contraseña errónea, cuenta inactiva o login bloqueado. Ninguna fila MUST contener la
contraseña introducida. Restaurar la sesión con la cookie «recordarme» y la impersonación MUST NOT
registrarse como login.

#### Scenario: Fallos con y sin cuenta
- **WHEN** alguien falla la contraseña de una cuenta existente y luego prueba un login inexistente
- **THEN** hay dos fallos con login, IP y motivo, el primero con usuario y el segundo sin él, y ninguno con la contraseña

### Requirement: El registro solo admite apuntar el logout de su sesión
Desde la aplicación, una fila MUST NOT modificarse ni borrarse, salvo para fijar una única vez el
momento de logout de un login correcto, en la fila del login de esa misma sesión. Repetir el logout
MUST NOT fallar ni cambiar nada. MUST NOT existir rutas de edición o borrado del registro.

#### Scenario: Logout repetido
- **WHEN** un usuario cierra sesión y la misma sesión vuelve a pedir logout
- **THEN** solo la fila de su login recibe el momento del primer logout y la petición no da error

#### Scenario: Intento de modificación
- **WHEN** el código intenta cambiar otro campo o borrar una fila
- **THEN** se rechaza y la fila queda igual

### Requirement: El listado de sesiones se filtra por fecha, usuario, login y resultado
Quien tenga `sessions.view` MUST poder listar el registro, más reciente primero, filtrando por rango
de fechas (ambos días incluidos), usuario, login introducido y resultado. Es de solo lectura.

#### Scenario: Filtro combinado
- **WHEN** se filtra por un usuario, un rango de fechas y el resultado fallido
- **THEN** solo aparecen los fallos de ese usuario dentro del rango, incluido el último día

#### Scenario: Sin permiso
- **WHEN** un usuario sin `sessions.view` abre el listado
- **THEN** se deniega

### Requirement: Un login se bloquea al alcanzar el máximo de fallos seguidos
El fallo número `max_login_attempts` con el mismo login normalizado, dentro de `lockout_minutes`
desde el primero, desde cualquier IP y exista o no la cuenta, MUST bloquearlo `lockout_minutes`
minutos completos desde ese fallo y registrar una única fila de bloqueo. Durante el bloqueo MUST
rechazarse incluso la contraseña correcta, sin alargarlo. Un login correcto MUST reiniciar el
contador. Una IP MUST tener además un tope de intentos por minuto.

#### Scenario: Bloqueo con fallos espaciados
- **WHEN** un login inexistente falla en los minutos 0, 1 y 14 desde IPs distintas, con máximo 3 y bloqueo de 15
- **THEN** se registra un bloqueo de 15 minutos y en el minuto 28 se sigue rechazando aunque la contraseña sea correcta

#### Scenario: Fin del bloqueo
- **WHEN** han pasado los minutos de bloqueo
- **THEN** el login vuelve a poder entrar con su contraseña

### Requirement: Un pico de fallos genera un aviso
Si en los últimos 5 minutos hay más de 10 fallos en total, incluidos los rechazados por bloqueo,
MUST reportarse una incidencia y MUST enviarse un correo a `alert_email`, si está configurado, con
el número de fallos, de logins y de IPs distintos y un enlace al listado, sin logins ni IPs. Tras
un aviso, MUST NOT haber otro durante 5 minutos.

#### Scenario: Pico de fallos
- **WHEN** se produce el undécimo fallo en menos de 5 minutos y hay `alert_email`
- **THEN** se reporta la incidencia y se envía un único correo aunque sigan los fallos
