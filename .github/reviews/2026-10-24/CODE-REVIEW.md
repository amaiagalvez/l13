# Code Review

Objeto: plan OpenSpec `openspec/changes/add-database-menus/` (proposal, design, tasks, spec `basics-navigation-menus`). Aún no hay código: se revisa el plan contra el código que va a tocar.

## Executive Summary

El plan es viable y proporcionado en su estructura (una clase `NavigationMenu`, un componente anónimo, pivote "alguno de" los permisos). Hay tres problemas que deben corregirse antes de implementar:

1. **La caché por usuario no cumple su objetivo**: con `CACHE_STORE=database` cada lectura de caché es una consulta, y el diseño hace 3 lecturas por carga, frente a ~1 consulta de permisos hoy, que ya suele pagar la policy. Además obliga a invalidar desde muchos puntos y ya se ha encontrado uno sin cubrir (`removeRole`).
2. **Cambia lo que ve cada usuario**: Work y Data hoy se muestran a todos y el plan les pone permisos `.view`. Esto contradice el objetivo declarado.
3. **Los permisos inexistentes dejan el ítem público**: si al sembrar se omite un slug que no existe, el ítem queda sin permisos y lo ve todo el mundo. Planning, Timeline y Work dashboard no tienen slug propio.

## Detected Stack

Laravel 13, PHP 8.5, Livewire 4, Flux free, Tailwind 4, PHPUnit, Dusk, Pint, PHPStan nivel 7, Docker. Caché: `database` en `demo13/.env:43` y `array` en los `phpunit.xml`. Monorepo con paquetes `basics13`, `entities13`, `projects13` y `entity-projects13`.

## Checks Executed

| Check | Resultado |
|---|---|
| `openspec validate add-database-menus --strict` | OK |
| PHPUnit, PHPStan, Pint, Dusk, build | NOT RUN: no hay código del cambio que comprobar |
| Especialistas | Laravel, Security, Database, Performance, Business Logic, Architecture, Testing, Frontend, Maintainability. Omitidos: API (sin endpoints) y DevOps (sin infraestructura) |
| Devil's Advocate | El subagente falló por límite de uso; el checklist de `devil-advocate.agent.md` lo aplicó el orquestador, verificando en el código `HasRoles::removeRole`, `PlanningListRequest`, `TimelineListRequest`, `routes/roles.php` y `sidebar.blade.php` |

## Critical Findings

Ninguno.

## High Findings

### PERF-001 — La caché por usuario no evita consultas con el store `database`

Severity: HIGH · Category: Performance / Architecture · File: openspec/changes/add-database-menus/design.md · Line: 18 · Confidence: HIGH

Problem: el diseño lee `menu:version`, `menu:{user}:{v}` y `menu:items:{v}`.
Evidence: `projects/demo13/.env:43` `CACHE_STORE=database`. `HasRoles` ya carga los slugs una vez por petición (`HasRoles.php:65`).
Impact: entre 2 y 3 SELECT sobre `cache` por carga, en lugar de 0. El test "Recarga sin consultas" pasa con el store `array` y no prueba nada sobre producción. Además, la invalidación global queda repartida en muchos puntos de escritura (ver BUG-001 y BUG-003).
Recommendation: cachear solo la lista global de ítems, con los slugs de sus permisos, e invalidarla desde las migraciones de siembra. Filtrar en cada petición con `hasPermissionTo` sobre los permisos ya cargados. Quitar las claves por usuario, el contador de versión y los hooks en roles y permisos. El requisito pasa a ser: "pintar el menú MUST NOT añadir consultas a las de los permisos del usuario". La visibilidad queda siempre al día y desaparecen DB-002, BUG-001, BUG-003 y MAINT-001.

### BUG-002 — Work y Data pasan de visibles para todos a restringidos

Severity: HIGH · Category: Business logic · File: openspec/changes/add-database-menus/tasks.md · Line: 3.1, 3.2 · Confidence: HIGH

Evidence: en `projects/demo13/resources/views/layouts/app/sidebar.blade.php:24-56` los ítems no tienen `can()`. El diseño dice "sin cambiar lo que ve cada usuario hoy".
Impact: los usuarios sin `projects.view` o `entities.view` pierden enlaces. Hoy esos enlaces les dan 403, así que el cambio puede ser deseable, pero no está declarado.
Recommendation: decidirlo explícitamente. Si se restringe, declararlo en el proposal como cambio de comportamiento y listar el slug exacto de cada ítem.

