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