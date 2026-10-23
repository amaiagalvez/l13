## Purpose

Configuración de cada paquete guardada en base de datos, con nombre y descripción en los cuatro
idiomas, consultable y editable desde la interfaz sin desplegar código.

## ADDED Requirements

### Requirement: Cada paquete siembra sus propias variables de configuración
Cada variable MUST identificarse por paquete y nombre de variable, únicos juntos, e indicar la clase
que la usa, su tipo, su valor y su nombre y descripción en `eu`, `es`, `fr` y `en`. Cada paquete MUST
sembrar solo sus variables, de forma idempotente y sin pisar un valor ya cambiado. `basics13` MUST
sembrar `max_login_attempts` (3), `lockout_minutes` (15) y `alert_email` (vacío).

#### Scenario: Siembra repetida
- **WHEN** la siembra se ejecuta de nuevo después de que alguien cambiara `max_login_attempts` a 5
- **THEN** no hay filas duplicadas y el valor sigue siendo 5

### Requirement: La pantalla de configuración agrupa las variables por paquete
Quien tenga `settings.view` MUST ver una pestaña por paquete con sus variables, con nombre y
descripción en el idioma de la interfaz o, si falta, en inglés. Quien tenga `settings.edit` MUST
poder cambiar solo el valor: los enteros entre 1 y 1440 y el correo vacío o válido. Los demás campos
MUST NOT cambiarse desde ninguna petición.

#### Scenario: Valor no válido
- **WHEN** se guarda `lockout_minutes` con 0 o con un valor que no es entero
- **THEN** se rechaza con error en ese campo y el valor anterior se conserva

#### Scenario: Solo lectura
- **WHEN** un usuario con `settings.view` pero sin `settings.edit` abre la pantalla e intenta guardar
- **THEN** ve los valores sin opción de guardar y el envío se deniega

### Requirement: Un valor cambiado se aplica en la siguiente petición
El código MUST leer el valor vigente; tras guardarlo, la siguiente petición MUST usar el nuevo. Si
la variable no existe o su valor está vacío o fuera de rango, MUST usarse el valor por defecto que
indique quien la lee.

#### Scenario: Cambio del máximo de intentos
- **WHEN** se cambia `max_login_attempts` de 3 a 5 desde la pantalla
- **THEN** el siguiente login solo se bloquea al quinto fallo seguido
