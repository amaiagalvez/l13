# Design: proyectos

## Discrepancias documentadas, no resueltas

### 1. `entity_id` es nullable en el esquema y obligatorio en la aplicación

- `zone-rules-models` dice que `projects.entity_id` es una FK `NOT NULL` con `restrictOnDelete`.
- La migración del bridge la crea `nullable()->constrained('CUM_entities')->restrictOnDelete()`.

El motivo está escrito en `packages/entity-projects13/config/entity-projects13.php` y es doble: la
tabla ya tenía filas cuando el bridge se instaló (y SQLite no admite añadir una columna `NOT NULL` a
una tabla poblada), y el paquete base no puede declarar una FK a una tabla que no debe conocer.

La invariante que la aplicación sí garantiza —todo proyecto tiene cliente— vive en la regla que
aporta el bridge, no en el esquema. Es el mismo patrón que el flag `active`: regla de aplicación,
no restricción de base de datos.

**Pendiente de decidir:** si algún día se acepta que un proyecto sin cliente es un estado válido,
`zone-rules-models` deja de estar desactualizada; si no, la documentación debe cambiar.

### 2. `UniqueProjectName` es código muerto

Existe `projects13/src/Http/Requests/UniqueProjectName.php` (implementa `DataAwareRule` + `Rule`, con
un `setData()` vacío y una comprobación manual de `Project::where('name', …)->first()` que **no**
aplica `whereNull('deleted_at')`). `ProjectRequest` no lo usa: su regla es un `Rule::unique(Project::class)`
como el de `EntityRequest`. No hay ninguna referencia al símbolo en el resto del repositorio.

Es decir: una regla que parece controlar el nombre frente a los eliminados pero no lo hace, junto a
la que sí lo hace, mientras el nombre de la clase sugiere que es la regla canónica del proyecto.

**Pendiente de decidir:** borrarla, o conectarla. No se documenta como invariante viva en
`spec.md` porque no lo es.

### 3. Doble búsqueda por nombre en papelera

`ListQueryBase::findTrashedByName()` acepta un `$projectId` opcional para que `EpicController` busque
el nombre **dentro de un proyecto**, y `EntityController`/`ProjectController` la llaman sin él. El
diálogo de conflicto de nombre se abre igual en los tres casos, pero solo el epic acotó el alcance.
`ARCHITECTURE.md` describe el diálogo en singular para los tres.

## Decisión que sostiene la spec

El nombre se valida en **dos capas** por el motivo que explica `TrashController`: la regla da el error
en el campo correcto en el caso normal, y el índice `active_name` (parcial sobre `deleted_at IS NULL`)
es lo que detiene el caso concurrente, que ninguna validación puede ver. El `active_name` es una
columna generada en MySQL/MariaDB (`IF(deleted_at IS NULL, name, NULL)`) y un índice parcial en
SQLite/PostgreSQL, ambos implementados por `addUniqueActiveNameIndex()`.