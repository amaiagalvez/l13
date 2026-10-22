## ADDED Requirements

### Requirement: La interfaz usa el idioma preferido del usuario
Cada usuario MAY guardar un idioma preferido entre `eu`, `es`, `fr` y `en`. Cualquier otro valor MUST
rechazarse al guardar. En cada petición autenticada, la interfaz MUST mostrarse en el idioma del usuario
y, si no tiene ninguno o el guardado ya no es válido, en `config('app.locale')`. Los selectores MUST
mostrar cada idioma por su nombre nativo, no por una bandera.

#### Scenario: Usuario con idioma
- **WHEN** un usuario con idioma `eu` abre cualquier página
- **THEN** la interfaz se muestra en euskera

#### Scenario: Usuario sin idioma
- **WHEN** un usuario sin idioma guardado abre cualquier página
- **THEN** la interfaz se muestra en `config('app.locale')`

#### Scenario: Código no soportado
- **WHEN** se guarda un usuario con idioma `de`
- **THEN** se rechaza con error en el campo idioma

#### Scenario: El usuario elige su idioma
- **WHEN** un usuario sin permisos de gestión de cuentas cambia su idioma desde los ajustes de perfil
- **THEN** se guarda, y la siguiente página se muestra en ese idioma
