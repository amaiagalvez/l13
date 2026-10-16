## ADDED Requirements
### Requirement: La autenticación reutilizable pertenece a la infraestructura base
`basics13` MAY proporcionar el stack común de autenticación y gestión de cuenta, incluido un modelo
de usuario predeterminado configurable. Los hosts MUST conservar la configuración y el arranque
específicos de su aplicación; este reparto MUST NOT introducir dependencias desde `basics13` hacia
una aplicación host ni hacia un paquete de dominio.

#### Scenario: Un host usa autenticación sin acoplarse a un dominio
- **WHEN** una aplicación instala el stack de autenticación de `basics13`
- **THEN** puede configurar su modelo y sus funciones sin que `basics13` dependa de la aplicación o de `customers13` o `projects13`

#### Scenario: Un host mantiene el comportamiento propio de su aplicación
- **WHEN** una aplicación usa el stack común de autenticación
- **THEN** conserva el shell, el dashboard y las demás responsabilidades específicas del host
