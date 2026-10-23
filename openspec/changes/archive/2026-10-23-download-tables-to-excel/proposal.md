# Proposal

## Why

Los listados de la aplicación (entidades, proyectos…) no permiten descargar lo que se está viendo.
Resolverlo listado a listado duplicaría la búsqueda, los filtros avanzados y el ámbito en cada
exportación, con el riesgo de que el fichero deje de coincidir con la tabla. Hace falta una
exportación a Excel global, reutilizable y consciente de los filtros y del ámbito (activos,
archivados, papelera).

## What Changes

- `basics13` incorpora `ExcelExportService`: serializa a XLSX una consulta ya filtrada, con cabeceras,
  título y nombre de fichero traducidos. No filtra ni pagina por su cuenta.
- Cada list query expone `filteredQuery()`, de modo que la página y la exportación comparten la misma
  búsqueda, los mismos filtros avanzados y el mismo ámbito, sin duplicar la lógica.
- Componente Blade compartido `x-basics13::list.excel-export`: envía por GET la búsqueda, los filtros
  y el ámbito actuales al endpoint de exportación del listado.
- `entities13` y `projects13` añaden su ruta y su acción de exportación. El nombre del fichero se
  traduce y distingue el ámbito (`bezeroak_…`, `bezeroak_artxibatuak_…`, `bezeroak_zakarrontzia_…`).
- `maatwebsite/excel` se declara en `basics13`, el paquete que usa la librería.

## Capabilities

### New Capabilities

- `excel-export`: exportación a Excel de un listado con los filtros y el ámbito que el usuario está
  viendo.

### Modified Capabilities

Ninguna.

## Impact

- `basics13`: servicio, componente Blade, `ListQueryBase::filtered()`, traducciones y dependencia
  `maatwebsite/excel`.
- `entities13` y `projects13`: `filteredQuery()` en su list query, acción y ruta de exportación,
  botón en el listado y traducciones (incluida `ID`).
- Sin cambios de esquema ni de datos.
