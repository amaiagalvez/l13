# Design: papelera y estado archivado

## Discrepancias documentadas, no resueltas

Estas specs describen lo que hace el código. Donde la documentación de `.github/` y
`ARCHITECTURE.md` dice otra cosa, se anota aquí. **Ninguna de las dos versiones se corrige desde
una spec**: corregirlas es un proposal.

### 1. `active` (columna) frente a `archived`/`archive` (vocabulario)

| Fuente | Vocabulario |
|--------|-------------|
| Columna y cast | `active` (bool, por defecto `true`) |
| Controller base | `Basics13\Http\Controllers\ArchivedController` |
| Controller de recurso | `XArchivedController` (`EntityArchivedController`, `ProjectArchivedController`, `EpicArchivedController`) |
| Verbos del controller | `archiveRecord()`, `activateRecord()` |
| Rutas | `{plural}.archived.index`, `{singular}.archive`, `{plural}.archived.activate` |
| Estados del transformer | `'active'`, `'archived'`, `'trash'` |
| `AGENTS.md` y `zone-rules-http` | «`active` flag», `XInactiveController`, `InactiveController`, `{singular}.deactivate`, `{plural}.archived.reactivate` |

El código no tiene ningún `InactiveController`. Las rutas `deactivate`/`reactivate` que describen las
zone rules no existen; las reales son `archive`/`archived.activate`.

**Pendiente de decidir:** unificar la documentación con el código (lo natural) o renombrar el código
(lo caro, porque toca rutas, transformers, factories y tests de navegador).

### 2. Rutas de los ficheros

La documentación histórica (`AGENTS.md`, zone rules) sitúa requests, policies, queries y transformers
en `app/Http/…`, `app/Policies`, `app/Queries`, `app/Transformers`, y las vistas en
`resources/views/{entities,projects,epics}/`. El código los tiene **dentro de cada paquete**:

```
packages/basics13/src/{Http,Policies,Queries,Transformers,Concerns,Support,Database}/
packages/entities13/src/…  packages/entities13/resources/views/entities/{list,form}.blade.php
packages/projects13/src/…   packages/projects13/resources/views/{projects,epics,planning,timeline}/
packages/entities13/routes/web.php   packages/projects13/routes/web.php
```

### 3. Claves de configuración y traducción

- Límites de longitud: `MaxLength::string()` / `::longText()` leen
  `config('basics13.validation.max_length.{string,longtext}')`. Las zone rules escriben
  `config('validation.max_length.string')`, clave que ya no existe.
- Mensajes: `__('basics13::messages.…')` y `__('entities13::messages.…')` desde
  `packages/*/resources/lang/{es,eu,fr,en}/` + `*.json`. Las zone rules hablan de `lang/*.json` en la
  aplicación.

## Por qué existe la separación restore / force delete

`TrashController` trata los dos verbos con garantías distintas, y su docblock lo explica: el restore
**no** toma bloqueo (el índice `active_name` es lo que detiene un nombre insertado a mitad de la
petición, y el `catch` de `UniqueConstraintViolation` es lo que lo convierte en respuesta de
conflicto), mientras que el borrado definitivo **sí** toma `lockForUpdate()` sobre la fila que borra
para no perder contra un force delete concurrente. De ahí los dos requirements separados de
`spec.md`.