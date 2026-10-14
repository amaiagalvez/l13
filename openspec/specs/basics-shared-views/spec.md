# Basics: vistas compartidas

## Purpose
Garantiza que los tres dominios compartan el mismo esqueleto de listado y de formulario, para que
añadir un recurso no produzca una cuarta versión del encabezado, la tabla o el estado vacío.

## Requirements

### Requirement: El esqueleto de página y de tabla no se duplica
Todo el esqueleto de un listado MUST venir de los componentes compartidos: encabezado, búsqueda,
pestañas, tabla, paginación, mensajes de vacío y acciones de fila.

#### Scenario: Se añade un recurso nuevo
- **WHEN** un dominio incorpora un listado más
- **THEN** reutiliza los componentes y solo declara columnas y payload

### Requirement: El estado vacío ofrece la creación
Una lista sin registros y sin búsqueda MUST ofrecer la creación; una vacía porque la búsqueda no
encontró nada MUST NOT ofrecerla. El vacío MUST cubrir las columnas de las integraciones.

#### Scenario: La lista sale vacía porque no hay registros
- **WHEN** el recurso no tiene registros y no hay término
- **THEN** el estado vacío incluye el botón de creación

### Requirement: Toda acción de fila destructiva se confirma
Una acción destructiva MUST pasar por un modal que nombre el registro y diga si es reversible. Si no
puede ejecutarse por dependencias, MUST aparecer deshabilitada con el motivo visible.

#### Scenario: Un registro tiene hijos que bloquean su borrado
- **WHEN** la fila ofrece borrar algo que no puede eliminarse
- **THEN** el botón aparece deshabilitado y explica la dependencia

### Requirement: El conflicto de nombre se resuelve en el sitio
Cuando el nombre ya lo usa un registro eliminado, la interfaz MUST presentar un modal con las dos
salidas posibles y MUST enviar los datos que la petición necesita para ejecutarlas.

#### Scenario: Se crea un registro cuyo nombre está en la papelera
- **WHEN** se envía un nombre que ya usa un registro eliminado
- **THEN** se vuelve al listado con el modal abierto y ambas acciones

### Requirement: Los formularios comparten envoltura y guarda
El contenedor del formulario MUST venir del componente compartido —cabecera, token, método, cambios,
doble envío y pie— y sus `data-test` MUST seguir la misma convención en todos los recursos.

#### Scenario: Un test de navegador abre otro formulario
- **WHEN** un flujo automatizado busca cancelar y enviar
- **THEN** los encuentra con el mismo patrón que en el resto