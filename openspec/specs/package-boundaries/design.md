# Design: límites de paquete

## Mapa de propietarios (para no repetirlo en cada spec)

| Paquete | Contiene | Domain-neutral |
|---------|----------|----------------|
| `basics13` | `Controller`, `TrashController`, `ArchivedController`, `RestoreRequest`, `TrashDestroyRequest`, `TrashRecordRequest`, `SearchableListRequest`, `SearchableSelectOptionsRequest`, `ListQueryBase`, `ListTransformer`, `TracksAuditColumns`, `TransactionalDelete`, `HasStates`, `UniqueConstraintViolation` + helpers, `FieldHints`, `MaxLength`, `DuskTestCase`, `ValidatesFormRequests`, componentes `list/` y `forms/`, `config/basics13.php` | sí |
| `customers13` | `Customer`, `CustomerPolicy`, `CustomerController`, `CustomerTrashController`, `CustomerArchivedController`, sus requests, `CustomerListQuery`, `CustomerSelectOptionsQuery`, `CustomerListTransformer`, vista `customers/{list,form}`, rutas, migraciones, `lang/` | sí (contrato `CustomerIntegration`) |
| `projects13` | `Project`, `Epic`, `EpicComment`, sus policies, controllers, requests, queries de `Projects/`, `Epics/`, `Planning/`, `Timeline/`, transformers, vistas de proyectos/epics/planning/timeline, rutas, migraciones, `lang/` | sí (contratos `ProjectDisplay/Query/ValidationIntegration`) |
| `customer-projects13` | `CustomerProjectsIntegration`, `CustomerCountsIntegration`, `ServiceProvider`, 2 migraciones, partials de `customers/`, `projects/`, `epics/`, `lang/` JSON, assets | — |
| `projects/demo13` | Auth (Fortify), `User`, settings, shell, middleware de usuario activo, seeders coordinadores | fuera del sistema de paquetes |

## Dónde vive cada invariante de frontera

- **Guardas de borrado con hijos**: `projects13` registra las de `Project` (epics) y `Epic`
  (comentarios) en su propio `ServiceProvider`. La de `Customer` (proyectos) **no puede** vivir ahí:
  la registra `customer-projects13`, porque es el único que conoce `Project`. Esa asimetría es
  intencionada y es la razón por la que `customers13` no tiene guardas propias.
- **Relaciones `Project::customer`, `Customer::projects`, `Customer::epics`**: `resolveRelationUsing()`
  desde el `ServiceProvider` del bridge, con claves explícitas (`belongsTo()`/`hasMany()` adivinan la
  clave a partir del nombre de la función, y dentro de una closure no hay nombre que adivinar).
- **Migración de `customer_id`**: la aporta el bridge aunque la tabla sea de `projects13`, porque la
  columna es conocimiento de cliente.

## Discrepancia documentada

`zone-rules-models` afirma que la guarda la registra `AppServiceProvider::blockDeletionWithChildren()`
y que ese método vive en la aplicación. El código real la registra en los `ServiceProvider` de
`projects13` y de `customer-projects13`, y `AppServiceProvider` no la tiene. Además, la misma zone
rule describe `projects.customer_id` como `NOT NULL`; la migración del bridge la crea **nullable**
(a propósito, documentado en `config/customer-projects13.php`) y la obligatoriedad vive en la
validación. Ver `openspec/specs/projects-domain/design.md`.

## Decisión de diseño que sostiene las specs

El cruce nunca se hace por herencia, por trait ni por `if (class_exists(OtherPackage::class))`. Se
hace por **contrato + binding en el contenedor**: el paquete base registra su valor por defecto con
`singletonIf`, el bridge lo sustituye con `singleton`. Ningún paquete depende del orden de registro
de Composer, y el base no necesita saber que el bridge existe.