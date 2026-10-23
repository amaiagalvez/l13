# basics-user-impersonation Specification

## Purpose
Permite a un superadministrador actuar como otro usuario para diagnosticar problemas sin conocer su
contraseña, sin perder su identidad original ni la trazabilidad de lo que escribe.

## Requirements

### Requirement: Solo un superadministrador impersona, y nunca a otro superadministrador
Iniciar una impersonación MUST requerir `is_super_admin`; tener el permiso `users.impersonate` por un
rol MUST NOT bastar. Un superadministrador MUST NOT impersonarse a sí mismo ni a otro
superadministrador. La acción MUST ofrecerse solo en el listado de usuarios activos.

#### Scenario: Un rol recibe el permiso de impersonar
- **WHEN** un usuario sin `is_super_admin` cuyo rol tiene `users.impersonate` intenta impersonar
- **THEN** se rechaza con 403 y la sesión no cambia

#### Scenario: Superadministrador sobre otro superadministrador
- **WHEN** un superadministrador intenta impersonar a otro superadministrador o a sí mismo
- **THEN** se rechaza con 403 y la fila no ofrece la acción

### Requirement: La identidad original se conserva para volver
Al impersonar, el `id` del superadministrador MUST guardarse en la sesión después del login del
usuario objetivo, para que la regeneración de la sesión no lo pierda. Salir MUST restaurar al
superadministrador y borrar esa marca; salir sin impersonación activa MUST NOT cambiar de usuario.

#### Scenario: El usuario impersonado sale
- **WHEN** durante una impersonación se sale de ella
- **THEN** la sesión vuelve al superadministrador original, sin marca de impersonación, en el listado de usuarios

### Requirement: La auditoría distingue al impersonado del superadministrador
Durante una impersonación, `created_by` MUST registrar al usuario impersonado, y `updated_by` y
`deleted_by` MUST registrar al superadministrador original.

#### Scenario: Escritura durante una impersonación
- **WHEN** el superadministrador, impersonando, crea y después edita un registro
- **THEN** `created_by` es el usuario impersonado y `updated_by` el superadministrador original
