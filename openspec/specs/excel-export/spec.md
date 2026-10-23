# excel-export Specification

## Purpose
Garantiza que la exportación a Excel de un listado contenga exactamente las filas que el usuario está
viendo, sin que cada dominio vuelva a implementar la búsqueda, los filtros ni el ámbito.

## Requirements

### Requirement: La exportación contiene las mismas filas que el listado
La exportación MUST aplicar la misma búsqueda, los mismos filtros avanzados y el mismo ámbito
(activos, archivados o papelera) que el listado desde el que se lanza, y MUST incluir todas las filas
que los cumplen, no solo la página actual. El servicio de exportación MUST NOT filtrar por su cuenta:
recibe la consulta ya filtrada que comparte con el listado.

#### Scenario: Exportar con búsqueda y filtros
- **WHEN** el usuario ha escrito un término y aplicado filtros avanzados y exporta
- **THEN** el fichero contiene solo los registros que coinciden, también los de otras páginas

#### Scenario: Exportar archivados o papelera
- **WHEN** el usuario exporta desde el listado de archivados o de la papelera
- **THEN** el fichero contiene los registros de ese ámbito, aunque la URL no lleve `scope`

### Requirement: El nombre del fichero está traducido e identifica el ámbito
El nombre del fichero MUST usar la etiqueta del recurso en el idioma de la interfaz y, en archivados
y papelera, la etiqueta traducida del ámbito, para que exportaciones del mismo recurso en ámbitos
distintos no compartan nombre.

#### Scenario: Exportar en euskera
- **WHEN** el usuario exporta los proyectos archivados con la interfaz en euskera
- **THEN** el fichero se llama `proiektuak_artxibatuak_<fecha>.xlsx`

### Requirement: La exportación es neutral respecto al dominio
El servicio y el componente de exportación MUST vivir en `basics13` sin lógica de ningún dominio;
cada listado solo MUST declarar su ruta, sus columnas y su consulta filtrada.

#### Scenario: Un listado nuevo añade la exportación
- **WHEN** un dominio añade el botón de exportación a su listado
- **THEN** no modifica el servicio ni el componente de `basics13`
