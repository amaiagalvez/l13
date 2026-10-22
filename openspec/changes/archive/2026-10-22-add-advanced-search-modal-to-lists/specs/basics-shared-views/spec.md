## ADDED Requirements

### Requirement: Los filtros avanzados viven en la URL
Los filtros activos MUST serializarse en la query string junto a la búsqueda simple, para que el
estado se comparta y sobreviva a la recarga. La cabecera MUST mostrar una insignia por filtro cuyo
botón de quitar enlace a la lista sin ese filtro, en la primera página. Mostrar los campos avanzados
del modal es estado local y MUST NOT ir a la URL.

#### Scenario: Se recarga una lista filtrada
- **WHEN** se abre una URL con filtros
- **THEN** la lista sale filtrada, el modal los muestra y la cabecera enseña sus insignias

## MODIFIED Requirements

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
