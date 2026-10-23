# Tasks

## 1. Servicio de exportación (basics13)

- [x] 1.1 `ExcelExportService` en `src/Support/Excel/`: serializa a XLSX una colección ya filtrada, con título, cabeceras y transformadores por columna. Verificación: `ExcelExportServiceTest`.
- [x] 1.2 El servicio no filtra ni pagina: recibe el `Builder` que ya aplicó búsqueda, filtros y ámbito. Verificación: el test comprueba que exporta las filas recibidas.
- [x] 1.3 `maatwebsite/excel` declarado en el `composer.json` de `basics13`, no en `demo13`. Verificación: `InstalledCatalogTest` de `demo13`.
- [x] 1.4 Tests del servicio en `tests/Feature/Support/Excel/ExcelExportServiceTest.php` (las Unit no pueden tocar la base de datos). Verificación: tests en verde.

## 2. Componente Blade (basics13)

- [x] 2.1 Componente `x-basics13::list.excel-export` con ruta, etiqueta, columnas y ámbito. Verificación: `PackageViewsTest`.
- [x] 2.2 Formulario GET con campos ocultos `search`, `filters[…]` y `scope`; las listas se emiten como entradas indexadas. Verificación: test de renderizado del componente.
- [x] 2.3 Componente añadido a los listados de `entities13` y `projects13`, con `:scope="$scope"`. Verificación: la acción del listado apunta a la ruta de exportación y lleva el ámbito.
- [x] 2.4 Tests del componente. Verificación: `PackageViewsTest` en verde.

## 3. Integración con filtros y ámbito

- [x] 3.1 `filteredQuery(string $search, array $filters, string $scope): Builder` en `ProjectListQuery` y `EntityListQuery`, compartiendo `baseQuery()` y `searchColumnsFor()` con el listado. Verificación: tests de listado existentes siguen en verde.
- [x] 3.2 `ListQueryBase::filtered()` como paso común de búsqueda y filtros; `paginate()` delega en él. Verificación: tests de filtros de ambos paquetes.
- [x] 3.3 La exportación recibe búsqueda, filtros avanzados y ámbito (activo, archivado, papelera) desde el request. Verificación: `ProjectExportTest` y `EntityExportTest`.
- [x] 3.4 Feature tests de la exportación con cada filtro y cada ámbito. Verificación: tests en verde.
- [x] 3.5 El nombre del fichero usa el idioma actual e identifica el ámbito (`archived_label` / `trash_label`). Verificación: tests del nombre en `basics13`, `projects13` y `entities13`.

## 4. Traducciones

- [x] 4.1 Etiqueta «Export to Excel» en es/eu/fr/en en `basics13`, el paquete que la declara. Verificación: tests de traducciones de cada paquete en verde.
- [x] 4.2 Claves `ID` y `Status` que faltaban en `entities13` y `projects13` en los cuatro idiomas. Verificación: `InstalledCatalogTest` y catálogos de traducciones.

## 5. Verificación final

- [x] 5.1 Suites de `entities13`, `projects13`, `entity-projects13` y `demo13`, Dusk, PHPStan nivel 7 y Pint en Docker. La cobertura se mantiene ≥ 80 %.
