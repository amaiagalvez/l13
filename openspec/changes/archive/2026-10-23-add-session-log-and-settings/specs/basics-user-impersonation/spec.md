## MODIFIED Requirements

### Requirement: La identidad original se conserva para volver
Al impersonar, el `id` del superadministrador MUST guardarse en la sesión antes del login del
usuario objetivo y MUST conservarse tras la regeneración de la sesión, para que el registro de
accesos distinga la impersonación de un login real. Salir MUST restaurar al superadministrador y
borrar esa marca después de su login; salir sin impersonación activa MUST NOT cambiar de usuario.

#### Scenario: El usuario impersonado sale
- **WHEN** durante una impersonación se sale de ella
- **THEN** la sesión vuelve al superadministrador original, sin marca de impersonación, en el listado de usuarios
