# Límites de paquete

## Purpose
Garantiza que `customers13` y `projects13` sigan siendo instalables y testeables por separado, y que
todo lo que necesita conocer ambos dominios viva en un único punto de cruce: el bridge.

## Requirements

### Requirement: La dirección de las dependencias es fija
Un paquete de dominio MUST NOT depender de otro dominio ni del bridge. La regla alcanza también la
dependencia implícita: claves foráneas, nombres de tabla, rutas, vistas, traducciones, factories,
seeders, tests, claves de payload y condiciones de UI. El bridge MUST poder depender de ambos.

#### Scenario: Un dominio necesita algo del otro dominio
- **WHEN** un cambio en `projects13` necesita leer o escribir un dato de cliente
- **THEN** se implementa en `customer-projects13` a través de un contrato, sin tocar `customers13`

### Requirement: Los dominios funcionan sin el bridge
Cada paquete de dominio MUST registrar un valor por defecto neutral para sus contratos de
integración, de modo que sea operable sin ningún paquete de integración y sin que el consumidor
tenga que comprobar si la integración existe.

#### Scenario: Se instala `projects13` sin `customer-projects13`
- **WHEN** el bridge no está instalado
- **THEN** los proyectos se crean, listan, archivan y borran igual, sin nada de cliente

### Requirement: La integración entra por los puntos de extensión existentes
Todo lo que aporta el bridge MUST pasar por los contratos y la configuración que el paquete base ya
expone, sin añadir abstracciones nuevas. El base MUST seguir siendo dueño de su parte: el bridge
aporta la implementación concreta, el esquema, las relaciones, las guardas y los partials de vista.

#### Scenario: El bridge añade una columna a la tabla de otro paquete
- **WHEN** `customer-projects13` añade `customer_id` a la tabla de proyectos
- **THEN** lo hace con su propia migración y registra las relaciones en ejecución

### Requirement: La infraestructura compartida no absorbe lógica entre dominios
`basics13` MUST limitarse a exponer contratos neutrales y comportamiento por defecto independiente
del dominio. Una regla que necesita saber de clientes y de proyectos a la vez MUST NOT trasladarse
allí para eludir el bridge.

#### Scenario: Una regla menciona ambos dominios
- **WHEN** una regla de negocio nombra clientes y proyectos
- **THEN** vive en el bridge y no se implementa en `basics13` como caso especial