# Basics: vistas compartidas

## Purpose
Garantiza que los tres dominios compartan el mismo esqueleto de listado y de formulario, para que
añadir un recurso no produzca una cuarta versión del encabezado, la tabla o el estado vacío.

## Requirements

### Requirement: El esqueleto de página y de tabla no se duplica
Todo el esqueleto de un listado MUST venir de los componentes compartidos: encabezado, búsqueda,
búsqueda avanzada, pestañas, tabla, paginación, mensajes de vacío y acciones de fila. El botón de
búsqueda avanzada MUST aparecer solo si el scope tiene algún campo filtrable visible, y un campo cuya
condición falla MUST NOT mostrarse ni siquiera con los campos avanzados desplegados.

#### Scenario: Se añade un recurso nuevo
- **WHEN** un dominio incorpora un listado más
- **THEN** reutiliza los componentes y solo declara columnas, payload y campos filtrables

#### Scenario: Un recurso no declara campos filtrables
- **WHEN** se renderiza su listado
- **THEN** la cabecera no ofrece búsqueda avanzada y la búsqueda simple sigue funcionando

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

### Requirement: Los filtros avanzados viven en la URL
Los filtros activos MUST serializarse en la query string junto a la búsqueda simple, para que el
estado se comparta y sobreviva a la recarga. La cabecera MUST mostrar una insignia por filtro cuyo
botón de quitar enlace a la lista sin ese filtro, en la primera página. Mostrar los campos avanzados
del modal es estado local y MUST NOT ir a la URL.

#### Scenario: Se recarga una lista filtrada
- **WHEN** se abre una URL con filtros
- **THEN** la lista sale filtrada, el modal los muestra y la cabecera enseña sus insignias
