# Design

## Contexto

Entidades y proyectos comparten el mismo listado base (`ListQueryBase`) y la misma vista de lista
para los ámbitos activo, archivado y papelera. La exportación tiene que producir exactamente las
filas que el usuario ve, sin volver a implementar búsqueda, filtros ni ámbito en cada paquete.

## Objetivos / No objetivos

**Objetivos**

- Exportación reutilizable en cualquier listado sin lógica específica de dominio.
- El fichero respeta la búsqueda simple, los filtros avanzados y el ámbito que se está viendo.
- El nombre del fichero va en el idioma de la interfaz y distingue el ámbito.
- Añadir la exportación a un listado no duplica el código de filtrado.

**No objetivos**

- Exportar solo la página actual: se exportan todas las filas que cumplen los filtros.
- Formatos distintos de XLSX o estilos personalizados por listado.
- Una API sin botón en la interfaz.

## Decisiones

- **El servicio no filtra.** `ExcelExportService` recibe un `Builder` ya filtrado y solo lo serializa.
  Así el único sitio que conoce la búsqueda, los filtros y el ámbito es el list query, y desaparece
  el riesgo de que la exportación y la tabla apliquen criterios distintos (por ejemplo, la columna
  `active` sin cualificar cuando la consulta une tablas).
- **`filteredQuery()` en cada list query.** `ProjectListQuery` y `EntityListQuery` exponen
  `filteredQuery(string $search, array $filters, string $scope): Builder`, construido sobre el mismo
  `baseQuery()` y las mismas `searchColumnsFor()` que usa el listado. `ListQueryBase::filtered()` es
  el paso compartido de búsqueda y filtros; `paginate()` delega en él.
- **El ámbito viene del listado, no de la URL.** El componente Blade recibe `:scope="$scope"` y lo
  envía como campo oculto, porque las rutas de archivados y papelera no llevan `?scope=`. Sin el
  prop, usa el `scope` de la query string.
- **Formulario GET.** El botón es un formulario GET con campos ocultos `search`, `filters[…]` y
  `scope`; los valores de filtro que son listas (selección múltiple, `between`) se emiten como
  entradas indexadas, no como JSON.
- **Nombre del fichero traducido y con ámbito.** `ExcelExportService::fileName($label, $scope)`
  compone `Str::slug()` de la etiqueta traducida del recurso, la etiqueta traducida del ámbito
  (`archived_label` / `trash_label` de `basics13`) cuando aplica, y una marca de tiempo:
  `proiektuak_artxibatuak_2026-10-23_10-15-00.xlsx`. Se reutilizan traducciones existentes para no
  añadir claves nuevas.
- **Dependencia en `basics13`.** `maatwebsite/excel` se declara en el `composer.json` de `basics13`,
  el paquete que usa la librería; los paquetes de dominio solo registran su `ExcelServiceProvider` en
  los tests.
- **Columnas definidas por el listado.** La vista pasa las columnas y sus transformadores al
  componente, y el controlador los usa al construir el servicio: cabeceras y celdas salen del mismo
  sitio que la tabla.

## Riesgos / Trade-offs

- [Riesgo] Un listado con muchos registros genera un fichero grande → Mitigación: es una descarga
  explícita; si hiciera falta, se acotaría en el propio list query.
- [Riesgo] Un filtro nuevo que el listado aplique de otra forma → Mitigación: página y exportación
  comparten `filteredQuery()`; el test de exportación y el del listado cubren el mismo criterio.
- [Riesgo] El nombre del fichero depende de claves de traducción de `basics13` → Mitigación: se
  reutilizan `archived_label` y `trash_label`, ya traducidas en los cuatro idiomas.

## Cuestiones abiertas

Ninguna.
