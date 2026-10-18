# Design: límites de paquete

## Mapa de propietarios (para no repetirlo en cada spec)

| Paquete | Contiene | Domain-neutral |
|---------|----------|----------------|
| `basics13` | `User` (modelo de cuenta, sustituible por el host vía `auth.providers.users.model`), `Controller`, `TrashController`, `ArchivedController`, `RestoreRequest`, `TrashDestroyRequest`, `TrashRecordRequest`, `SearchableListRequest`, `SearchableSelectOptionsRequest`, `ListQueryBase`, `ListTransformer`, `TracksAuditColumns`, `TransactionalDelete`, `HasStates`, `UniqueConstraintViolation` + helpers, `FieldHints`, `MaxLength`, `DuskTestCase`, `ValidatesFormRequests`, componentes `list/` y `forms/`, `config/basics13.php` | sí |
| `entities13` | `Entity`, `EntityPolicy`, `EntityController`, `EntityTrashController`, `EntityArchivedController`, sus requests, `EntityListQuery`, `EntitySelectOptionsQuery`, `EntityListTransformer`, vista `entities/{list,form}`, rutas, migraciones, `lang/` | sí (contrato `EntityIntegration`) |
| `projects13` | `Project`, `Epic`, `EpicComment`, sus policies, controllers, requests, queries de `Projects/`, `Epics/`, `Planning/`, `Timeline/`, transformers, vistas de proyectos/epics/planning/timeline, rutas, migraciones, `lang/` | sí (contratos `ProjectDisplay/Query/ValidationIntegration`) |
| `entity-projects13` | `EntityProjectsIntegration`, `EntityCountsIntegration`, `ServiceProvider`, 2 migraciones, partials de `entities/`, `projects/`, `epics/`, `lang/` JSON, assets | — |
| `projects/demo13` | Auth (Fortify), settings, shell, middleware de usuario activo, seeders coordinadores | fuera del sistema de paquetes |

## Dónde vive cada invariante de frontera

- **Guardas de borrado con hijos**: `projects13` registra las de `Project` (epics) y `Epic`
  (comentarios) en su propio `ServiceProvider`. La de `Entity` (proyectos) **no puede** vivir ahí:
  la registra `entity-projects13`, porque es el único que conoce `Project`. Esa asimetría es
  intencionada y es la razón por la que `entities13` no tiene guardas propias.
- **Relaciones `Project::entity`, `Entity::projects`, `Entity::epics`**: `resolveRelationUsing()`
  desde el `ServiceProvider` del bridge, con claves explícitas (`belongsTo()`/`hasMany()` adivinan la
  clave a partir del nombre de la función, y dentro de una closure no hay nombre que adivinar).
- **Migración de `entity_id`**: la aporta el bridge aunque la tabla sea de `projects13`, porque la
  columna es conocimiento de cliente.

## Discrepancia documentada

`zone-rules-models` afirma que la guarda la registra `AppServiceProvider::blockDeletionWithChildren()`
y que ese método vive en la aplicación. El código real la registra en los `ServiceProvider` de
`projects13` y de `entity-projects13`, y `AppServiceProvider` no la tiene. Además, la misma zone
rule describe `projects.entity_id` como `NOT NULL`; la migración del bridge la crea **nullable**
(a propósito, documentado en `config/entity-projects13.php`) y la obligatoriedad vive en la
validación. Ver `openspec/specs/projects-domain/design.md`.

## Decisión de diseño que sostiene las specs

El cruce nunca se hace por herencia, por trait ni por `if (class_exists(OtherPackage::class))`. Se
hace por **contrato + binding en el contenedor**: el paquete base registra su valor por defecto con
`singletonIf`, el bridge lo sustituye con `singleton`. Ningún paquete depende del orden de registro
de Composer, y el base no necesita saber que el bridge existe.