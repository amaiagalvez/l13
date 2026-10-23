## ADDED Requirements

### Requirement: El panel de administración resume cuentas y accesos del periodo
Quien tenga `sessions.view` MUST poder ver un panel con el número de usuarios no borrados, el número
de intentos fallidos y, por cada día del periodo, los inicios de sesión correctos en total y los
usuarios distintos que los hicieron. El periodo MUST ser 7, 30 o 90 días terminando hoy (30 por
defecto, también ante un valor desconocido). Los días sin accesos MUST aparecer con cero. Los
bloqueos MUST NOT contarse como fallos aparte. Sin `sessions.view` la página MUST responder 403.

#### Scenario: Recuentos del periodo
- **WHEN** en los últimos 7 días un usuario entra dos veces hoy, otro una vez hoy, hay tres fallos y un bloqueo, y hubo un fallo hace 8 días
- **THEN** con periodo 7 el panel muestra 3 fallos, 7 días y hoy con 3 inicios de 2 usuarios únicos

#### Scenario: Periodo desconocido
- **WHEN** se pide el panel con periodo 45
- **THEN** se muestra el periodo de 30 días

#### Scenario: Sin permiso
- **WHEN** un usuario sin `sessions.view` abre el panel
- **THEN** recibe 403
