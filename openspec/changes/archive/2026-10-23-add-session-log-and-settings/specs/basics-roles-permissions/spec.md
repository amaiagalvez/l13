## MODIFIED Requirements

### Requirement: Los permisos se siembran por paquete y son de solo lectura
Cada permiso MUST tener el slug `{módulo}.{acción}`, con la acción en `view`, `create`, `edit`,
`delete`, `archive`, `restore`, `force_delete`, `impersonate` o `manage`. Cada paquete MUST sembrar
solo los permisos de los módulos que comprueban sus policies; `basics13` MUST sembrar solo `users`,
`roles`, `sessions` y `settings`. Sembrar MUST ser idempotente y MUST NOT quitar permisos ya
asignados a roles. La interfaz MUST NOT crear, editar ni borrar permisos: solo los lista agrupados
por módulo y los asigna a roles.

#### Scenario: Siembra repetida
- **WHEN** la siembra de permisos se ejecuta dos veces
- **THEN** no hay permisos duplicados y los roles conservan los suyos
