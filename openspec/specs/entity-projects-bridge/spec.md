# Bridge entity-projects

## Purpose
Garantiza que el cruce entre clientes y proyectos quede contenido en un único paquete, que instalar
ese paquete no rompa los proyectos existentes y que su aportación no duplique nada.

## Requirements

### Requirement: El cruce se aporta por contrato
El comportamiento compartido MUST entrar por los contratos y la configuración de los paquetes base, y
MUST llegar a listas, formulario, relaciones y guardas por esos puntos.

#### Scenario: El bridge aporta una columna a la lista de proyectos
- **WHEN** se instala el bridge sobre un dominio que ya funciona
- **THEN** la columna la declara el bridge y la vista base deja un hueco

### Requirement: Instalar el bridge no oculta proyectos
Toda consulta que cruce con clientes MUST usar unión por la izquierda, nunca interior, para que un
registro sin cliente MUST seguir apareciendo. La regla de cliente MUST impedir ese estado.

#### Scenario: La tabla ya tenía filas antes de instalar el bridge
- **WHEN** se lee el listado tras instalar el bridge
- **THEN** los proyectos previos aparecen sin cliente en vez de desaparecer

### Requirement: La columna es opcional en el esquema y obligatoria en la aplicación
La columna MUST ser nullable, porque la tabla ya tiene filas y el base no conoce clientes. La
obligatoriedad MUST vivir en la validación del bridge.

#### Scenario: La migración se ejecuta sobre una tabla poblada
- **WHEN** se añade la columna sobre proyectos existentes
- **THEN** se aplica sin fallar y las filas previas quedan a nulos

### Requirement: Un cliente no desaparece mientras tiene proyectos
La guarda MUST contar también los proyectos eliminados y MUST tomar el bloqueo de fila en la misma
transacción que escribe el borrado.

#### Scenario: Un proyecto aparece entre la comprobación y el borrado
- **WHEN** se asigna un proyecto a un cliente cuyo borrado está en curso
- **THEN** el borrado espera al bloqueo y falla, sin proyecto huérfano

### Requirement: Las relaciones y la UI no invaden el dominio base
Las relaciones que cruzan los dominios MUST registrarse en ejecución desde el paquete que las sostiene.
Las columnas inyectadas MUST ser sólo lo que aporta el cruce.

#### Scenario: El bridge no está instalado
- **WHEN** un paquete de dominio se usa solo
- **THEN** sus modelos no exponen relación hacia el otro dominio

### Requirement: El filtro por cliente lo aporta el bridge
El campo filtrable de cliente del listado de proyectos MUST llegar por el contrato de integración, no
desde el paquete base. MUST ser múltiple, admitir proyectos sin cliente y MUST NOT ofrecerse cuando
la lista ya está acotada a un cliente.

#### Scenario: Se filtran proyectos sin el bridge instalado
- **WHEN** el listado de proyectos se usa sin el bridge
- **THEN** la búsqueda avanzada no ofrece el campo de cliente

### Requirement: El selector de cliente del proyecto se autoriza por el permiso del proyecto
Las opciones de cliente del formulario de proyecto MUST ofrecerse a quien tiene `projects.create` o
`projects.edit`, aunque no tenga `entities.view`, y MUST seguir limitadas a clientes activos. Sin
ninguno de esos permisos, MUST denegarse.

#### Scenario: Crear proyecto sin ver clientes
- **WHEN** un usuario con `projects.create` y sin `entities.view` abre el formulario de proyecto
- **THEN** el selector ofrece los clientes activos y puede guardar el proyecto

#### Scenario: Sin permisos de proyecto
- **WHEN** un usuario sin `projects.create` ni `projects.edit` pide las opciones de cliente
- **THEN** recibe 403

### Requirement: Sin entities.view, al editar el selector de cliente se limita a los de tus proyectos
Quien no tiene `entities.view` MUST recibir, al crear un proyecto, todos los clientes activos, para que
pueda crear su primer proyecto. Al editar un proyecto MUST recibir solo los clientes activos de los
proyectos de los que es miembro, más el cliente actual del proyecto, y la validación MUST rechazar
cualquier otro. Con `entities.view`, el selector MUST ofrecer todos los clientes activos.

#### Scenario: Alta sin proyectos
- **WHEN** un usuario con `projects.create`, sin `entities.view` y sin proyectos abre el alta de proyecto
- **THEN** el selector ofrece todos los clientes activos y puede guardar el proyecto

#### Scenario: Opciones acotadas al editar
- **WHEN** un usuario sin `entities.view` edita uno de sus proyectos
- **THEN** solo ve los clientes de sus proyectos

#### Scenario: Cliente ajeno enviado al editar
- **WHEN** ese usuario envía en la edición un cliente que no está en sus proyectos
- **THEN** la validación lo rechaza en el campo cliente