### BUG-004 — Un slug inexistente deja el ítem público

Severity: HIGH · Category: Security / Correctness · File: design.md · Line: 16 · Confidence: HIGH

Evidence: el diseño dice "omite los que no encuentra", y en la spec un ítem sin permisos lo ve cualquier usuario autenticado. Planning y Timeline autorizan con `viewAny(Project)`, es decir `projects.view` (`PlanningListRequest.php:23`, `TimelineListRequest.php:12`). Work dashboard no comprueba nada. No existen `planning.view` ni `timeline.view`.
Impact: una errata en la siembra deja el ítem visible para todos sin que nada falle.
Recommendation: si un slug no existe, la siembra MUST fallar. Fijar en tasks el permiso de cada ítem: Planning y Timeline → `projects.view`; Work dashboard → ninguno.

## Medium Findings

### BUG-001 — `removeRole` no invalidaría el menú

Severity: MEDIUM · File: packages/basics13/src/Concerns/HasRoles.php · Line: 88-92 · Confidence: HIGH
Evidence: `removeRole` hace `detach` sin pasar por `syncRoles`, y el plan solo invalida en `syncRoles`.
Impact: el usuario sigue viendo ítems de un rol que ya no tiene.
Recommendation: desaparece con la solución de PERF-001. Si se mantiene la caché por usuario, invalidar también aquí y añadir un test.

### BUG-003 — Cambiar `is_super_admin` no invalida el menú

Severity: MEDIUM · File: design.md · Line: 18 · Confidence: HIGH
Recommendation: desaparece con PERF-001. Si no, incluir el flag en la clave de caché.

### BUG-005 — Faltan ítems de administración en la siembra

Severity: MEDIUM · File: tasks.md · Line: 1.2 · Confidence: HIGH
Evidence: Permissions y Permissions summary dependen hoy de `roles.view` (`sidebar.blade.php:87-101`), y Administration dashboard de `sessions.view` con `data-test="nav-admin-dashboard"`. Ninguno aparece en la lista de la tarea 1.2.
Recommendation: listar la clave, el permiso y el `data-test` de los 9 ítems de administración.

### BUG-006 — Grupos vacíos tras filtrar rutas

Severity: MEDIUM · File: design.md · Line: 19 · Confidence: HIGH
Evidence: los grupos vacíos se ocultan en `for()`, pero `Route::has` se aplica después, al pintar.
Recommendation: aplicar `Route::has` dentro de `for()`, antes de ocultar los grupos vacíos.

### ARCH-001 — `basics13` siembra los grupos de dominio `work` y `data`

Severity: MEDIUM · File: design.md · Line: 17 · Confidence: HIGH
Impact: el paquete neutral conoce la agrupación de los dominios, en contra de `package-boundaries`, y contradice la regla de la propia spec "solo ítems de sus propias rutas".
Recommendation: `projects13` siembra `work`, `entities13` siembra `data` y `basics13` solo `administration`. `parent_key` sin FK permite cualquier orden de migraciones.

### DB-001 — El pivote no se sincroniza en siembras posteriores

Severity: MEDIUM · File: tasks.md · Line: 1.2 · Confidence: MEDIUM
Impact: si una versión posterior quita un permiso a un ítem, la fila antigua se queda en el pivote y, con la regla "alguno de", el acceso se amplía.
Recommendation: por cada clave, borrar sus filas del pivote y reinsertarlas en una transacción. El `down()` de cada migración de dominio borra solo sus propias claves.

### TEST-001 — Tests que no prueban lo que dicen

Severity: MEDIUM · File: tasks.md · Line: 2.2, 2.3, 4.1 · Confidence: HIGH
Problem: el diseño dice que el store `array` "recalcula en cada petición", pero persiste dentro del mismo test. Además, "Cambio de idioma" no tiene test, no se fijan los 9 `data-test` que usa Dusk y no hay test de las exclusiones de ruta activa.
Recommendation: medir solo las consultas a `BAS_menus` y a las tablas de permisos. Añadir un Feature test en demo13 que compruebe los `data-test` actuales y un caso por cada exclusión.

### SEC-001 — La visibilidad debe respetar `{módulo}.manage`

