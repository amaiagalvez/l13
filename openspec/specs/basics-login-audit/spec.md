# basics-login-audit Specification

## Purpose
Registro inalterable de inicios y fines de sesión, bloqueo de un login tras varios fallos seguidos y
aviso cuando los fallos se disparan, para auditar accesos y frenar la fuerza bruta.

## Requirements

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

### Requirement: El panel de administración resume los accesos del periodo
Quien tenga `sessions.view` MUST poder ver los intentos fallidos del periodo y, por día, los inicios
de sesión correctos y los usuarios distintos que los hicieron, con cero los días sin accesos. Los
fallos MUST incluir los rechazos por login bloqueado; las filas `lockout` MUST NOT contarse. Sin
`sessions.view` la página MUST responder 403.

#### Scenario: Recuentos del periodo
- **WHEN** en los últimos 7 días un usuario entra dos veces hoy, otro una vez hoy, hay dos fallos, un rechazo por login bloqueado y una fila de bloqueo, y hubo un fallo hace 8 días
- **THEN** con periodo 7 el panel muestra 3 fallos, 7 días y hoy con 3 inicios de 2 usuarios únicos

#### Scenario: Sin permiso
- **WHEN** un usuario sin `sessions.view` abre el panel
- **THEN** recibe 403

### Requirement: El panel de administración resume las cuentas del periodo
Sobre las cuentas no borradas, el panel MUST mostrar: total, activas (con algún inicio de sesión
correcto en el periodo), nuevas (creadas en el periodo), inactivas (no archivadas y sin inicio de
sesión correcto en el periodo), pendientes (sin email verificado) y crecimiento (nuevas respecto a
las creadas antes del periodo, en %, o «—» si no había ninguna). Activas, nuevas y fallos MUST
compararse con el periodo anterior de igual duración en % con signo, sin comparación si allí eran 0.

#### Scenario: KPIs de cuentas
- **WHEN** hay 4 cuentas creadas hace un mes y 1 hoy sin verificar, una de las antiguas archivada y otra borrada, y solo una antigua entró en el periodo
- **THEN** con periodo 7 el panel muestra total 4, activas 1, nuevas 1, inactivas 2, pendientes 1 y crecimiento 33 %

#### Scenario: Comparación con el periodo anterior
- **WHEN** hubo 4 fallos en los 7 días anteriores al periodo y 5 en el periodo, y ningún usuario nuevo en ninguno de los dos
- **THEN** la tarjeta de fallos muestra «+25 %» frente a los 7 días anteriores y la de nuevos no muestra comparación

### Requirement: El panel avisa solo cuando hay motivo
El panel MUST mostrar un bloque de alertas con, cada una solo si su condición se cumple: cuentas
pendientes de verificar (alguna), cuentas no archivadas sin inicio de sesión en los últimos 30 días
(alguna), aumento inusual de registros (al menos 5 nuevas y al menos el doble que en el periodo
anterior) y cuentas archivadas en el periodo (alguna). Sin ninguna, el bloque MUST NOT aparecer.

#### Scenario: Solo las alertas justificadas
- **WHEN** hay una cuenta sin verificar y todas las demás entraron ayer, 6 cuentas nuevas en el periodo frente a 2 en el anterior, y ninguna archivada
- **THEN** aparecen las alertas de verificación y de registros, y no las de inactividad ni de archivadas

#### Scenario: Sin alertas
- **WHEN** ninguna condición se cumple
- **THEN** el bloque de alertas no aparece

### Requirement: El panel reparte los accesos del periodo por dispositivo
El panel MUST mostrar los inicios de sesión correctos del periodo, en total y en % por tipo de
dispositivo, con los tres tipos siempre presentes (0 % si no hay ninguno). Sin inicios de sesión
MUST indicarlo en lugar del gráfico.

#### Scenario: Reparto por dispositivo
- **WHEN** en el periodo hay 3 inicios desde escritorio, 1 desde móvil, ninguno desde tablet, un fallo desde móvil y un inicio de hace 8 días desde tablet
- **THEN** con periodo 7 el panel muestra 4 sesiones: escritorio 75 %, móvil 25 % y tablet 0 %

### Requirement: El periodo del panel es de 7, 30 o 90 días
El periodo MUST terminar hoy, incluido, y durar 7, 30 o 90 días. Sin periodo o con cualquier otro
valor MUST usarse 7.

#### Scenario: Periodo desconocido
- **WHEN** se pide el panel con periodo 45
- **THEN** se muestra el periodo de 7 días
