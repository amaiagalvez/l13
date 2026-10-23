## ADDED Requirements

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

### Requirement: El periodo del panel es de 7, 30 o 90 días
El periodo MUST terminar hoy, incluido, y durar 7, 30 o 90 días. Sin periodo o con cualquier otro
valor MUST usarse 7.

#### Scenario: Periodo desconocido
- **WHEN** se pide el panel con periodo 45
- **THEN** se muestra el periodo de 7 días
