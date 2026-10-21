# Guía de uso con tu configuración actual. Todo cuelga de tres piezas que ya tienes: 

* AGENTS.md (contexto que Copilot carga en cada sesión vía copilot-instructions.md)
* .github/instructions/ (reglas que se inyectan solas al tocar cada ruta: http, models, views, tests y packages)
* los agentes de .github/agents/ y los flujos guiados (skills) de .github/skills/

Encima de esas tres hay una cuarta capa, **OpenSpec / Spec-Driven Development** (abajo, "Cambios con spec"). Los cuatro docs de `.github/docs/` no se solapan: `how.md` (este) es el manual de uso, `spec_sdd.md` explica las specs, `review-rules.md` lo que bloquea una revisión y `architecture/ARCHITECTURE.md` la estructura del monorepo.

## Reutilización en OpenCode y Claude Code

Los originales siguen en `.github/agents/` y `.github/skills/`. Ejecuta los dos scripts desde la raíz del repositorio después de clonar o al añadir o renombrar un agente o una skill:

```
bash testing13/agents/link-opencode-assets.sh
bash testing13/agents/link-claude-assets.sh
```

Ambos localizan la raíz del repositorio automáticamente y son seguros de volver a ejecutar. **Después de modificar `.github/agents/` (añadir, borrar o renombrar agentes), ejecuta siempre ambos scripts para mantener los symlinks actualizados en `.opencode/agents/` y `.claude/agents/`.** Los scripts eliminan primero los symlinks previos en sus carpetas de agentes y vuelven a crearlos. Esto asegura que nuevos agentes queden disponibles y los removidos queden ocultos.

* OpenCode descubre agentes en `.opencode/agents/`; son symlinks a `.github/agents/*.agent.md`. El instalador intenta reiniciar automáticamente su servicio en segundo plano con `opencode service restart`.
* Claude Code descubre agentes en `.claude/agents/`; son symlinks directos a `.github/agents/*.agent.md`, sin copiar ni adaptar el frontmatter. Los permisos de Copilot/OpenCode no se convierten a restricciones de herramientas de Claude.
* Las skills no se enlizan una a una: `.opencode/skills/` y `.claude/skills/` son un **symlink de carpeta completa** a `.github/skills/`. Añadir, borrar o renombrar una skill no necesita nada más; el script borra y recrea ese symlink.
* Los archivos `.github/instructions/*.instructions.md` siguen siendo instrucciones de Copilot. OpenCode y Claude necesitan seguir las indicaciones de zona de `AGENTS.md` o de las propias skills; el `applyTo` de Copilot no se aplica en esos clientes.

Los scripts eliminan primero los symlinks previos en sus carpetas de agentes y vuelven a crearlos. Claude retira también las copias generadas por la versión anterior del script; otros archivos normales se conservan. Los enlaces están en `.gitignore`.

Las rutas `.claude/{agents,skills}/` y `.opencode/{agents,skills}/` son de solo lectura en VS Code, tanto al abrir la carpeta como el workspace. Edita agentes y skills desde `.github/`; los cambios de contenido se reflejan inmediatamente en los enlaces. Ejecuta los scripts para reconstruirlos tras añadir, borrar o renombrar archivos. Este bloqueo es del editor: los symlinks no impiden escribir en los originales desde un terminal u otro editor.

Si el script no encuentra el CLI o falla el reinicio, informa del problema; en ese caso ejecuta `opencode service restart` manualmente.

# Punto de partida común (los flujos)

* Abre VS Code en la raíz del repo y comprueba en el chat: Boost MCP activo (laravel-boost en el selector de tools) y modo Agent (no "Ask": necesitas que edite archivos).
Nueva sesión por tarea. El contexto de AGENTS.md + las instrucciones de ruta entran solos; no se lo repitas.
* Los comandos seguros (php artisan test, pint --dirty, make:, npm run build, docker compose up) se ejecutan sin pedirte clic. Si pide otra cosa, revísala.
* **Ojo con dónde vive el código**: `projects/demo13` es solo el contenedor (auth/Fortify, User, ajustes) con 11 ficheros en `app/`. Entities, projects, epics y la infra compartida viven en `packages/{basics13,entities13,projects13,entity-projects13}`, cada uno con su propio PHPUnit. Un cambio que toque ambos dominios va al bridge `entity-projects13`; los dominios no se conocen entre sí.

