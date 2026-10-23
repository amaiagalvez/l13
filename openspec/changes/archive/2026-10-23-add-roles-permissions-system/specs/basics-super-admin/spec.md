## Purpose

La cuenta con acceso total que instala el host: cómo se crea, qué puede hacer y por qué otros usuarios
no pueden verla ni modificarla.

## ADDED Requirements

### Requirement: La cuenta super-admin solo se crea por comando y tiene todos los permisos
Ser super-admin MUST depender de un flag que no se puede asignar en masa ni fijar desde ningún
formulario, y MUST NOT depender del identificador de la cuenta. Solo un comando de consola MUST poder
crearla, y MUST fallar sin cambios si ya existe un super-admin, si el correo está en uso o si la
contraseña está vacía. El super-admin MUST tener todos los permisos sin roles asignados y MUST NOT
recibir roles.

#### Scenario: El primer registro no es super-admin
- **WHEN** el primer usuario se registra en una instalación nueva
- **THEN** no es super-admin y solo tiene el rol de registro configurado

#### Scenario: Alta por comando
- **WHEN** se ejecuta el comando con un correo libre y una contraseña válida
- **THEN** crea una cuenta activa, verificada y super-admin

#### Scenario: Ya existe un super-admin
- **WHEN** se ejecuta el comando y ya hay un super-admin
- **THEN** falla y no crea ni modifica ninguna cuenta

### Requirement: Otros usuarios no ven ni modifican la cuenta super-admin
La cuenta super-admin MUST NOT aparecer en los listados de usuarios (activos, archivados y papelera),
en los selectores de usuarios ni en el resumen de permisos. Otros usuarios MUST NOT editarla,
archivarla, moverla a la papelera, borrarla ni cambiar sus roles. El super-admin MUST poder iniciar
sesión y gestionar su propia cuenta desde los ajustes como cualquier usuario.

#### Scenario: Oculta en los listados
- **WHEN** un usuario con `users.view` abre el listado de usuarios, los archivados o la papelera
- **THEN** la cuenta super-admin no aparece

#### Scenario: Acceso directo denegado
- **WHEN** otro usuario envía una edición, un archivado o un borrado de la cuenta super-admin
- **THEN** recibe 403 y la cuenta no cambia

#### Scenario: Gestión de la propia cuenta
- **WHEN** el super-admin inicia sesión con "recordarme", cambia su contraseña y activa 2FA
- **THEN** todas las operaciones se completan