Severity: MEDIUM · File: spec.md · Line: 21 · Confidence: MEDIUM
Impact: un join SQL sobre el pivote ocultaría "Roles" a quien solo tiene `roles.manage`.
Recommendation: decidir con `hasPermissionTo`, que ya resuelve `manage` (`HasRoles.php:68`), y añadir el escenario "solo `roles.manage` ve Roles".

## Low Findings

- **MAINT-001 — Duración de la caché contradictoria** (`design.md:18`): `rememberForever` frente a un TTL de 1 día, cuando la usuaria confirmó "sin TTL". Queda resuelto con PERF-001, porque una sola clave global no deja entradas huérfanas.
- **MAINT-002 — `active_pattern` con un formato propio de comas y `!`** (`design.md:20`): usar dos columnas, `active_pattern` y `active_except`, que se pasan tal cual a `routeIs`.
- **I18N-001 — Etiquetas de dominio** (`tasks.md` 1.3, 3.1, 3.2): cada paquete reutiliza sus propias claves en `resources/lang/{es,eu,fr,en}.json`. `basics13` solo guarda las de administración.
- **DOC-001 — Incoherencia sobre el Dashboard**: el proposal dice "ítem del host o fijo" y el diseño decide "fijo". Alinear el proposal.
- **SPEC-001 — La spec describe la implementación de la caché** ("independiente del idioma", "sin recalcular"): dejar solo lo observable, es decir, que las etiquetas siguen el idioma activo.

## Informational Findings

- La impersonación es correcta: el menú es el del usuario autenticado, que es el impersonado.
- El menú no sustituye a las policies, que siguen protegiendo las rutas.
- La capability nueva está justificada y la spec tiene unas 49 líneas, dentro del presupuesto.
- Las etiquetas y `data-test` se pintan con `{{ }}`.

## Security
SEC-001, BUG-004.

## Bugs / Correctness
BUG-001 a BUG-006.

## Database
DB-001. Se rechaza DB-002 (atomicidad del contador) porque desaparece con PERF-001.

## Performance
PERF-001.

## Architecture
ARCH-001, MAINT-002.

## Testing
TEST-001.

## Production
Sin hallazgos propios. PERF-001 cubre el crecimiento de la tabla `cache`.

## Maintainability
MAINT-001, I18N-001, DOC-001, SPEC-001.

## Rejected Findings

| Hallazgo | Motivo |
|---|---|
| Atomicidad y carreras de `menu:version`, y desalojo del contador (Database, Security) | Desaparece con PERF-001 |
| Orden de migraciones entre paquetes (Database) | `parent_key` no tiene FK, así que no rompe nada. Con fallo explícito ante slug inexistente (BUG-004) se detecta un orden erróneo |
| FK propia sobre `parent_key` con cascade (Database) | Complejidad sin un problema demostrado. El `down()` borra las claves propias (DB-001) |
| Invalidación al archivar o borrar un usuario (Security) | El usuario inactivo no llega a ver el menú, porque la autenticación lo bloquea |
| Test de coherencia menú ↔ `viewAny` para cada ítem (Frontend) | Duplica BUG-004 y TEST-001 |
| Granularidad global de la invalidación (Performance) | Coste bajo, y desaparece con PERF-001 |
| `data-test` nuevos en ítems que no lo tenían (Business Logic) | Inofensivo |

## Action Plan

1. **PERF-001**: rediseñar la caché como una lista global de ítems, invalidada solo en la siembra, con filtrado en memoria mediante `hasPermissionTo`. Actualizar la spec, el design y las tareas 2.2 y 2.3. Esto elimina BUG-001, BUG-003, MAINT-001 y SPEC-001. Requiere confirmación de la usuaria, porque cambia lo que se acordó sobre la caché.
2. **BUG-002 y BUG-004**: decidir si Work y Data se restringen o no. Fijar el permiso exacto de cada ítem y hacer que la siembra falle ante un slug inexistente.
3. **BUG-005 y TEST-001**: listar los 9 ítems de administración con su clave y `data-test`, y fijarlos con un Feature test en demo13.
4. **ARCH-001 e I18N-001**: cada dominio siembra su grupo y sus etiquetas.
5. **DB-001 y BUG-006**: sincronizar el pivote en cada siembra y aplicar `Route::has` dentro de `for()`.
6. **SEC-001**: añadir el escenario de `manage`.
7. **MAINT-002 y DOC-001**: usar la columna `active_except` y alinear el proposal.