## Añadir una funcionalidad
1. Activa la skill `new-feature` y pásale la descripción:
```
skill new-feature — CRUD de facturas: listado con búsqueda, formulario, papelera y restore, con i18n
```
2. Déjalo seguir su guion: lee arquitectura/reglas de ruta → decide el paquete dueño → plan corto (aquí corrige el alcance si algo no te cuadra) → tests rojos → implementa siguiendo el patrón de los hermanos (XController + XTrashController + XArchivedController, Form Requests, policy, XListQuery, XListTransformer, `<x-list.*>`, Flux).
3. Supervisa los puntos de control, no el proceso:
   * El plan inicial: ¿querías eso o algo distinto? ¿Y en qué paquete cae?
   * Los `resources/lang/{es,eu,fr,en}.json` en el paquete dueño y los data-test si hay UI.
4. Verificación final: el agente ya ha corrido los tests y las guardas de paquete y te ha dado su resultado real. Tú comprueba que cuadra con el diff y, si tocaba UI, que el build se hizo (auto-aprobado) o levanta dev (-p 5173:5173 ... run dev).
5. Revisa el diff y commitea tú.

Sin skill también funciona: describe la feature y el agente acabará en lo mismo. La skill ya no aporta la regla de no montar abstracciones de más (eso vive ahora en `AGENTS.md`, siempre activo); lo que aporta es el guion fijo: plan antes de escribir, TDD, las guardas de paquete al final y el informe.

Si lo que quieres es un **recurso CRUD entero** (modelo + migración + vistas), usa la skill `new-resource` y pásale el nombre: `skill new-resource — Invoice`. Monta el set completo calcado a Entity (XController + XTrashController + XArchivedController, XRequest/XListRequest/XRestoreRequest, policy, XListQuery, XListTransformer, vistas con x-list.table y x-forms.tracked-resource, lang en 4 locales) dentro del paquete dueño, siguiendo TDD, y termina exigiendo pint, PHPStan y las guardas de paquete en verde. `new-feature` sigue siendo lo indicado para features que no son un recurso completo.

## Revisar incongruencias
Cuatro niveles, de menos a más:

### Rápido
* "Revisa los cambios sin commitear de mi rama"	
* Informe en el chat; solo entonces se carga review-rules.md

