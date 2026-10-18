## ADDED Requirements

### Requirement: El tipo de cliente tiene nombre en 4 idiomas
El tipo de cliente MUST almacenar su nombre en euskera (`name_eu`), español (`name_es`), francés (`name_fr`) e inglés (`name_en`). El euskera es obligatorio; los demás son opcionales pero se validan igual si se envían.

#### Scenario: Se crea un tipo de cliente con solo nombre en euskera
- **WHEN** se envía `name_eu="Enpresa"` sin los demás idiomas
- **THEN** se crea correctamente y los otros idiomas quedan `NULL`

#### Scenario: Se crea un tipo de cliente con todos los idiomas
- **WHEN** se envía `name_eu="Enpresa"`, `name_es="Empresa"`, `name_fr="Entreprise"`, `name_en="Enterprise"`
- **THEN** se crea correctamente con los 4 nombres

#### Scenario: Se rechaza creación sin nombre en euskera
- **WHEN** se intenta crear sin `name_eu` o con `name_eu` vacío
- **THEN** se rechaza con error de validación en `name_eu`

### Requirement: Unicidad del nombre por idioma entre activos
Cada nombre en cada idioma MUST ser único entre los tipos de cliente no eliminados (incluidos archivados). Un nombre puede reutilizarse si el registro que lo usaba está en la papelera. La unicidad se respalda con índices parciales en base de datos.

#### Scenario: Dos tipos de cliente con mismo nombre en euskera
- **WHEN** existe un tipo activo con `name_eu="Enpresa"` e intentas crear otro con `name_eu="Enpresa"`
- **THEN** se rechaza por conflicto de unicidad en euskera

#### Scenario: Mismo nombre en euskera pero uno está en papelera
- **WHEN** existe un tipo en papelera con `name_eu="Enpresa"` e intentas crear uno activo con `name_eu="Enpresa"`
- **THEN** se permite crear y ofrece restaurar o crear nuevo

#### Scenario: Unicidad independiente por idioma
- **WHEN** existe un tipo con `name_es="Empresa"` e intentas crear otro con `name_es="Empresa"` pero diferente `name_eu`
- **THEN** se rechaza por conflicto en español, aunque el euskera sea diferente

### Requirement: CRUD canónico con papelera, archivado y selectores
El tipo de cliente sigue el patrón canónico: Controller + TrashController + ArchivedController + ListQuery + SelectOptionsQuery + Transformer + FormRequest + Policy + Factory + Vistas Flux + Tests. Los selectores solo devuelven activos.

#### Scenario: Listado de tipos de cliente activos
- **WHEN** se accede al índice de tipos de cliente
- **THEN** se muestra tabla con nombres (euskera principal + selector de idioma), estado activo/archivado, acciones

#### Scenario: Formulario de creación/edición con 4 idiomas
- **WHEN** se abre el formulario
- **THEN** muestra 4 campos de texto con etiquetas "Nombre (eu)", "Nombre (es)", "Nombre (fr)", "Nombre (en)"

#### Scenario: Selector solo ofrece tipos activos
- **WHEN** se consulta el endpoint de opciones
- **THEN** devuelve solo tipos con `active=true` y `deleted_at=NULL`, ordenados por nombre euskera

### Requirement: Borrado bloqueado si hay clientes usando el tipo
Un tipo de cliente con clientes asociados (incluidos en papelera) NO puede pasar a papelera ni borrarse definitivamente. La guarda debe contar clientes eliminados.

#### Scenario: Intentar borrar tipo usado por cliente en papelera
- **WHEN** un tipo tiene un cliente en papelera
- **THEN** el borrado se rechaza informando de la dependencia

### Requirement: El flag archivado no afecta a los clientes
Archivar un tipo de cliente NO archiva sus clientes: siguen activos y en sus listados correspondientes.

#### Scenario: Se archiva un tipo con clientes activos
- **WHEN** un tipo con clientes se archiva
- **THEN** sus clientes permanecen en el listado de activos

### Requirement: Etiquetas y validación en 4 idiomas (es/eu/fr/en)
Todos los textos de UI, validación y mensajes MUST existir en euskera, español, francés e inglés con paridad verificada por test.

#### Scenario: Validación de nombre único en euskera
- **WHEN** falla unicidad en euskera
- **THEN** el mensaje de error aparece en el idioma activo de la aplicación (eu/es/fr/en)

#### Scenario: Mensajes de éxito en euskera
- **WHEN** se crea/actualiza/archiva/restaura/borra
- **THEN** el mensaje flash usa la traducción correspondiente