### Puntual
* Selecciona un agente concreto del selector del chat (#security-reviewer, #database-reviewer…)	
* Auditoría de un área, READ-ONLY
* Consulte las guardas automatizadas: **ResourceUniformityTest**, **ModelSchemaParityTest**, **PackageIsolationTest** — estos tests vigilan la coherencia de recursos, paridad de fillables y aislamiento de paquetes respectivamente.

### Completa
* skill `full-review`
* Informe en .github/reviews/YYYY-MM-DD/CODE-REVIEW.md con IDs estables (SEC-001, DB-004…) tras el abogado del diablo

### Consistencia entre recursos
* skill `consistency-review`
* Compara entities/projects/epics (o el recurso nuevo) contra los patrones canónicos de forms y listados y lista las divergencias con IDs (PAT-001…). Útil tras varios `new-resource` seguidos: la forma del último recurso nuevo se va desvianzando sin que nadie se dé cuenta.

Flujo recomendado: `full-review` → abre el CODE-REVIEW.md → selecciona los hallazgos que te convenzan → sesión nueva → skill `fix-review` con los IDs:

```
skill fix-review — SEC-002 y DB-004
```
Esa skill reverifica que el hallazgo sigue existiendo, hace el cambio mínimo y ejecuta los tests. La revisión es asesoría: verifica a mano seguridad, auth y migraciones destructivas.

## Arreglar bugs
* Bug en comportamiento → sesión nueva y descríbelo con stack/error: "El botón de restore no aparece en la papelera de epics tras borrar con comentarios" → él reproduce con el test más estrecho (en el paquete dueño del recurso, no en la app), distingue si es bug de código o de test, cambio mínimo, Pint, rerun.
* Tests rojos → skill `fix-tests` (ruta del test o --filter=...). Distingue bug/test/entorno y nunca dice "en verde" sin haberlo ejecutado.
* Bug de UI/JS → pega error de consola (Boost MCP browser-logs si hay pestaña de debug) → el fix va con Dusk solo si es comportamiento JS (regla de tests.instructions.md).
* Hallazgo de revisión → no lo repitas: usa la skill `fix-review` con su ID para que respete la evidencia original.

## Cambios con spec (OpenSpec)

Antes de los flujos, una decisión de encuadre: **si el cambio es un bug que restaura lo ya especificado, se arregla directo**. Si introduce capacidades, rompe algo o mueve arquitectura, primero se escribe el *why* en `openspec/changes/<id>/` y luego se implementa.

* `openspec-propose` — redacta el change (proposal, specs, design, tasks).
* `openspec-apply-change` — implementa sus tasks.
* `openspec-archive-change` — al mergear, mueve el change a `changes/archive/` y actualiza `specs/`.

Son skills, iguales en los tres clientes (`.github/skills/openspec-*`). Las reglas del proyecto están en `openspec/config.yaml`; el "para qué sirve" en `.github/docs/spec_sdd.md`. Un proposal = un cambio atómico = un PR, con `## ADDED|MODIFIED|REMOVED Requirements` y un `#### Scenario:` por requirement. `openspec/specs/` no se edita a mano. Para saber qué spec leer antes de tocar algo, AGENTS.md tiene el mapa.

## Comprobaciones automáticas (no hace falta pedirlas)
Dos suites distintas, y conviene no confundirlas:

**1. La app** (`php artisan test` desde `projects/demo13`) cubre auth, settings, dashboard y users. Ahí viven `ArchitectureTest` (User soft-deletable sin policy, `{!! !!}` limitado al QR de 2FA) y `ModelSchemaParityTest` (paridad `#[Fillable]` ↔ esquema del User).

**2. Los paquetes**, cada uno con su PHPUnit y su sqlite en memoria. Desde la raíz del repo:
```
bash testing13/phpunit.sh projects13
bash testing13/phpunit.sh entity-projects13
```
(o `bash testing13/runAll.sh <paquete>` para la cadena completa: dump-autoload, composer install, pint, phpstan, phpunit).

Las guardas que vigilan la coherencia de los recursos están ahí, no en la app:
* **projects13/tests/Unit/ArchitectureTest** — cada recurso trae su set completo (controller, controller de papelera, archived controller, requests, policy, list query, transformer) y los controllers son thin: sin `DB::`/`Schema::`/`dd()`/`paginate()` y con FormRequest declarado en todo endpoint que recibe input.
* **entity-projects13/tests/Unit/ValidationCoverageTest** — todo lo que un modelo puede asignar en masa (`#[Fillable]`) y todo lo que los controllers leen con `$request->string()/boolean()` tiene su regla en un FormRequest. Sube un campo, una migration o un flag de formulario sin validar y se pone rojo.
* **entity-projects13/tests/Feature/ModelSchemaParityTest** — modelos y esquema no se desincronizan: una columna nueva sin `#[Fillable]` falla hasta que la añades (o la declaras como columna de servidor).
* **entity-projects13/tests/Feature/ResourceUniformityTest** — los recursos no se desvían entre sí: mismo orden de reglas y mismo `reuse_deleted_name` en los `XRequest`, mismas reglas de búsqueda en los `XListRequest`, mismas habilidades de policy con su call-site HTTP, factories con los mismos estados, mismo estado `archived` en todo modelo con flag `active`, mismo page size y desempate en las list queries, y el mismo sobre de acciones y `editPayload` allowlisted en los transformers. Un recurso nuevo que se desvíe se pone rojo aquí.
* **entities13|projects13/tests/Unit/PackageIsolationTest** — un dominio no depende del otro ni del bridge.

## Guardas automáticas
* **ResourceUniformityTest** — vigila la coherencia entre recursos: mismo orden de reglas, mismos `reuse_deleted_name` en `XRequest`, mismas reglas de búsqueda en `XListRequest`, mismas habilidades de policy con su call-site HTTP, factories con los mismos estados, mismo estado `archived` en todo modelo con flag `active`, mismo page size y desempate en las list queries, y el mismo sobre de acciones y `editPayload` allowlisted en los transformers.
* **ModelSchemaParityTest** — asegura la paridad `#[Fillable]` ↔ esquema de base de datos: una columna nueva sin `#[Fillable]` falla hasta que se declara (o se marca como columna de servidor).
* **PackageIsolationTest** — garantiza que un dominio no depende del otro ni del bridge: las fronteras de paquetes se respetan y los tests fallingan si hay fugas de dependencia.
* **PHPStan baseline consistency check** — revisa que las baselines por paquete sean consistentes y que los errores nuevos sean genuinos antes de añadirlos (no workarounds de versión). `basics13` tiene 11 entradas preexistentes; `entities13` y `projects13` generan baselines vacías; `entity-projects13` requiere atención especial (actualmente 4 errores no resueltos en `PackageSeederTest.php`).

El resto del automatismo:
* `composer ci:check` (en `projects/demo13`) = `config:clear` + pint --test + PHPStan (nivel 9) + suite de la app con cobertura mínima del 80% y Clover en `storage/logs/coverage.xml`. Cubre **solo la app**: ni este script ni el workflow de CI ejecutan un solo test de paquete. Cada paquete se cubre con su propio `composer ci:check`, desde su repo.

**Mantenimiento de baselines PHPStan**: las baselines (`phpstan-baseline.neon`) son específicas por paquete y no deben copiarse entre proyectos. `basics13` tiene 11 entradas preexistentes como trabajo alrededor de versiones de Fortify; `entities13` y `projects13` generan baselines vacías en primer análisis. `entity-projects13` requiere atención especial (actualmente 4 errores no resueltos en `PackageSeederTest.php`). **Política**: revisar cada error nuevo antes de añadirlo a la baseline — las entradas deben reflejar errores genuinos de análisis estático, no workarounds de versión. Al añadir un paquete nuevo, generar baseline fresh y revisar cada entrada antes de aceptarla.
* Tests en paralelo en local: `DX php artisan test --parallel --compact <ruta>` — los flags van **antes** de la ruta. Nunca en CI: los jobs comparten una BD. Requiere el grant `laravel\_test\_%` (ya aplicado, y en `packages/basics13/tooling/mysql/init/02-parallel-test-databases.sql` para volúmenes nuevos).
* Restaurar desde la papelera ya pide `XRestoreRequest` (policy + `resolve_name_conflict` validado). Si añades un endpoint con input, la validación va en su FormRequest, nunca en el controller.
* CI (master y PRs): cache de Composer y npm, un job Dusk, y se salta entera si el diff solo toca `.md`. Dependabot cubre composer, npm, docker y github-actions.

> Ojo: cada repo (`l13`, `demo13`, `basics13`, `entities13`, `projects13`, `entity-projects13`) tiene su propio remoto, y los paquetes están en `.gitignore` de `l13`. Un cambio de paquete se commitea en **su** repo, no en `l13`; y viceversa: este doc solo se versiona aquí.

## Reglas del juego
* Recordar una convención: edita a mano `.github/instructions/<zona>.instructions.md` (http, models, views, tests y packages; mismo `applyTo`, añade el bullet). Ahí vive todo lo que no quieras repetir.
* Tokens: una sesión = una tarea; ciérrala. Las reglas de ruta solo entran cuando se toca la ruta, y las skills solo cuando las activas.
* Arquitectura (repositorios, DTOs, servicios…): el prompt y review-rules.md lo bloquean sin causa concreta — si de verdad lo necesitas, dilo explícitamente. Lo mismo con las fronteras de paquetes: si un cambio no cabe en un dominio ni en el bridge, para y pregunta.
* Tras cada sesión: git diff + tests tú mismo + commit.